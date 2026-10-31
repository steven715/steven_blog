# TCP 與傳輸層核心筆記

> 從「推送訊息為何會漏」出發，整理傳輸層的因果模型：TCP 保證了什麼、沒保證什麼、可靠性該建在哪一層。

---

## 1. 分層定位

| 層 | 協議 | 負責 |
|---|---|---|
| L3 網路層 | IP | 定址、路由：把封包從 A 主機送到 B 主機 |
| L4 傳輸層 | TCP / UDP | 端對端傳輸語意：port、可靠性、順序、流量與壅塞控制 |
| L7 應用層 | HTTP、WebSocket、gRPC、FIX… | 訊息格式、framing、業務語意 |

- L3、L4 傳統上都在**作業系統核心**實作；應用開發者碰到的介面是 socket API，它開在 L4 上。
- QUIC 的關鍵改變之一：把傳輸層邏輯搬到 **user space**，跑在 UDP 上。
- IP 層對應用的槓桿較小，但仍有幾個開關：MTU／分片、ECN（標記位在 IP header）、DSCP/QoS、多播（multicast）。

---

## 2. TCP Header（RFC 9293）

固定 20 bytes，加上 Options 最多 40 bytes，共最多 60 bytes。每列 32 bits：

```
 0                   1                   2                   3
+-------------------------------+-------------------------------+
|        Source Port (16)       |     Destination Port (16)     |
+-------------------------------+-------------------------------+
|                    Sequence Number (32)                       |
+---------------------------------------------------------------+
|                 Acknowledgment Number (32)                    |
+-------+-------+---------------+-------------------------------+
|Offset | Rsvd  |  Flags (8)    |          Window (16)          |
|  (4)  |  (4)  |               |                               |
+-------+-------+---------------+-------------------------------+
|         Checksum (16)         |      Urgent Pointer (16)      |
+-------------------------------+-------------------------------+
|              Options + Padding（0–40 bytes，可變）             |
+---------------------------------------------------------------+
```

| 欄位 | 作用 |
|---|---|
| Source / Destination Port | 識別兩端的應用。**IP 位址不在這裡**，在 L3 的 IP header |
| Sequence Number | 這段資料第一個 byte 在串流中的編號（**byte 編號，不是封包計數**） |
| Acknowledgment Number | 「我下一個期待的 byte 編號」＝ 此前全部收齊（累積確認） |
| Data Offset | header 長度（單位 32-bit 字）。因 Options 可變長，靠它找出 payload 起點。值 5＝20 bytes，最大 15＝60 bytes。**與順序無關** |
| Reserved | 保留位，必須為 0，留作協議擴充（ECN 的 CWR/ECE 就是從保留空間劃出來的） |
| Flags | CWR、ECE、URG、ACK、PSH、RST、SYN、FIN |
| Window | 接收方廣告的 **rwnd**：「我接收緩衝還能收多少 bytes」。**不是訊息大小** |
| Checksum | 校驗 header + payload（計算時帶入含 IP 位址的 pseudo-header） |
| Urgent Pointer | 搭配 URG，實務上幾乎不用 |
| Options | MSS、Window Scale、SACK、Timestamp 等，多在握手時協商 |

補充：
- TCP header **沒有 payload 長度欄位**，長度由 `IP total length − IP header − TCP header` 推出。UDP 反而有 Length 欄位。
- Window 只有 16 bits（最大 65535 bytes），高 BDP 線路不夠用 → **Window Scale** 選項：實際視窗 ＝ 廣告值 × 2^scale，握手時協商一次。

---

## 3. TCP vs UDP vs QUIC

| | TCP | UDP | QUIC |
|---|---|---|---|
| 標頭 | 20–60 bytes | 8 bytes（port×2、Length、Checksum） | 包在 UDP payload 內，大部分加密 |
| 連線 | 三次握手 | 無 | 傳輸 + TLS 1.3 握手合併，1-RTT／0-RTT |
| 可靠／順序 | 內建 | 無 | 內建，per-stream |
| 流量控制 rwnd | 有 | 無 | 有（user space 實作） |
| 壅塞控制 cwnd | 有（核心） | 無 | 有（user space 實作） |
| 隊頭阻塞 | 單一 byte 串流，掉一包全卡 | 不適用 | 多條獨立 stream，掉包只阻塞該 stream |
| 連線識別 | 四元組（IP:port × 2） | — | Connection ID，可做連線遷移 |
| 實作位置 | 核心 | 核心 | user space，迭代快、CPU 成本較高 |

