# Linux Process 健康檢查與效能分析 Runbook

> **適用環境**：Linux（CentOS 7+、Ubuntu 20.04+、Debian 11+ 等主流發行版）
> **適用場景**：C/C++ 長駐 daemon process 的運維診斷
> **前置需求**：coreutils（預裝）、procfs（預裝）、perf（需安裝 `linux-tools` 或 `perf`）
> **文件目標**：不依賴 AI，人可獨立完成從「觀察」到「判定」到「行動」的完整診斷

---

## Part 1：Process-Level 健檢

核心觀念：process-level 健檢觀察的是整個 process 的外部表現，不涉及程式內部邏輯。你關心的是四個維度——記憶體、File Descriptor、Thread 狀態、CPU 使用率。每個維度都有「正常」與「異常」的判定邏輯，以及異常時的下一步動作。

---

### 1.1 記憶體觀測

#### 目標

判斷 process 是否存在 memory leak。memory leak 的定義是：process 持有的記憶體隨時間**單調遞增且不回收**。

#### 關鍵指標

| 指標 | 來源 | 意義 |
|------|------|------|
| **VmRSS** | `/proc/<pid>/status` | Resident Set Size，process 實際佔用的物理記憶體。**這是判斷 leak 的主要指標。** |
| **VmSize** | `/proc/<pid>/status` | Virtual Memory Size，process 映射的虛擬位址空間總量。包含尚未實際分配的部分，數字通常遠大於 RSS。 |
| **VmPeak** | `/proc/<pid>/status` | VmSize 的歷史最高值。如果 VmSize == VmPeak，代表虛擬記憶體從未縮減過（不一定是問題，很多程式配了就不還）。 |
| **VmSwap** | `/proc/<pid>/status` | 被換出到 swap 的記憶體量。正常情況應為 0。如果非 0，代表系統記憶體壓力大，該 process 的部分記憶體被擠到磁碟，效能會劇烈下降。 |

#### 指令

**即時快照：**

```bash
grep -E "^(VmRSS|VmSize|VmPeak|VmSwap|Threads)" /proc/<pid>/status
```

**趨勢記錄（推薦至少跑 2 小時，每 60 秒一筆）：**

```bash
while true; do
  echo "$(date '+%Y-%m-%d %H:%M:%S') $(awk '/VmRSS/{print $2}' /proc/<pid>/status) kB"
  sleep 60
done | tee /tmp/mem_trend_<name>.log
```

#### 如何判讀

**正常的模式：**

- RSS 在某個範圍內波動（例如 100MB ± 15MB），隨業務量上下浮動，但長期不會超出這個區間。
- 啟動初期 RSS 快速上升（載入設定、建立連線池、預分配 buffer），之後趨於平穩。
- VmPeak > VmSize 代表曾經分配後有釋放，是健康的。

**異常的模式：**

- RSS **持續單調遞增**，完全沒有下降的段落。例如每小時增加 1MB，跑 7 天就多了 168MB。這是典型的 slow leak。
- VmSwap 非 0 且持續增長。代表整台機器的記憶體快用完了，你的 process（或其他 process）正在被 swap 壓迫。
- VmSize 在運行期間出現突然的大幅跳升（例如從 500MB 跳到 2GB），而你沒有預期中的大量分配行為。

**異常時下一步：**

1. 用 smaps 看記憶體長在哪個區段：
   ```bash
   awk '/^[0-9a-f]/{region=$0} /^Rss:/{if($2>1024) print $2,"kB",region}' /proc/<pid>/smaps | sort -rn | head -10
   ```
   如果最大的幾塊都在 `[heap]`，大概率是應用層 malloc 了沒 free。如果是某個 `.so` 映射特別大，可能是該 library 內部的問題。

2. 用 gdb 抓 call stack 看當下在做什麼：
   ```bash
   gdb -batch -ex "thread apply all bt" -p <pid> 2>/dev/null
   ```

3. 若判定確實有 leak 且需要精確定位，重啟時掛 valgrind（測試環境）：
   ```bash
   valgrind --leak-check=full --track-origins=yes ./your_program
   ```
   或者用 gperftools 的 heap profiler（效能衝擊小很多，可在接近 production 的環境用）：
   ```bash
   LD_PRELOAD=/usr/lib64/libtcmalloc.so HEAPPROFILE=/tmp/heap ./your_program
   ```

---

### 1.2 File Descriptor 觀測

#### 目標

判斷 process 是否存在 FD leak。FD leak 代表程式持續開啟 socket、file、pipe 卻沒有正確 close，最終會撞到系統上限導致 process 無法建立新連線或開新檔案。

#### 關鍵指標

| 指標 | 來源 | 意義 |
|------|------|------|
| **Open FD 數量** | `ls /proc/<pid>/fd \| wc -l` | 當前打開的 file descriptor 總數。 |
| **FD Limit** | `/proc/<pid>/limits` 中 `Max open files` | 該 process 能打開的 FD 上限（soft limit / hard limit）。 |
| **FD 類型分佈** | `ls -la /proc/<pid>/fd` | 每個 fd 指向什麼（socket、file、pipe、eventfd、anon_inode 等）。 |

#### 指令

**即時快照：**

