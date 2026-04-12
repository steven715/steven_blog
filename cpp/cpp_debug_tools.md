# C/C++ 調試工具使用指南

> 適用環境：Ubuntu 22.04 / GCC 11 / x86-64  
> 目標讀者：有 C++ 基礎、需要在多執行緒業務系統中進行調試與效能分析的開發者

---

## 工具全覽

| 工具 | 類型 | 需重新編譯 | 可用於生產環境 | 速度損耗 |
|------|------|-----------|--------------|---------|
| AddressSanitizer | 記憶體錯誤偵測 | 是 | 否 | ~2x |
| ThreadSanitizer | Data race 偵測 | 是 | 否 | ~5-15x |
| Valgrind (Memcheck) | 記憶體錯誤偵測 | 否 | 否 | 10-50x |
| Valgrind (Helgrind) | Data race 偵測 | 否 | 否 | 10-50x |
| rr | 錄製重播調試 | 是 | 否 | 1.5-3x |
| strace | 系統呼叫追蹤 | 否 | 是 | 低 |
| perf | 效能分析 | 否 | 是 | 極低 |
| eBPF | 動態追蹤 | 否 | 是 | 極低 |
| gprof | 效能分析 (已過時) | 是 | 否 | 中等且失真 |

---

## 一、AddressSanitizer / ThreadSanitizer

### 定位

編譯期插樁工具，在每次記憶體存取前後插入檢查邏輯，出錯時立即中止並給出精確的 call stack。

- **ASan**：偵測 heap/stack 的非法存取、use-after-free、記憶體洩漏
- **TSan**：偵測多執行緒的 data race（兩個執行緒同時存取同一記憶體，且至少一個是寫入）

### 使用場景

- 懷疑有 use-after-free 或 buffer overflow，但 crash 難以重現
- 準備修改多執行緒共享物件的欄位之前，確認現有的 race 範圍
- 修改完 `atomic` 欄位後，驗證 data race 已消除

### 使用方式

```bash
# ASan：偵測記憶體錯誤
g++ -fsanitize=address -fno-omit-frame-pointer -g -O1 your_code.cpp -o your_program
./your_program

# TSan：偵測 data race（不可與 ASan 同時使用）
g++ -fsanitize=thread -fno-omit-frame-pointer -g -O1 your_code.cpp -o your_program
./your_program

# 控制輸出行為
ASAN_OPTIONS=halt_on_error=0:log_path=/tmp/asan.log ./your_program
TSAN_OPTIONS=halt_on_error=0:log_path=/tmp/tsan.log ./your_program
```

### 輸出判讀

```
==12345==ERROR: ThreadSanitizer: data race
  Write of size 8 at 0x... by thread T2:
    #0 ProfitCalculator::updatePrice  ProfitCalculator.cpp:87
  Previous read of size 8 at 0x... by thread T1:
    #0 RiskRuleHandler::calcMargin    RiskRuleHandler.cpp:134
```

關鍵欄位：`Write/Read of size N`（存取大小）、`by thread Tx`（哪個執行緒）、call stack（問題位置）。

### 書中對應章節

《高效 C/C++ 調試》第 10 章「內存調試工具」— Google Address Sanitizer 節，包含實際案例與工具箱配置說明。

### 延伸閱讀