TCP 那堆欄位存在的唯一理由：在不可靠的 IP 上，偽造出一條**可靠、有序的 byte 串流**。UDP 則什麼都不做，要什麼自己在上層加。

### QUIC 的取捨補充
- **連線遷移 ≠ 拿到 CID 就能接管**。身分憑證是 TLS 金鑰；位址改變時要做 path validation（`PATH_CHALLENGE` / `PATH_RESPONSE`）。CID 頻繁輪換是為了防追蹤（隱私），不是防接管。
- **端點本身被攻陷**時，攻擊者就是合法端點，任何傳輸協議都無法分辨。對應的防線是認證層的**撤銷**（session 失效、重新認證、金鑰輪換），不在傳輸層。
- **0-RTT 的風險是 replay**，且無前向保密。協議明定 0-RTT 只能承載冪等、可安全重放的操作；哪些操作可重放只有應用知道語意，所以最終責任在應用。

### 僵化（ossification）
TCP 欄位明文、在核心實作，沿路中間設備會檢查甚至改寫 header，導致 TCP 難以演進。QUIC 加密整包，中間設備只看得到「一包 UDP」，因此無從干涉。

---

## 4. 序號與 ACK

- 序號就是一個整數，數的是 **byte**。十進制、十六進制只是寫法，不影響意義。
- 送出 100 bytes、seq＝1000 → 這段是第 1000–1099 個 byte；對方回 ack＝1100。**增量＝送了幾個 byte，不是 +1**。
- **累積 ACK**：ack＝N 代表 N 之前全部收齊。後面的 ACK 是前面 ACK 的超集。
- **去重**：接收方記著「下一個期待的 byte」。到達的 segment 若涵蓋的 byte 範圍已收過，即為重複，丟棄並仍回 ACK。判斷單位是 **byte 範圍的重疊**，因此重傳合併、部分重疊都能正確拼接。

### 三次握手
- `SYN → SYN/ACK → ACK`，**只在建立連線時做一次**。
- 之後資料直接流動，不再帶 SYN；但每段資料仍會被 ACK（常為累積、延遲批次）。

---

## 5. Buffer 的角色

Buffer 不是單點，是一條鏈：

```
應用層 buffer → 核心 socket send/recv buffer → NIC ring buffer(DMA) → 網卡 FIFO → 沿路交換器/路由器的 buffer
```

核心 socket buffer 的職責：
1. **速率解耦**：生產者與消費者速度、時間點不一致。
2. **send buffer＝重傳儲存**：已送出未被 ACK 的資料必須留著，隨時可能重傳。這就是為什麼視窗左緣要等 ACK 才前進。
3. **receive buffer＝亂序重組**：亂序到達的 segment 暫存，缺口補齊後才依序交給應用（隊頭阻塞的來源）。

軟硬體分工：機械性搬運與切割可下放硬體（checksum offload、TSO/GSO、LRO）；**政策邏輯**（何時重傳、發多快、亂序怎麼重組）仍在軟體。

Header 欄位本身不「做」事，是兩端 TCP 狀態機之間溝通狀態的共同語言。

---

## 6. Sliding Window

從 sender 角度，byte 串流被切成四段：

```
|  已送已ACK  |  已送未ACK（在途） |  可立即送  |  尚不可送  |
|  (已釋放)   |                    |            | (超出視窗) |
              ^-------------- Window ----------^
              左緣：收到 ACK 才前進      右緣：對方騰出空間才前進
```

- Window ＝ 在途 + 可立即送 ＝ `min(rwnd, cwnd)`
- 作用：流量控制、pipelining（避免 stop-and-wait）、與壅塞控制耦合。
- 接收方桶滿時廣告 `Window = 0` → sender 停送，等 window update。這是隔著網路的**背壓**。

### 與 LeetCode sliding window 的關係
共用「有序序列上維護一段連續區間、O(1) 推進邊界」的幾何直覺，但機制不同：

| | LeetCode | TCP |
|---|---|---|
| 範圍 | 單機、純計算 | 跨兩台機器 |
| 目的 | 避免重算，O(n²) → O(n) | 流量控制、pipelining、可靠性 |
| 邊界控制者 | 兩個指標都在自己手上 | 左緣由對方 ACK 推動，右緣由對方 rwnd 與自身 cwnd 決定 |
| 失敗語意 | 無 | 丟包、重傳、逾時 |

---

## 7. 水管模型