```bash
# FD 數量與上限
echo "Open: $(ls /proc/<pid>/fd 2>/dev/null | wc -l)  Limit: $(awk '/open files/{print $4}' /proc/<pid>/limits)"

# FD 類型分佈（Top 10）
ls -la /proc/<pid>/fd 2>/dev/null | awk '{print $NF}' | sort | uniq -c | sort -rn | head -10

# 檢查是否有 deleted fd
ls -la /proc/<pid>/fd 2>/dev/null | grep deleted
```

**趨勢記錄：**

```bash
while true; do
  echo "$(date '+%Y-%m-%d %H:%M:%S') $(ls /proc/<pid>/fd 2>/dev/null | wc -l)"
  sleep 60
done | tee /tmp/fd_trend_<name>.log
```

#### 如何判讀

**正常的模式：**

- FD 數量穩定不變，或在一個小範圍內波動（例如連線進來 +1，處理完 -1）。
- FD 類型能對應到你的架構設計。例如：用了 epoll 就會看到 `anon_inode:[eventpoll]`，用了 eventfd 就會看到 `anon_inode:[eventfd]`，連了 MongoDB/Kafka 就會看到 `socket:` 類型的 fd。
- 少量的 `(deleted)` fd 是常見的，通常是 log rotation 造成的（見下方說明）。

**異常的模式：**

- FD 數量隨時間**持續單調遞增**。這代表有東西在反覆 open 而沒有 close——可能是 socket 連線建了沒斷、檔案開了沒關、pipe 建了沒收。
- FD 數量接近 limit（例如 limit 是 1024 而目前已經 900+）。一旦撞到上限，`socket()`、`open()` 等系統呼叫會返回 `EMFILE` 錯誤，程式可能直接異常。
- `(deleted)` 的數量持續增加。少量固定的 deleted（例如 log 檔）無害，但如果數量在漲，代表有程式反覆建立暫存檔然後從磁碟刪除但沒 close fd。

**關於 (deleted) FD 的深入說明：**

Linux 的檔案系統有一個特性：一個檔案被 `unlink`（刪除目錄入口）後，如果還有 process 持有它的 fd，該 inode 不會真正釋放，磁碟空間也不會回收。這就是為什麼你會看到 `(deleted)` 標記。

最常見的場景是 **log rotation**：外部工具（logrotate、cron job）把舊 log 檔刪了或 rename 了，但 process 還握著舊 fd 繼續寫。結果就是 process 以為自己在寫 log，但寫的是一個沒有目錄入口的幽靈 inode——新 log 沒人寫、舊 log 佔著空間刪不掉。

**解法：**
- 使用 log library 內建的 rotation（如 spdlog 的 `daily_file_sink`、`rotating_file_sink`），由程式自己管理檔案生命週期。
- 若用外部 logrotate，在 config 中使用 `copytruncate` 策略，它會複製後清空原檔而非刪除，fd 不會斷。
- 若程式支援 signal handler，可以用 `SIGHUP` 通知程式重新開啟 log 檔（Nginx、很多 daemon 都這麼做）。

**異常時下一步：**

1. 確認是什麼類型的 fd 在增長：
   ```bash
   # 看 socket 明細
   ls -la /proc/<pid>/fd 2>/dev/null | grep socket
   # 交叉比對 socket 資訊
   cat /proc/<pid>/net/tcp   # TCP 連線
   cat /proc/<pid>/net/tcp6  # TCP6 連線
   ```

2. 用 strace 即時觀察 open/close 行為：
   ```bash
   # 只追蹤 open/close/socket/connect 相關的系統呼叫，跑 10 秒
   timeout 10 strace -e trace=open,openat,close,socket,connect -p <pid> 2>&1 | tail -50
   ```

3. 用 lsof 看完整的 fd 列表（比 /proc 的 ls 更詳細）：
   ```bash
   lsof -p <pid>
   ```

---

### 1.3 Thread 狀態觀測

#### 目標

判斷 process 的 thread 是否正常運作——有沒有 thread 卡死、有沒有 thread 空轉吃 CPU、thread 數量是否在預期範圍內。

#### 關鍵指標

| 指標 | 來源 | 意義 |
|------|------|------|
| **Thread 數量** | `/proc/<pid>/status` 中 `Threads` | 當前 thread 總數。應與你程式設計的 thread 數一致。 |
| **Thread 狀態（STAT）** | `ps -p <pid> -T -o spid,stat,%cpu,wchan` | 每個 thread 的狀態碼、CPU 用量、以及在 kernel 中等待的位置。 |

#### STAT 欄位速查

| 代碼 | 意義 | 常見場景 |
|------|------|----------|
| **R** | Running / Runnable | 正在 CPU 上執行或在 run queue 等排程 |
| **S** | Interruptible Sleep | 在等 I/O、lock、timer 等，可被 signal 喚醒。**這是最正常的狀態。** |
| **D** | Uninterruptible Sleep | 在等不可中斷的 I/O（通常是 disk I/O）。短暫出現正常，**持續出現代表 I/O 有問題**（磁碟慢、NFS 卡住等）。 |
| **Z** | Zombie | process 已結束但 parent 還沒 wait() 回收。少量短暫出現無害，**持續存在代表 parent process 有 bug**。 |
| **T** | Stopped | 被 signal 暫停（SIGSTOP、SIGTSTP），或被 debugger attach。 |
| **l** | Multi-threaded | 修飾符，表示 process 有多個 thread。 |