- [AddressSanitizer — LLVM 官方文件](https://clang.llvm.org/docs/AddressSanitizer.html)
- [ThreadSanitizer — LLVM 官方文件](https://clang.llvm.org/docs/ThreadSanitizer.html)
- [GCC Instrumentation Options](https://gcc.gnu.org/onlinedocs/gcc/Instrumentation-Options.html)
- OS 知識補充：[虛擬記憶體與 shadow memory 機制](https://en.wikipedia.org/wiki/Shadow_memory)

---

## 二、Valgrind（Memcheck / Helgrind）

### 定位

虛擬機器式工具，將程式放在模擬 CPU 上執行，攔截每一個記憶體操作。**不需重新編譯**，適合手邊只有 binary 的場景。

- **Memcheck**：記憶體錯誤偵測，功能與 ASan 重疊，但能更完整追蹤記憶體洩漏路徑
- **Helgrind**：data race 偵測，功能與 TSan 重疊，速度更慢

### 使用場景

- 沒有原始碼或無法重新編譯，只有 binary
- 需要完整的記憶體洩漏 call stack（長期執行的 daemon 的洩漏追蹤）
- 需要確認第三方函式庫是否有洩漏

### 使用方式

```bash
# 安裝
sudo apt install valgrind

# Memcheck：完整記憶體洩漏報告
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         --log-file=/tmp/valgrind.log \
         ./your_program

# Helgrind：data race 偵測
valgrind --tool=helgrind \
         --log-file=/tmp/helgrind.log \
         ./your_program

# Callgrind：效能分析（搭配 kcachegrind 視覺化）
valgrind --tool=callgrind ./your_program
kcachegrind callgrind.out.*
```

### 輸出判讀

```
==12345== 48 bytes in 1 blocks are definitely lost in loss record 1 of 3
==12345==    at 0x4C2FB0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck.so)
==12345==    by 0x10916B: createPosition (PositionManager.cpp:42)
==12345==    by 0x1091F3: main (main.cpp:15)
```

`definitely lost`：確定洩漏。`possibly lost`：可能洩漏（通常與 shared_ptr 循環引用有關）。

### 書中對應章節

《高效 C/C++ 調試》第 14 章「內存泄漏」— 分析與調試節。

### 延伸閱讀

- [Valgrind 官方手冊](https://valgrind.org/docs/manual/manual.html)
- [Memcheck 手冊](https://valgrind.org/docs/manual/mc-manual.html)
- OS 知識補充：[malloc 內部結構 — ptmalloc 設計](https://sourceware.org/glibc/wiki/MallocInternals)

---

## 三、rr（Record and Replay）

### 定位

Mozilla 開發的錄製重播調試器，將程式的完整執行過程（包含所有執行緒的時序）錄製下來，之後可以用 GDB 介面反覆重播，支援**反向執行**（`reverse-continue`、`reverse-next`）。

解決的核心問題：多執行緒 bug 通常難以穩定重現，rr 把「能不能重現」變成「要倒帶多遠」。

### 使用場景

- 多執行緒 race condition 偶發 crash，正常調試無法穩定重現
- 想從 crash 點往回追，找到是哪一刻的操作導致記憶體損壞
- 風控系統在特定時序下出現計算異常，需要重播當時的執行過程

### 使用方式

```bash
# 安裝
sudo apt install rr
# 或從 GitHub 抓最新版
# https://github.com/rr-debugger/rr/releases

# 確認 CPU 支援（需要 Skylake 以上或 AMD Zen 以上）
rr cpufeatures

# 錄製
rr record ./your_program [args]

# 重播（進入 GDB 介面）
rr replay

# GDB 介面內的反向指令
(rr) continue              # 正向執行到下一個事件
(rr) reverse-continue      # 反向執行到上一個事件
(rr) reverse-next          # 反向單步
(rr) reverse-finish        # 反向執行到函數進入點

# 重播最近一次錄製
rr replay -d rust-gdb      # 搭配 rust-gdb 使用
```

### 注意事項

- 需要 `perf_event_paranoid` 設定為 1 以下：`echo 1 | sudo tee /proc/sys/kernel/perf_event_paranoid`
- 不支援 AVX-512 的某些指令集，部分 HFT 優化程式碼可能需要調整
- 錄製時 CPU 必須關閉 hardware performance counters 以外的機制

### 書中對應章節

《高效 C/C++ 調試》第 8 章「更多調試方法」— 逆向調試節（8.4）。

### 延伸閱讀

- [rr 官方網站](https://rr-project.org/)
- [rr GitHub](https://github.com/rr-debugger/rr)
- 硬體知識補充：[CPU performance counters 原理](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)（Intel SDM Volume 3B，Chapter 19）

---

## 四、strace

### 定位

追蹤 process 對 Linux kernel 發出的**系統呼叫**（syscall），以及接收到的訊號。不需重新編譯，可直接附加到正在執行的 process。

### 使用場景

- 程式在某個 I/O 操作卡住，懷疑是 `read()`、`write()`、`epoll_wait()` 沒有返回
- 確認程式實際開啟了哪些檔案、連接了哪些 socket
- 調試 Kafka consumer 為何沒有收到訊息（追蹤 socket 系統呼叫）
- 確認設定檔是否被正確讀取

### 使用方式

```bash
# 追蹤新啟動的程式
strace ./your_program

# 附加到正在執行的 process
strace -p <PID>

# 只追蹤特定類型的系統呼叫
strace -e trace=network ./your_program      # 網路相關
strace -e trace=file ./your_program         # 檔案相關
strace -e trace=read,write ./your_program   # 讀寫

# 統計各系統呼叫的時間與次數
strace -c ./your_program

# 追蹤子執行緒
strace -f ./your_program

# 輸出到檔案（多執行緒時按 PID 分開）
strace -f -o /tmp/strace.log ./your_program
```

### 輸出判讀

```
epoll_wait(5, [], 1, 100)    = 0 <0.100152>   # 等了 100ms 沒有事件
read(6, "...", 4096)         = -1 EAGAIN       # 非阻塞讀取，目前沒有資料
connect(7, {sa_family=AF_INET, ...}, 16) = -1 ECONNREFUSED  # 連線被拒
```

### 書中對應章節

《高效 C/C++ 調試》第 12 章「更多調試工具」— strace 節（12.1），包含實戰故事 7「僵屍進程」。

### 延伸閱讀

- [strace 官方文件](https://strace.io/)
- [Linux syscall table](https://syscalls.mebeim.net/?table=x86/64/x64/latest)
- OS 知識補充：[Linux 系統呼叫機制](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html)

---

## 五、perf

### 定位

Linux 核心內建的**效能分析工具**，透過硬體效能計數器（PMU）和 kernel tracepoint 進行取樣，開銷極小，可在生產環境使用。

### 使用場景

- 確認風控系統的 CPU 時間分布（哪個函數最耗時）
- 找出 cache miss 嚴重的程式碼路徑
- 分析 8 個 worker 執行緒的負載是否均衡
- 確認系統呼叫是否成為瓶頸

### 使用方式

```bash
# 安裝
sudo apt install linux-tools-common linux-tools-$(uname -r)

# 基本效能剖析（取樣 30 秒）
perf record -g ./your_program
perf report

# 附加到正在執行的 process
perf record -g -p <PID> sleep 30
perf report

# 即時顯示最耗 CPU 的函數（類似 top）
perf top -p <PID>

# 統計硬體事件（cache miss、branch miss 等）
perf stat -e cache-misses,cache-references,instructions,cycles ./your_program

# 產生火焰圖（需搭配 FlameGraph 工具）
perf record -F 99 -g -p <PID> sleep 30
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flame.svg
```

### 重要的效能指標

```
Instructions per cycle (IPC) > 2：CPU 使用效率高
Cache miss rate > 10%：記憶體存取模式值得優化
Branch miss rate > 5%：分支預測失敗，考慮消除分支
```

### 書中對應章節

《高效 C/C++ 調試》第 12 章「更多調試工具」— Perf 節（12.3）。

### 延伸閱讀

- [perf 官方 wiki](https://perf.wiki.kernel.org/)
- [Brendan Gregg — perf 使用教學](https://www.brendangregg.com/perf.html)
- [FlameGraph 工具](https://github.com/brendangregg/FlameGraph)
- 硬體知識補充：[CPU cache 層次結構與 cache line](https://en.wikipedia.org/wiki/CPU_cache)
- 硬體知識補充：[Intel PMU 效能計數器](https://perfmon-events.intel.com/)

---

## 六、eBPF

### 定位

Linux 核心的可程式化追蹤框架，允許在核心內安全地執行自訂程式，可以追蹤幾乎任何核心事件，**幾乎沒有效能開銷**，是目前生產環境動態追蹤的最強工具。

### 使用場景

- 生產環境中追蹤特定函數的執行時間，不重啟服務
- 分析 Kafka consumer 的網路延遲分布
- 追蹤鎖的競爭情況（哪個 mutex 等待時間最長）
- 監控記憶體分配頻率（追蹤 `malloc`/`free` 呼叫）

### 使用方式（透過 BCC 工具集，最易上手）

```bash
# 安裝 BCC
sudo apt install bpfcc-tools linux-headers-$(uname -r)

# 追蹤指定 PID 的所有系統呼叫延遲
sudo syscount -p <PID> -i 5

# 追蹤函數執行時間（以 malloc 為例）
sudo funclatency -p <PID> 'c:malloc'

# 追蹤 TCP 連線延遲（適合 Kafka 診斷）
sudo tcpconnect -p <PID>
sudo tcpretrans

# 追蹤函數呼叫次數（uprobe，需指定 binary 路徑）
sudo uflow -l c++ <PID>

# 使用 bpftrace 撰寫自訂追蹤腳本
# 追蹤某個函數的執行時間分布
sudo bpftrace -e '
  uprobe:/path/to/binary:_ZN15RiskRuleHandler10calcMarginEv { @start[tid] = nsecs; }
  uretprobe:/path/to/binary:_ZN15RiskRuleHandler10calcMarginEv
  / @start[tid] /
  { @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }
'
```

### 書中對應章節

《高效 C/C++ 調試》第 12 章「更多調試工具」— eBPF 節（12.4），包含完整的環境準備、程式撰寫、編譯與執行流程。

### 延伸閱讀

- [eBPF 官方網站](https://ebpf.io/)
- [BCC 工具集 GitHub](https://github.com/iovisor/bcc)
- [bpftrace GitHub](https://github.com/bpftrace/bpftrace)
- [Brendan Gregg — BPF Performance Tools（書籍）](https://www.brendangregg.com/bpf-performance-tools-book.html)
- OS 知識補充：[eBPF 核心架構](https://docs.kernel.org/bpf/index.html)

---

## 七、gprof（不建議用於多執行緒場景）

### 定位

GNU 的傳統取樣式效能分析器，透過 `-pg` 旗標在程式中插入計數器，程式執行完產生 `gmon.out`，再用 `gprof` 解析。

### 已知限制（在現代多執行緒系統下的根本問題）

- 多執行緒數據會混合，無法區分各執行緒的負載
- `-pg` 旗標會干擾編譯器優化，測出的效能數據與生產環境不符
- 看不到 kernel 層的時間消耗

### 使用方式（僅供參考）

```bash
g++ -pg -g your_code.cpp -o your_program
./your_program
gprof your_program gmon.out > analysis.txt
```

**建議**：直接用 `perf` 取代，沒有上述任何限制，且不需要重新編譯。

---

## 工具選擇決策樹

```
程式出現問題
├── 開發/測試環境
│   ├── Crash 或記憶體錯誤
│   │   ├── 有原始碼 → AddressSanitizer
│   │   └── 只有 binary → Valgrind Memcheck
│   ├── Data race / 多執行緒 bug
│   │   ├── 有原始碼 → ThreadSanitizer
│   │   ├── 只有 binary → Valgrind Helgrind
│   │   └── 難以重現 → rr (錄製後反向調試)
│   └── 效能問題
│       └── perf / Valgrind Callgrind
│
└── 生產環境（不能重啟、不能重新編譯）
    ├── 程式卡住 / I/O 異常 → strace
    ├── CPU 使用率高 → perf top / perf record
    ├── 需要追蹤特定函數 → eBPF (bpftrace / BCC)
    └── 需要網路延遲分析 → eBPF (tcpconnect / tcpretrans)
```

---

## 對應業務系統的典型工作流

以風控系統（多執行緒、長期執行的 daemon）為例：

**階段一：修改前確認問題範圍**
```bash
# 用 TSan 掃描現有 data race
g++ -fsanitize=thread -g -O1 ... && ./risk_engine
# 輸出每一個 race 的位置，作為修改清單
```

**階段二：修改後驗證**
```bash
# 確認 atomic 改動後 TSan 不再報錯
g++ -fsanitize=thread -g -O1 ... && ./risk_engine
# 同時用 ASan 確認沒有引入新的記憶體錯誤
g++ -fsanitize=address -g -O1 ... && ./risk_engine
```

**階段三：上生產前效能基線**
```bash
# 建立效能基線，改動後對比
perf stat -e cache-misses,instructions,cycles ./risk_engine
```

**階段四：生產環境監控**
```bash
# 服務上線後，定時確認沒有異常的系統呼叫行為
strace -c -p <PID> sleep 60

# 如果出現效能退化，用 perf 找熱點
perf record -g -p <PID> sleep 30 && perf report
```

---

## 參考資源整理

### 書籍
- [高效 C/C++ 調試（本指南主要參考）](https://www.tenlong.com.tw/products/9787302649717)
- [C/C++ 代碼調試的藝術](https://www.tenlong.com.tw/products/9787115554635)
- [BPF Performance Tools — Brendan Gregg](https://www.brendangregg.com/bpf-performance-tools-book.html)

### OS 基礎知識
- [Linux 核心記憶體管理](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)
- [Linux Inside（系統呼叫、記憶體、執行緒）](https://0xax.gitbooks.io/linux-insides/content/)
- [Linux Kernel Development — Robert Love](https://www.tenlong.com.tw/products/9787115475145)

### 硬體基礎知識
- [CPU Cache 與 False Sharing](https://www.aristeia.com/TalkNotes/codedive-CPUCacheRunSummary.pdf)
- [Memory Barriers — Linux Kernel 文件](https://www.kernel.org/doc/html/latest/core-api/wrappers/memory-barriers.html)
- [Intel 64 and IA-32 Architectures Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

### 線上工具
- [Compiler Explorer（即時看 assembly 輸出）](https://godbolt.org/)
- [Quick Bench（微基準測試）](https://quick-bench.com/)