```
         粗細 = 頻寬
     ┌──────────────────────┐
 S → │ ≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈≈ │ → R [接收桶 = receive buffer]
     └──────────────────────┘
         長度 = RTT
     容量 = 頻寬 × RTT = BDP
```

- **RTT**（Round-Trip Time）：封包送出到對應 ACK 回來的往返時間。持續變動，TCP 持續量測並算出平滑估計（SRTT），用來決定重傳逾時（RTO）。與應用層的 response time 不同。
- **水**＝在途的 bytes（連續流動，不是一顆顆封包）。回程是**回執單**（累積 ACK）。
- **速率 ≈ 在途量 / RTT**。

| | Flow control | Congestion control |
|---|---|---|
| 變數 | rwnd（receive window） | cwnd（congestion window） |
| 控制方 | 接收方 | 發送方 |
| 保護對象 | 接收桶 | 中間共享網路 |
| 值從哪來 | **被告知**：寫在 header 的 Window 欄位 | **被推測**：從丟包、RTT 等間接訊號推出，沒有欄位 |
| 單位 | bytes（在途存量，不帶時間） | bytes（在途存量，不帶時間） |

sender 實際可在途量 ＝ `min(rwnd, cwnd)`。

相關名詞：**ssthresh**（slow-start threshold）＝ cwnd 在慢啟動與壅塞避免之間切換的門檻。

UDP 沒有 rwnd 也沒有 cwnd：接收緩衝滿了直接丟包且不通知 sender；也不會因網路壅塞自動放慢。需要的應用自己在上層實作（QUIC 全做、WebRTC 挑著做、行情多播用序號 + gap 偵測 + 旁路補發）。

### 跨洲為什麼慢
- 視窗 < BDP：放完一個視窗的量就得空等一個 RTT，管子大半是空的，吞吐被壓在 `視窗 / RTT`。
- 視窗 ≥ BDP：第一張回執回來前就能持續填滿管子，頻寬吃滿。
- **開大視窗／多條連線並行改善的是吞吐，不會縮短單一筆資料的 RTT。**

判斷準則：
- 痛的是「頻寬用不滿」→ 開大視窗（Window Scale）、多連線並行。
- 痛的是「一個來回太久」（realtime、遊戲、互動）→ 只能就近部署，縮短物理距離。

**視窗能填滿管子，但縮不短管子。**

---

## 8. 一次傳輸 + 丟包的完整劇本

設定：每段 100 bytes，起始 seq＝1000，視窗允許 6 段。
S1=1000–1099、S2=1100–1199、S3=1200–1299、S4=1300–1399、S5=1400–1499、S6=1500–1599。

1. **Pipeline 送出**：S1–S6 一次送出，留在 send buffer。左緣 1000、右緣 1600。
2. **穩態滑動**：收到 ACK 1100 → 左緣到 1100、S1 釋放、右緣滑到 1700、可送 S7。
3. **丟包**：S3 掉了，S4–S6 到達。接收方有序只到 1199，每收一段都回 **ACK 1200**（dup ACK）。S4–S6 卡在 receive buffer 等待——隊頭阻塞。
4. **Fast retransmit**：sender 收到 3 個 dup ACK，不等逾時，從 send buffer 重傳 S3。若無後續 segment 觸發 dup ACK，只能等 RTO 逾時重傳（更慢、懲罰更重）。有 SACK 時接收方可明講缺哪段。
5. **補齊**：S3 到達，S3–S6 一次交給應用，回累積 **ACK 1600**。
6. **恢復**：左緣從 1200 跳到 1600，視窗前進。

同時驚動壅塞控制：
- fast retransmit 後進入 fast recovery，cwnd 約砍半（乘性遞減）。
- RTO 逾時則 cwnd 降到 1 MSS，重回 slow start。
- 之後線性回爬（加性遞增）→ **AIMD 鋸齒**。

### 情境推導
- **ACK 在回程掉了**：累積 ACK 自癒，後面的 ACK 涵蓋前面的資訊，通常不觸發重傳。只有後續 ACK 全掉，才由 RTO 逾時重傳；接收方收到重傳資料會判定重複並回 ACK。
- **封包被複製**：依 byte 範圍判定已收過，丟棄並回 ACK。
- **RTT 飆高**：不必然卡死。是否卡住取決於 `min(rwnd, cwnd)` 是否 ≥ 新的 BDP；另外延遲式壅塞控制（Vegas、BBR）會把 RTT 上升視為壅塞訊號而主動縮小 cwnd。兩條因果鏈要分開看。