#### wchan 欄位速查

`wchan` 表示 thread 當前在 kernel 的哪個函數裡等待。常見的正常值：

| wchan | 意義 |
|-------|------|
| `futex_wait_queue` 或 `futex_wait` | 在等 futex（mutex、condition_variable 底層都是它）。正常。 |
| `ep_poll` | 在等 epoll 事件。event-driven 架構的 thread 常駐在這裡。正常。 |
| `poll_schedule_timeout` | 在等 poll/select。正常。 |
| `nanosleep` | 在 sleep。正常，前提是你預期它在 sleep。 |
| `pipe_read` / `pipe_write` | 在等 pipe 的另一端。正常，前提是有對應的 producer/consumer。 |
| `-`（dash） | 不在 kernel 中等待，正在 CPU 上跑 user-space code。 |

#### 指令

```bash
# 所有 thread 的狀態、CPU 用量、等待位置
ps -p <pid> -T -o spid,stat,%cpu,wchan --no-headers

# 按 CPU 用量排序，看誰最忙
ps -p <pid> -T -o spid,stat,%cpu,wchan --no-headers | sort -k3 -rn

# 即時觀察 thread（互動式，按 H 切換到 thread 視圖）
top -H -p <pid>
```

#### 如何判讀

**正常的模式：**

- Thread 數量穩定，與你程式設計的 thread pool size 一致。
- 大部分 thread 處於 `S` 狀態（sleeping），少數 `R` 狀態（running）。
- running 的 thread 數量合理——例如一個行情 relay，1-2 個 thread running 是正常的。
- wchan 都在合理的位置（`ep_poll`、`futex_wait` 等）。

**異常的模式：**

- **某個 thread CPU 持續 100%，wchan 是 `-`**：可能卡在 busy loop（例如 `while(true)` 沒有 yield、spin lock 拿不到）。用 `perf top -t <spid>` 可以即時看該 thread 的 CPU 花在哪個 function。
- **thread 出現 `D` 狀態且持續不動**：不可中斷的 I/O 等待，通常是底層儲存有問題（磁碟慢、NFS timeout、卡在 kernel driver）。`D` 狀態的 thread 無法被 kill，只能等 I/O 完成或 reboot。
- **thread 數量不斷增加**：代表程式在不斷 create thread 而沒有 join/detach，是一種 resource leak。
- **出現 `Z` 狀態**：zombie process，parent 沒有 wait() 回收。如果你的架構有 fork child process，要確認 parent 有正確處理 SIGCHLD。

**異常時下一步：**

1. 對特定 thread 抓 stack trace：
   ```bash
   gdb -batch -ex "thread apply all bt" -p <pid> 2>/dev/null
   ```

2. 對特定 thread 做 strace：
   ```bash
   strace -p <spid> -tt 2>&1 | head -50
   ```

3. 如果懷疑 deadlock，看所有 thread 的 stack 是否都卡在 lock 相關的 call（`pthread_mutex_lock`、`__lll_lock_wait`）：
   ```bash
   gdb -batch -ex "thread apply all bt" -p <pid> 2>/dev/null | grep -E "(pthread_mutex|__lll_lock|futex)"
   ```

---

### 1.4 CPU 使用率觀測

#### 目標

判斷 process 的 CPU 使用率是否合理，以及是在做有效計算還是在空轉。

#### 指令

**即時快照：**

```bash
ps -p <pid> -o pid,%cpu,%mem,etime,cmd --no-headers
```

注意：`ps` 的 `%cpu` 是該 process **整個生命週期的平均 CPU 使用率**，不是當前瞬間值。如果你的 process 已經跑了 7 天，這個數字會被歷史資料稀釋。

**即時 CPU 使用率（用 top）：**

```bash
top -b -n1 -p <pid> | tail -1
```

這裡的 `%CPU` 是上一個採樣間隔的瞬間值，更能反映當前狀態。

**系統呼叫分佈（快速了解 process 在做什麼類型的工作）：**

```bash
timeout 5 strace -cp <pid> 2>&1
```

`-c` 是統計模式，5 秒後輸出一張系統呼叫頻率表。你可以從中看出這個 process 主要在做 I/O（read/write 多）、網路（recvfrom/sendto 多）、還是 sleep（nanosleep/futex 多）。

#### 如何判讀

**正常的模式：**

- CPU 使用率與業務量正相關。盤中行情密集時 CPU 高，盤後低。
- strace 統計中，系統呼叫類型與你的架構設計一致。例如行情 relay 服務應該大量 recvfrom/sendto，config 服務應該大部分時間在 futex_wait/epoll_wait。

**異常的模式：**

- **CPU 使用率異常高但業務量沒有變化**：可能是演算法退化（例如 O(n²) 的邏輯隨資料量膨脹）、busy loop、或 lock contention 嚴重導致 spin。
- **CPU 使用率為 0 但程式應該在工作**：可能是 main thread 卡死、或者連線斷了沒有重連。
- **strace 顯示大量 futex 呼叫**：可能有嚴重的 lock contention，多個 thread 在搶同一把鎖。