---

## 9. 壅塞控制設計空間

本質：多個 sender 各自獨立，只能從間接訊號推測共享資源的狀態，是一個**分散式控制問題**。

四條取捨軸：

1. **壅塞訊號**
   - 丟包式（Reno、CUBIC）：等 buffer 溢出才知道，易產生 bufferbloat。
   - 延遲式（Vegas、FAST）：RTT 一漲就退，與丟包式同場競爭會被餓死。
   - 顯式（ECN、DCTCP）：路由器主動標記。
   - 模型式（BBR）：主動估算瓶頸頻寬與最小 RTT，依估算值 pacing。
2. **優化目標**：吞吐量／低延遲／平穩速率。吞吐與延遲是最根本的張力（深 buffer 換吞吐，延遲隨之上升）。
3. **環境**：共享網際網路（須對其他流公平）／自有鏈路（colo、資料中心，可「自私」、可用顯式訊號）。
4. **與可靠性的耦合**：大檔一律重傳；即時影音寧可丟棄遲到資料，改用 FEC + 自適應位元率。

| 原型 | 場景 | 要點 |
|---|---|---|
| CUBIC | 通用、大檔（Linux 預設） | 丟包式，三次曲線探測，填滿長肥管線 |
| BBR | 想同時拿高吞吐與低延遲 | 模型式，估 BDP 並 pacing；估錯時公平性有爭議 |
| DCTCP 等 | 資料中心 | ECN 顯式訊號，微秒級 |
| FIX / 下單 | 專線、低丟包 | 重點是確定性延遲與不丟不亂；關 Nagle（`TCP_NODELAY`） |
| 行情分發 | 一對多 | UDP multicast，序號 + gap 偵測 + 旁路補發 |
| WebRTC (GCC) | 即時影音 | UDP，頻寬估計回饋到**編碼器**調位元率，FEC、jitter buffer |
| LEDBAT | 背景傳輸 | 延遲式、極度謙讓，有人要用就退讓 |

---

## 10. 分層邊界：TCP 到此為止

**TCP 只保證 bytes 從源到目的正確、有序地送達。**

- TCP 是 byte 串流，**不保留訊息邊界**。一次 read 可能是半則訊息，也可能是兩則黏在一起（粘包／拆包）。
- Segment 怎麼切由 MSS、Nagle、TSO 決定，與應用訊息邊界無關。
- 把串流切回訊息叫 **framing**，由上層負責：
  - HTTP/1.1：`Content-Length` 或 chunked
  - gRPC：長度前綴
  - Kafka：每個 request/response 前綴 size
  - WebSocket：自有 frame 格式
- 應用回調拿到「完整訊息」，是 framing 層湊齊的，不是 TCP 遞上來的「完整封包」。
- 自己走 raw TCP，framing、心跳、重連、背壓都得自己實作。

---

## 11. 端到端論證與應用層可靠性

**End-to-End Argument**（Saltzer / Reed / Clark）：「可靠送達」這類保證，只有真正在乎結果的兩個端點靠彼此確認才能完整實現；底層只能做效能優化。

- TCP ACK 只證明「對方核心收到 bytes」。
- 伺服器 `send()` 成功只證明「本機核心收下了」。
- 都證明不了「對方應用收到並處理了」。

### WebSocket / Socket.IO 提供什麼
- **WebSocket（RFC 6455）**：Ping/Pong（活性偵測）、Close 握手。**無訊息級 ACK、無序號、無補發**。
- **Socket.IO**：自有心跳、自動重連、選用 per-message ack。**自動重連 ≠ 補回斷線期間的訊息**。
- Socket.IO 4.6+ 的 Connection State Recovery：以 offset 在短暫斷線後補發（預設保留 2 分鐘）；伺服器重啟會失效、多節點需搭配 adapter、官方明言不保證每次成功，仍需自行處理狀態同步。

### 四塊積木
1. **應用層心跳**：快速偵測連線死亡，把靜默死亡變成可偵測的死亡。只負責偵測，不負責找回訊息。
2. **單調遞增序號 + gap 偵測**：接收端記住最後序號，跳號即知漏了什麼。
3. **應用層 ack + 伺服器 outbox**：未被 ack 的留在待送佇列，重連時或依「我最後收到 N」重送。「寫進 socket」不算送達，「客戶端 ack」才算。
4. **冪等／message-id 去重**：補發必然帶來重複。以業務鍵（如 trade-id）去重，重放第二次為 no-op。

### 傳遞語意
- 只能在 **at-most-once**（發了不管）與 **at-least-once**（補發 + 去重）之間選。
- 「exactly-once」是 at-least-once + 冪等處理做出來的效果，不是傳遞層的保證。
- 不要 stop-and-wait：低頻事件可 per-message ack + 逾時；高頻資料用累積 ack（類似 TCP 滑動視窗）。
- 逾時太短會誤判狂補、太長恢復慢；緩衝要有界 + TTL，長缺口改從持久化日誌重放。

---

## 12. 「伺服器推了、客戶端沒收到」的排查框架

### 情況 A：客戶端有斷線／重連紀錄 → 半開連線（half-open）
- 連線在 WAN 上靜默死亡（Wi-Fi 瞬斷、NAT／防火牆逾時清除狀態、行動網路切換），沒有 FIN/RST。
- 伺服器端 socket 仍是 `ESTABLISHED`，`write()` 成功、無 error log，但 bytes 永遠到不了。
- TCP 最終靠重傳逾時發現，可能是數十秒後；沒開 keepalive 可能不主動發現。
- 重連開的是新連線，不知道舊連線丟了什麼。

### 情況 B：客戶端沒有斷線紀錄卻仍漏 → 嫌疑轉回兩端
在沒斷的 TCP 連線中 byte 串流不會掉，因此：

**B1：bytes 有到客戶端，死在應用層**
- 回調例外被吞、反序列化失敗被靜默吞掉。
- 頁面切換／重新訂閱瞬間到達，處理器正在拆除重建，沒人接。
- 接收執行緒 dispatch 到 UI 執行緒時佇列滿或 UI 被阻塞。

**B2：訊息沒進入該客戶端的串流，伺服器不視為錯誤**
- 多節點路由漏接：客戶端連 A 節點，事件在 B 節點發佈，跨節點 pub/sub 或 sticky session 沒接上。
- 訂閱競態：事件觸發時客戶端尚未完成訂閱，被過濾掉。
- 慢消費者背壓：送出緩衝溢出時框架直接丟訊息，僅記 debug 級或不記。

### 一刀切開的診斷問題
**「那些 bytes 有沒有抵達客戶端那台機器？」**
- 客戶端抓包（Wireshark / tcpdump）：有到＝應用層吞了；沒到＝伺服器未送出或路由漏接。
- 更省事：訊息帶序號。跳號且無斷線事件＝傳輸中或伺服器端遺失；序號連續但業務沒觸發＝應用層吞了。

---

## 13. 推送鏈路的「慢／卡」常見真因

先量測再換協議。WebSocket 本身就在 TCP 上，改走 raw TCP 是橫向移動，TCP 的隊頭阻塞、握手、壅塞行為都不變。

| 症狀來源 | 檢查與處置 |
|---|---|
| Socket.IO 退化成 long-polling | 確認實際 transport 是 WebSocket |
| Nagle + delayed ACK | `setTcpNoDelay(true)` |
| Nginx 緩衝 | WebSocket 反代設 `proxy_buffering off`、調整 timeout |
| 過期報價排隊 | **conflation**：新值覆蓋舊值時丟棄佇列中的舊值 |
| 切頁時畫面空白 | **snapshot + delta**：訂閱即給當前快照，之後推增量 |
| 導航時訂閱 churn | 避免整個拆除重建訂閱 |
| UI 執行緒重繪壓力 | 節流、合併後再更新 |
| 地理延遲 | 唯一解是就近接入（edge / PoP） |
| WAN 掉包導致 HoL | 只有 QUIC 系（WebTransport）在協議層解掉 |

---

## 14. 即時推送方案對照