**異常時下一步：** 進入 Part 2 的 perf 分析，定位 CPU 時間花在哪些 function 上。

---

### 1.5 一鍵健檢腳本

將以上四個維度整合成一個可直接執行的腳本：

```bash
#!/bin/bash
# healthcheck.sh - Linux Process 健檢腳本
# 用法: healthcheck.sh <pid>
# 建議放置位置: /usr/local/bin/

set -euo pipefail

P=${1:?"用法: $0 <pid>"}

# 檢查 process 是否存在
if [ ! -d "/proc/$P" ]; then
  echo "Error: Process $P 不存在"
  exit 1
fi

PNAME=$(cat /proc/$P/comm 2>/dev/null || echo "unknown")

echo "=========================================="
echo " Process 健檢報告: $PNAME (PID: $P)"
echo " 檢查時間: $(date '+%Y-%m-%d %H:%M:%S')"
echo "=========================================="

echo ""
echo "--- 基本資訊 ---"
ps -p "$P" -o pid,ppid,stat,%cpu,%mem,rss,vsz,etime,cmd --no-headers 2>/dev/null

echo ""
echo "--- 記憶體 ---"
grep -E "^(VmPeak|VmSize|VmRSS|VmSwap|VmData|Threads)" /proc/"$P"/status

echo ""
echo "--- File Descriptors ---"
FD_COUNT=$(ls /proc/"$P"/fd 2>/dev/null | wc -l)
FD_LIMIT=$(awk '/open files/{print $4}' /proc/"$P"/limits 2>/dev/null)
FD_USAGE_PCT=$(awk "BEGIN{printf \"%.1f\", ($FD_COUNT/$FD_LIMIT)*100}")
echo "Open: $FD_COUNT  Limit: $FD_LIMIT  Usage: ${FD_USAGE_PCT}%"

DELETED_COUNT=$(ls -la /proc/"$P"/fd 2>/dev/null | grep -c deleted || true)
if [ "$DELETED_COUNT" -gt 0 ]; then
  echo "⚠ Deleted FDs: $DELETED_COUNT"
  ls -la /proc/"$P"/fd 2>/dev/null | grep deleted | awk '{print "  " $NF}'
fi

echo ""
echo "--- FD 類型分佈 (Top 5) ---"
ls -la /proc/"$P"/fd 2>/dev/null | awk 'NR>3{print $NF}' | \
  sed 's|/proc/.*||; s|\(.*\)\[.*\]|\1|' | sort | uniq -c | sort -rn | head -5

echo ""
echo "--- Thread 狀態 ---"
THREAD_SUMMARY=$(ps -p "$P" -T -o spid,stat,%cpu --no-headers 2>/dev/null | \
  awk '{s[$2]++; c+=$3} END{for(k in s) printf "%s:%d ", k, s[k]; printf "\nTotal CPU: %.1f%%\n", c}')
echo "$THREAD_SUMMARY"

# 簡易異常檢測
echo ""
echo "--- 快速診斷 ---"

RSS_KB=$(awk '/VmRSS/{print $2}' /proc/"$P"/status 2>/dev/null)
SWAP_KB=$(awk '/VmSwap/{print $2}' /proc/"$P"/status 2>/dev/null)

if [ "${SWAP_KB:-0}" -gt 0 ]; then
  echo "⚠ 記憶體: VmSwap 非零 (${SWAP_KB} kB)，系統記憶體壓力可能較大"
else
  echo "✓ 記憶體: VmSwap 為 0，無 swap 壓力"
fi

if [ "$FD_COUNT" -gt $((FD_LIMIT * 80 / 100)) ]; then
  echo "⚠ FD: 使用率超過 80%，接近上限"
else
  echo "✓ FD: 使用率 ${FD_USAGE_PCT}%，正常"
fi

if [ "$DELETED_COUNT" -gt 0 ]; then
  echo "⚠ FD: 有 $DELETED_COUNT 個 deleted fd，可能是 log rotation 問題"
fi

D_COUNT=$(ps -p "$P" -T -o stat --no-headers 2>/dev/null | grep -c "D" || true)
if [ "$D_COUNT" -gt 0 ]; then
  echo "⚠ Thread: $D_COUNT 個 thread 處於 D (uninterruptible sleep) 狀態"
else
  echo "✓ Thread: 無 D 狀態 thread"
fi

Z_COUNT=$(ps -p "$P" -T -o stat --no-headers 2>/dev/null | grep -c "Z" || true)
if [ "$Z_COUNT" -gt 0 ]; then
  echo "⚠ Thread: $Z_COUNT 個 zombie thread"
fi

echo ""
echo "=========================================="
```

**安裝與使用：**

```bash
sudo cp healthcheck.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/healthcheck.sh

# 使用
healthcheck.sh $(pidof your_program)
healthcheck.sh 12345
```

---

## Part 2：perf 效能分析

核心觀念：Part 1 告訴你 process 層級的「外部表現」，Part 2 用 perf 深入到 process 內部，回答「CPU 時間花在哪些 function 上」和「硬體資源（cache、branch predictor）的使用效率如何」。

perf 有兩個主要使用模式：`perf stat` 回答「整體效率如何」，`perf record` + `perf report` 回答「時間花在哪裡」。建議先跑 stat 建立全局觀，再用 record 鑽到細節。

---

### 2.1 前置準備

#### 安裝

```bash
# CentOS / RHEL
sudo yum install perf

# Ubuntu / Debian
sudo apt install linux-tools-$(uname -r) linux-tools-common
```

#### 確認版本與權限

```bash
perf --version

# 如果不是 root，需要調整 perf_event_paranoid
# 查看當前值
cat /proc/sys/kernel/perf_event_paranoid
```

`perf_event_paranoid` 控制非 root 使用者可以取得哪些 event data：

| 值 | 意義 |
|----|------|
| `-1` | 權限全開，不受任何限制。**測試環境建議設這個。** |
| `0` | 不允許 raw tracepoint access，但可使用 `perf stat`、`perf record` 取得 CPU events data。 |
| `1` | 不允許 CPU events data，但可使用 `perf stat`、`perf record` 取得 kernel profiling data。 |
| `2` | 不允許任何量測。但 `perf report`、`perf timechart` 等分析既有紀錄的指令仍可使用。**大部分發行版預設值。** |
| `3` | 完全禁止所有 perf event access。**Ubuntu 部分較新版本的預設值**（kernel 4.6+ 才有此值）。 |

```bash
# 測試環境建議：
sudo sysctl kernel.perf_event_paranoid=-1
```

#### 解禁 kernel symbol（kptr_restrict）

如果你需要在 perf report 中看到 kernel 函數名稱（`[k]` 標記的符號），需要先取消 kernel pointer 的禁用，否則 kernel 符號會顯示為全零位址：

```bash
# 查看當前值
cat /proc/sys/kernel/kptr_restrict
# 0: 允許查看 kernel pointer（需要設這個）
# 1: 隱藏 kernel pointer（大部分發行版預設）
# 2: 即使 root 也隱藏

# 測試環境建議：
sudo sh -c "echo 0 > /proc/sys/kernel/kptr_restrict"
```

#### 用 perf list 確認可用事件

不同 CPU 支援的 hardware event 不同、不同 kernel 版本支援的 software / tracepoint event 也不同。在正式使用前，先跑 `perf list` 確認當前環境支援哪些事件：

```bash
# 列出所有可用 event（tracepoint 需要 root 權限）
perf list

# 只看 hardware event
perf list hw

# 只看 software event
perf list sw

# 只看 tracepoint（需 root）
sudo perf list tracepoint
```

`perf list` 的輸出就是你在 `-e` 參數中可以使用的 event 名稱。如果你指定了一個當前環境不支援的 event，perf 會報錯，此時回來查 `perf list` 即可。

#### 確認 debug symbol

perf 的輸出品質取決於 symbol 資訊。沒有 symbol 的話，你只會看到記憶體位址而非 function 名稱。

```bash
# 檢查你的程式是否有 debug info
file ./your_program
# 期望看到 "not stripped" 或 "with debug_info"

# 如果是 stripped 的，重新編譯時加上：
# -g        產生 debug info
# -fno-omit-frame-pointer  保留 frame pointer，讓 -g call graph 更完整
```

對於 C++ 程式，建議在 CMake 中至少對 RelWithDebInfo 設定這兩個 flag：

```cmake
set(CMAKE_CXX_FLAGS_RELWITHDEBINFO "-O2 -g -fno-omit-frame-pointer")
```

---

### 2.2 perf stat — 硬體層級全局觀

#### 目標

回答：「這個 process 的 CPU 運算效率如何？是在做有效計算，還是在等記憶體、等 branch prediction？」

#### 指令

```bash
# 基礎版：採樣 10 秒
perf stat -p <pid> -- sleep 10

# 詳細版：包含 cache、branch、context switch
perf stat -e cycles,instructions,cache-references,cache-misses,branches,branch-misses,context-switches,cpu-migrations,page-faults -p <pid> -- sleep 10

# 針對特定 thread
perf stat -t <tid> -- sleep 10

# 重複執行多次取平均與變異（適合 benchmark 場景，對已啟動的 daemon 不適用）
perf stat --repeat 5 -e cache-misses,cache-references,instructions,cycles ./your_program
```

`--repeat <n>` 會重複執行 n 次目標程式，輸出每個 event 的平均值和 `+-` 百分比表示變異程度。這在做效能比較時很有用——如果某個 event 的 `+-` 超過 5%，代表數據不穩定，需要更多次取樣或排除干擾因素。

#### Event 修飾符（`:u` / `:k`）

在 event 名稱後加修飾符，可以限定只統計特定層級的事件：

| 修飾符 | 意義 | 使用場景 |
|--------|------|----------|
| `:u` | 只統計 user-space 發生的事件 | 只關心你自己程式的行為，排除 kernel 干擾 |
| `:k` | 只統計 kernel-space 發生的事件 | 懷疑瓶頸在系統呼叫或 driver |
| 不加 | 統計 user + kernel 兩者 | 一般情況 |

範例：