| 方案 | 可靠補發 | 訂閱即快照 | 特點與限制 |
|---|---|---|---|
| Socket.IO + Connection State Recovery | 部分（短斷線、offset） | 需自建 | 不換棧的止血方案；重啟失效、多節點需 adapter |
| Centrifugo | 有：頻道 history + 序號，重連自動補、保證順序 | 有：cache recovery 給最新一筆 | 獨立即時層，後端以 API 發佈；delta 壓縮、Protobuf、單連線多訂閱；recovery 針對短斷線，長缺口需回源拉全量；需確認 .NET client 支援 |
| MQTT（EMQX、HiveMQ；.NET 用 MQTTnet） | QoS 1（at-least-once） | retained message | broker 模式，通用性高，適合內外統一 pub/sub |
| SignalR | 無跨重連保證 | 需自建 | .NET 原生，連線管理與 fallback 完整，可靠性要自己疊 |
| WebTransport（HTTP/3 / QUIC） | stream 可靠、datagram 不可靠，可依資料類型分流 | 需自建 | 2026 年 3 月起主流瀏覽器 Baseline；解掉 TCP HoL；Java 端 server 生態仍在成熟 |
| NATS JetStream / Kafka | 持久化日誌、offset 重放 | — | 後端事件骨幹，不直接面向客戶端 |
| Netty / gRPC | TCP 可靠，訊息級需自建 | — | 仍在 TCP 上，HoL 不變；gRPC 瀏覽器端需 gRPC-web |
| Aeron | UDP、微秒級 | — | 機房內服務間通訊，不面向客戶端 |
| raw TCP | 需全部自建 | 需自建 | framing、心跳、重連、背壓全要自己做 |

資料分類決定選型：
- **交易事件（成交、跟單）**：低頻、每則都要到、要順序、要去重 → at-least-once + 冪等。
- **行情、配置**：高頻，痛點是卡而非漏；中間可丟，但最新值必須正確且在 → snapshot + delta + conflation。純 fire-and-forget 不保證最新值會到，因此不適用 WebRTC 式模型。

---

## 15. 常見誤區與修正

| 誤區 | 修正 |
|---|---|
| TCP header 含來源／目的 IP | IP 在 L3；TCP 只有 port |
| Window 是訊息大小 | Window 是接收方剩餘容量（rwnd） |
| Data Offset 與排序有關 | 它標示 header 長度；排序靠 seq/ack |
| L3 是 TCP/UDP、L4 是 IP | L3＝IP，L4＝TCP/UDP |
| 每則訊息都要三次握手 | 握手只在建立連線時一次 |
| 握手後就 fire-and-forget | 資料仍持續被 ACK |
| 序號每個封包 +1 | 序號是 byte 編號，增量＝送出的 byte 數 |
| 應用收到的是 TCP 的完整封包 | 收到的是 framing 層組好的完整訊息 |
| 寫入成功＝對方收到 | 寫入成功＝本機核心收下 |
| 自動重連＝補回訊息 | 重連只是新連線，不知道舊連線丟了什麼 |
| RTT 高＝視窗卡死 | 取決於視窗是否 ≥ BDP，以及壅塞控制是否主動縮 cwnd |
| rwnd / cwnd 是速率 | 兩者都是在途存量（bytes），速率是推導結果 |
| 開大視窗能讓跨洲變快 | 只改善吞吐，不縮短 RTT |
| 改走 raw TCP 能解卡頓與漏訊息 | WebSocket 已在 TCP 上；要解 HoL 需 QUIC 系 |
| 0-RTT 的風險是髒資料 | 風險是 replay；只能承載冪等操作 |

---

## 16. 本質 vs 偶然複雜

本質複雜來自四條約束：
1. 介質不可靠（丟、亂序、重複、延遲變動）
2. 資源共享且無中央協調
3. 兩端無共享時鐘與記憶體，只能靠延遲且可能遺失的訊息推測對方狀態
4. 要在上述弱基底上對上層提供強保證

對應的核心機制：序號定位、ACK 消除不確定性、視窗管流量與 pipelining、計時器 + 重傳應對丟包、AIMD 應對共享。

其餘（dup ACK 門檻、SACK 格式、RTO 公式、各家壅塞控制變體）屬偶然複雜，可由上述模型推導或查閱。

---

## 17. 自我檢驗題

用模型推導，不靠記憶：

1. 接收方應用長時間不讀資料，sender 會發生什麼？依序描述 rwnd 與 sender 行為。
2. Window Scale 協商失敗、只能用 65535 bytes 視窗，RTT 150 ms 的鏈路單連線吞吐上限約多少？
3. 為什麼累積 ACK 讓零星 ACK 遺失不需要重傳？什麼情況下仍會觸發 RTO？
4. 重傳時把 S3+S4 合併成一個 segment 送出，接收方已收過 S4，如何處理？
5. 伺服器 `send()` 成功、無錯誤紀錄，客戶端也無斷線紀錄，但業務未觸發。列出可能原因與區分方法。
6. 為什麼行情資料適合 conflation，而成交事件不行？
7. 只把 Socket.IO 換成 raw TCP，哪些問題會消失、哪些不變、哪些變成要自己做？
8. QUIC 的多 stream 解掉了什麼？為什麼 HTTP/2 的多工解不掉？