```bash
# 只看 user-space 的 branch miss
perf stat -e branch-misses:u,branch-instructions:u -p <pid> -- sleep 10

# 只看 kernel-space 的 cache miss
perf stat -e cache-misses:k,cache-references:k -p <pid> -- sleep 10
```

這在你的行情系統場景特別有用：如果 `cache-misses`（不加修飾符）很高，加上 `:u` 和 `:k` 分別跑一次，就能知道是你的程式的資料結構 cache 不友善，還是 kernel 在搬資料時造成的 cache miss。

#### 輸出範例與判讀

```
 Performance counter stats for process id '31469':

      3,842,156,789      cycles
      5,126,432,108      instructions              #    1.33  insn per cycle
        128,456,321      cache-references
         12,845,632      cache-misses              #   10.00% of all cache refs
        987,654,321      branches
         14,814,815      branch-misses             #    1.50% of all branches
             12,456      context-switches
                 23      cpu-migrations
                456      page-faults

      10.001234567 seconds time elapsed
```

#### 關鍵指標判讀

**IPC (Instructions Per Cycle)**

這是最重要的單一數字。它告訴你 CPU 每個時鐘週期完成了多少條指令。

| IPC 範圍 | 意義 | 可能原因 |
|----------|------|----------|
| > 2.0 | 非常高效 | 計算密集且 cache 命中率高，CPU pipeline 充分利用 |
| 1.0 - 2.0 | 正常 | 大部分應用落在這個區間 |
| 0.5 - 1.0 | 偏低 | 可能有較多 cache miss 或 branch miss，CPU 時常在等資料 |
| < 0.5 | 很低 | 嚴重的 memory stall，CPU 大部分時間在等 L3 cache 或主記憶體 |

對於你的行情系統場景：quote_relay 如果 IPC 在 1.0 以上，代表它在高效處理數據；如果 IPC 低於 0.5，可能是行情資料結構的存取模式對 cache 不友善（例如大量 pointer chasing、random access）。

**Cache Miss Rate**

```
cache-misses / cache-references × 100%
```

| Cache Miss Rate | 意義 |
|----------------|------|
| < 5% | 優秀，資料存取模式對 cache 友善 |
| 5% - 15% | 正常 |
| 15% - 30% | 偏高，可能有優化空間（資料結構調整、memory layout 優化） |
| > 30% | 嚴重，資料存取模式可能有根本性問題 |

注意：perf stat 的 `cache-references` 和 `cache-misses` 預設指的是 **Last Level Cache (LLC)**，通常是 L3。L1/L2 的 miss 會更頻繁但影響較小，因為 L2 miss 還可以被 L3 接住。如果 LLC miss rate 高，就代表要去主記憶體拿資料了，latency 會是 cache hit 的幾十到上百倍。

**Branch Miss Rate**

```
branch-misses / branches × 100%
```

| Branch Miss Rate | 意義 |
|-----------------|------|
| < 1% | 優秀 |
| 1% - 3% | 正常 |
| 3% - 10% | 偏高，可能有太多不規則的 if/else 或 switch/case |
| > 10% | 嚴重，考慮用 branchless 寫法或重新設計邏輯 |

對 HFT 場景，branch miss 的代價特別大——每次 miss 都要 flush pipeline，浪費十幾個 cycle。

**Context Switches**

| 數量級（每秒） | 意義 |
|---------------|------|
| < 1000 | 正常 |
| 1000 - 10000 | 偏高，可能有過多的 lock contention 或 I/O wait |
| > 10000 | 異常，嚴重的 contention 或設計問題 |

Context switch 分「自願」和「非自願」兩種。自願的（voluntary）是 thread 主動放棄 CPU（sleep、wait lock），非自願的（involuntary）是被 scheduler 搶走的（time slice 用完）。用 `perf stat -e context-switches,cpu-migrations` 看的是總數。如果要區分，看 `/proc/<pid>/status` 中的 `voluntary_ctxt_switches` 和 `nonvoluntary_ctxt_switches`。

---

### 2.3 perf record + perf report — CPU 熱點分析

#### 目標

回答：「CPU 時間花在哪些 function 上？call chain 是什麼？」

#### 指令

```bash
# 基礎 record：採樣 10 秒，頻率 999Hz，帶 call graph
perf record -g -F 999 -p <pid> -- sleep 10

# 只記錄 user-space 的 cycles（排除 kernel 干擾）
perf record -g -F 999 -e cycles:u -p <pid> -- sleep 10

# 結果會存成 perf.data（當前目錄）

# 查看報告（互動式 TUI）
perf report

# 純文字輸出（適合存檔或傳給別人看）
perf report --stdio > perf_report.txt

# 只看 top 20 hotspot
perf report --stdio --no-children | head -60
```

**關於取樣頻率的調整：** 如果你發現某些函式在 `perf report` 中沒有出現（可能是取樣頻率太低沒抓到），可以提高 `-F` 的值。系統有一個取樣頻率上限：

```bash
# 查看當前最大允許的取樣頻率
cat /proc/sys/kernel/perf_event_max_sample_rate

# 如果需要更高的頻率（測試環境）
sudo sysctl kernel.perf_event_max_sample_rate=10000
```

一般 999 已經足夠，除非你的 hotspot 非常分散或函式呼叫非常短暫。

#### 參數解釋

| 參數 | 意義 | 選擇建議 |
|------|------|----------|
| `-g` | 記錄 call graph（call chain） | 務必加，否則只看到 function 名稱但不知道誰呼叫它 |
| `-F 999` | 採樣頻率 999Hz（每秒 999 次） | 999 而非 1000 是為了避免與系統 timer 同步產生偏差。一般用 99-999 都可以，頻率越高越精確但 overhead 越大 |
| `--call-graph dwarf` | 使用 DWARF info 展開 call graph | 比預設的 frame pointer 方式更準確，但 overhead 更大。如果 `-g` 的結果 call chain 不完整（出現很多 `[unknown]`），改用這個 |
| `-- sleep 10` | 採樣持續 10 秒 | 根據需要調整。行情系統建議在盤中採樣才有意義 |

#### perf report 的 TUI 操作

進入 `perf report` 後：

| 按鍵 | 功能 |
|------|------|
| `Enter` | 展開/收合 function 的 call graph |
| `+` | 展開 call chain |
| `E` | 展開所有 call chain |
| `q` | 離開 |
| `/` | 搜尋 function 名稱 |
| `P` | 切換百分比顯示模式 |

#### 如何判讀 perf report 的輸出

一份典型的 `perf report --stdio` 輸出長這樣：

```
# Overhead  Command          Shared Object        Symbol
# ........  ...............  ...................  .............................
    18.32%  quote_relay      quote_relay.bin      [.] MarketData::decode
    12.45%  quote_relay      quote_relay.bin      [.] RingBuffer::push
     9.87%  quote_relay      libpthread.so        [.] pthread_mutex_lock
     8.21%  quote_relay      quote_relay.bin      [.] QuoteRouter::dispatch
     6.54%  quote_relay      libc.so              [.] __memcpy_avx_unaligned
     5.12%  quote_relay      [kernel.kallsyms]    [k] copy_user_enhanced_fast_string
     ...
```

**逐欄解釋：**

- **Overhead**：該 function 佔總 CPU 採樣的百分比。注意這裡有 `children` 和 `self` 兩種模式：
  - `self`：只計算 function 本身的 CPU 時間，不包含它呼叫的其他 function。
  - `children`（perf report 預設）：包含該 function 及其所有子呼叫的 CPU 時間。
  - 用 `--no-children` 參數可以切換到 self 模式，通常更容易找到真正的 hotspot。
- **Shared Object**：程式碼所在的二進位檔（你的 .bin、系統 .so、或 kernel）。
- **Symbol 前的標記**：`[.]` 代表 user-space，`[k]` 代表 kernel-space。

**判讀策略：**

1. **先看 self overhead 最高的前 5 個 function**。這些是你的程式真正在花時間的地方。

2. **區分「預期的 hotspot」和「意外的 hotspot」**：
   - 預期的：行情 relay 的 decode/encode、資料分發、序列化。這些是核心邏輯，花時間合理。
   - 意外的：`pthread_mutex_lock` 排名很前面（lock contention）、`malloc`/`free` 佔比高（頻繁配置記憶體）、`memcpy` 佔比高（大量資料複製）。

3. **看 kernel vs user 的比例**：
   - 如果 kernel 函數（`[k]` 標記）佔比超過 30-40%，代表程式花了很多時間在系統呼叫上（I/O、context switch、memory mapping）。
   - 理想狀態是大部分時間花在 user-space 的業務邏輯上。

4. **展開 call graph 追溯 hotspot 的來源**：
   如果 `pthread_mutex_lock` 佔 10%，展開它的 call chain 可以看到是哪個 function 在拿鎖、是哪把鎖在競爭。

#### 常見的 hotspot 模式與優化方向

| 排名靠前的 function | 可能意義 | 優化方向 |
|---------------------|---------|----------|
| `pthread_mutex_lock` / `__lll_lock_wait` | Lock contention 嚴重 | 減少 critical section 範圍、改用 lock-free 結構、改用 read-write lock |
| `malloc` / `free` / `operator new` | 頻繁的記憶體分配 | 使用 memory pool、object pool、arena allocator、std::pmr |
| `memcpy` / `memmove` | 大量資料複製 | 改用 zero-copy 設計、pass by reference、move semantics |
| `__GI___libc_write` / `write` | I/O 操作多 | 減少 log 頻率、使用 async I/O、buffer write |
| `_raw_spin_lock` (kernel) | Kernel lock contention | 減少系統呼叫頻率、batch 處理 |
| `copy_user_enhanced_fast_string` (kernel) | User-kernel 資料搬移 | 減少 read/write 次數、增大 buffer size |

---

### 2.4 perf 進階：鎖定特定問題

#### 看特定 thread 的 CPU 熱點

```bash
# 先找到目標 thread 的 TID
ps -p <pid> -T -o spid,stat,%cpu --no-headers | sort -k3 -rn

# 對特定 thread 做 record
perf record -g -F 999 -t <tid> -- sleep 10
perf report
```

#### 看 cache miss 發生在哪些 function

```bash
# 記錄 LLC miss 事件
perf record -e cache-misses -g -p <pid> -- sleep 10
perf report
```

這會告訴你哪些 function 觸發了最多的 cache miss，對優化資料結構的 memory layout 非常有用。

#### 看 context switch 發生在哪裡

```bash
perf record -e context-switches -g -p <pid> -- sleep 10
perf report
```

可以看到 context switch 主要發生在哪些 call path 上，通常是 lock wait 或 I/O wait。

#### 即時觀察（不存檔）

```bash
# 類似 top 的即時顯示，看當前 CPU 在執行什麼 function
perf top -p <pid>

# 只看特定 thread
perf top -t <tid>
```

`perf top` 適合快速 debug，不需要 record → report 的兩步流程。

---

### 2.5 perf 輸出的保存與分享

```bash
# 產出可以直接閱讀的純文字報告
perf report --stdio > /tmp/perf_report_$(date +%Y%m%d_%H%M%S).txt

# 產出帶 call graph 的完整報告
perf report --stdio -g > /tmp/perf_callgraph_$(date +%Y%m%d_%H%M%S).txt

# 產出更易讀的折疊格式（可搭配 FlameGraph 工具產生火焰圖）
perf script > /tmp/perf_script.out
```

#### 火焰圖（Flame Graph）

火焰圖是視覺化 perf 資料最直觀的方式。如果環境允許安裝 git：

```bash
git clone https://github.com/brendangregg/FlameGraph.git

perf record -g -F 999 -p <pid> -- sleep 10
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flamegraph.svg
```

產出的 SVG 可以用瀏覽器開啟，滑鼠移上去可以看每個 function 的 CPU 佔比。

---

## 附錄 A：快速指令速查

所有指令假設 `$P` 為目標 PID。

| 目的 | 指令 |
|------|------|
| 一鍵健檢 | `healthcheck.sh $P` |
| 記憶體快照 | `grep -E "^(VmRSS\|VmSize\|VmPeak\|VmSwap)" /proc/$P/status` |
| 記憶體趨勢 | `while true; do echo "$(date +%T) $(awk '/VmRSS/{print $2}' /proc/$P/status)"; sleep 60; done \| tee /tmp/mem.log` |
| FD 數量 | `ls /proc/$P/fd \| wc -l` |
| FD 明細 | `ls -la /proc/$P/fd` |
| Deleted FD | `ls -la /proc/$P/fd \| grep deleted` |
| Thread 狀態 | `ps -p $P -T -o spid,stat,%cpu,wchan --no-headers` |
| Thread 互動式 | `top -H -p $P` |
| Call stack | `gdb -batch -ex "thread apply all bt" -p $P` |
| 系統呼叫統計 | `timeout 5 strace -cp $P` |
| 可用 perf event | `perf list` |
| 解禁 kernel symbol | `sudo sh -c "echo 0 > /proc/sys/kernel/kptr_restrict"` |
| 硬體 counter | `perf stat -p $P -- sleep 10` |
| 硬體 counter（多次平均） | `perf stat --repeat 5 -e cache-misses,cache-references,instructions,cycles ./prog` |
| CPU 熱點 record | `perf record -g -F 999 -p $P -- sleep 10` |
| CPU 熱點（僅 user-space） | `perf record -g -F 999 -e cycles:u -p $P -- sleep 10` |
| CPU 熱點 report | `perf report --stdio --no-children` |
| Cache miss 熱點 | `perf record -e cache-misses -g -p $P -- sleep 10` |
| 即時 CPU 熱點 | `perf top -p $P` |
| 查看最大取樣頻率 | `cat /proc/sys/kernel/perf_event_max_sample_rate` |
| 火焰圖 | `perf script \| stackcollapse-perf.pl \| flamegraph.pl > flame.svg` |

---

## 附錄 B：FHS 目錄速查（腳本放哪裡）

| 目錄 | 用途 | 誰管的 |
|------|------|--------|
| `/usr/bin/` | 系統套件的指令 | 套件管理器（yum/apt），別手動放 |
| `/usr/local/bin/` | 本機管理員自訂的指令和腳本 | 你自己，**healthcheck.sh 放這裡** |
| `/usr/local/sbin/` | 同上，但慣例是只給 root 用的管理工具 | 你自己 |
| `/opt/` | 第三方獨立軟體（完整套裝） | 軟體供應商 |
| `~/bin/` | 個人用的腳本，不影響其他 user | 你自己 |

---

## 附錄 C：CentOS 7.9 特殊注意事項

以下是 CentOS 7.9 與較新發行版（Ubuntu 22.04+）的主要差異，升級後可以忽略：

- **Kernel 3.10**：eBPF 基本不可用（需要 4.15+）。升級到 Ubuntu 24.04（kernel 6.8）後可以使用 `bpftrace`、BCC tools 等強大的動態追蹤工具。
- **perf 版本較舊**：部分新的 hardware event 可能不支援，但本文使用的指令都相容。
- **glibc 版本 2.17**：部分新工具（如較新版的 valgrind）可能需要更高版本的 glibc。
- **systemd 版本 219**：部分較新的 systemd 功能（如 `MemoryMax`、`WatchdogSec` 的進階用法）可能不可用。
