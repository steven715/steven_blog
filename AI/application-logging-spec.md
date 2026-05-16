# 應用程式日誌規範

> 本文件為個人開發規範，適用於後端服務、交易系統、基礎設施等應用層程式。
> 庫（library）層的日誌規範請參考獨立文件，因消費者與控制權歸屬不同。

---

## 1. 設計原則

### 1.1 核心理念

日誌的存在目的是 **重建事件現場**，而不是記錄一切。每一條日誌都應該能回答以下至少一個問題：

- 發生了什麼？（what）
- 為什麼會發生？（why）
- 影響了什麼？（impact）
- 接下來會怎樣？（action taken）

---

## 2. 日誌格式規範

### 2.1 基本欄位（沿用 muduo 慣例並擴展）

```
<timestamp> <level> <thread> <file:line> <event_key=value pairs>
```

範例：

```
2026-05-16 14:23:01.234567 INFO  worker-3 quote_engine.cpp:142 event=service_started version=2.3.1 symbol_count=4823
```

**強制欄位**：

- `timestamp`：微秒級精度
- `level`：等級（見第 3 節）
- `thread`：線程名或 ID
- `file:line`：發生位置（原始碼檔名與行號），由 logging 框架自動帶入（如 `__FILE__:__LINE__`）
- `event`：事件名稱，使用 snake_case

**理由**：服務本身已是查看日誌時的已知前提（從哪個檔案或集群來），不需再記模組名；相對地，**精確的發生位置**能讓人在沒有 trace 的情況下，直接跳到原始碼脈絡。

### 2.2 結構化原則

message 部分必須使用 `key=value` 格式，避免自由文本。

✅ **推薦**：
```
event=order_submit symbol=TXFF6 side=B qty=10 latency_us=42
```

❌ **避免**：
```
Submitted order for TXFF6, buy 10 contracts, took 42us
```

理由：結構化格式可以直接被 awk、jq、ELK 解析，無需後續寫 parser。

### 2.3 欄位命名一致性

同一個概念在所有日誌中必須使用同一個 key，建立全局命名表：

| 概念 | 統一 key |
|------|---------|
| 延遲（微秒） | `latency_us` |
| 延遲（毫秒） | `latency_ms` |
| 連接識別 | `conn_id` |
| 訂單識別 | `order_id` |
| 商品代碼 | `symbol` |
| 錯誤碼 | `error_code` |
| 對端地址 | `peer` |

**理由**：後續告警規則、儀表板查詢、日誌聚合的維護成本，與命名一致性成反比。

---

## 3. 日誌等級規範

採用四級劃分。判斷依據：**是否需要人介入** > 是否影響運行。

### 3.1 ERROR — 需要立即處理

**定義**：功能失敗或服務異常，需要 oncall 介入。

**內容要求**：必須包含 what failed、why、影響範圍、已採取的動作，讓人不需要翻其他日誌就能初步判斷。

**範例**：

```
ERROR worker-3 order_router.cpp:287 event=order_submit_failed order_id=88471 symbol=TXFF6
      error_code=E_UPSTREAM_TIMEOUT upstream=tse-gw-2 timeout_ms=500
      action=order_rejected_to_client client_id=C1042
```

**典型場景**：
- 訂單路由失敗且無法重試
- 配置載入失敗導致服務無法啟動
- 關鍵下游服務完全不可達
- 資料完整性校驗失敗

### 3.2 WARN — 異常但已處理，需關注趨勢

**定義**：發生異常但內部已處理，單條無意義，**趨勢**才有意義。

**內容要求**：應包含累計指標，幫助判斷是否升級為事件。

**範例**：

```
WARN  io-1 upstream_session.cpp:94 event=connection_reset peer=10.0.1.5:9100
      retry_count=3 next_retry_in_ms=200 cumulative_failures_1m=12
```

**典型場景**：
- 連接中斷但自動 failover 成功
- 訂閱合約失敗（部分失敗，整體服務可用）
- 接近 buffer/queue 容量上限
- 配置中有不認識的欄位（向前相容處理）

**反例（不該用 WARN）**：
- 「看了也不會處理的事」→ 應該 INFO 或 DEBUG
- 「影響功能的錯誤」→ 應該 ERROR

### 3.3 INFO — 狀態變化與業務里程碑

**定義**：服務狀態變化、業務關鍵事件、生命週期里程碑。

**頻率原則**：穩態下頻率應該**很低**。如果 INFO 每秒幾百條，大部分應該降為 DEBUG。

**內容要求**：啟動類 INFO 應一行涵蓋所有 ops 想知道的快照資訊。

**範例**：

```
INFO  main main.cpp:58 event=service_started version=2.3.1 config_hash=a8f3e2
      subscribed_sources=[BLP,CTP,KG] symbol_count=4823 listen=0.0.0.0:7700
```

**典型場景**：
- 服務啟動、關閉、配置重載
- 訂閱了哪些 source
- 主要組件初始化完成
- 業務關鍵事件（例如交易日切換、結算開始）

**邊界判斷**：
- 「行情服務器訂閱哪些合約」的摘要 → INFO
- 「每個合約逐筆訂閱」→ DEBUG

### 3.4 DEBUG — 開發者排查用

**定義**：生產環境預設關閉，需要時可動態打開協助排查。

**內容要求**：應包含足夠資訊重建場景。

**範例**：

```
DEBUG quote tick_dispatcher.cpp:213 event=tick_received source=BLP symbol=2330
      seq=8847291 bid=585.0 ask=585.5 latency_from_exchange_us=142
```

**典型場景**：
- 單一合約逐筆訂閱明細
- 內部狀態轉換
- 條件分支的選擇結果
- 重要函式的進入/離開（針對問題排查）

---

## 4. 檔案組織策略

### 4.1 單一日誌檔

**核心目標**：跨模組、跨服務的事件能在時序上對齊重建。

**實作建議**：

- 當天日誌統一寫入 `app.log`，不依等級分檔（ERROR 的監控改由日誌系統 / metrics 處理，不靠檔案切割）
- 每條日誌都帶足夠的 correlation 欄位（`trace_id`、`session_id`、`order_id`），方便聚合重建
- 透過 ELK / Loki 等系統做邏輯統一查詢

**理由**：分等級檔案的成本是「重建場景時要 merge 多個檔案」，而 ERROR 的告警價值早已被 metrics / 日誌系統承擔，沒必要再付這個成本。

### 4.2 跨服務時間對齊

**前提條件**：所有機器必須 NTP 同步，理想情況下 PTP 同步到微秒級。

**理由**：日誌單檔還是多檔的價值，遠不如時間能對得起來重要。

### 4.3 Rotation 策略

採用「**跨日 + 大小超限**」雙條件輪轉：

- **跨日輪轉**：每日 00:00 將前一日的 `app.log` 改名為 `YYYYMMDD_app.log`（如 `20260516_app.log`），新的一天從空的 `app.log` 開始。
- **大小輪轉**：單一檔案超過 **500 MB** 時，於檔名後加上序號做切割（如 `20260516_app.log.1`、`20260516_app.log.2`），同日內依序累加。
- 避免 rotate 時產生 gap，使用 copytruncate 或 reopen 訊號。
- 保留策略：依硬碟容量決定保留 N 天，過期檔案自動清除。

---

## 5. 業務 Profile 規範

業務 profile（延遲、吞吐、佇列深度等）統一以 **INFO 等級**寫入 `app.log`，**每分鐘輸出一次彙總統計**，與事件日誌共用同一條時序。

**範例**：

```
INFO  worker-3 profile_reporter.cpp:45 event=profile_minute window=60s
      tick_to_trade_us_p50=82 tick_to_trade_us_p99=412 tick_to_trade_us_max=8521
      quote_latency_us_p50=15 quote_latency_us_p99=89
      orders=12483 ticks=2845611 queue_depth_max=147
```

**要點**：

- 每分鐘一次：用最小粒度承載 P50 / P99 / max / count 等彙總值，避免逐筆寫入造成噪音。
- 在程式內聚合：應用層自己維護 1 分鐘的滑動視窗，到時 flush 一行。
- 離群事件另外以 WARN 記錄，附上當時的值與閾值，便於回查：

```
WARN  worker-3 latency_monitor.cpp:67 event=latency_outlier metric=tick_to_trade_us value=8521
      threshold=2000 order_id=88471 symbol=TXFF6
```

---

## 6. 檢查清單（Code Review 用）

新增日誌時的自檢項目：

- [ ] 等級是否符合定義？（特別注意 WARN/INFO 邊界）
- [ ] 是否使用結構化 key=value 格式？
- [ ] 欄位名稱是否符合全局命名表？

---

## 7. 參考資料

- 《Linux 多線程服務端編程》陳碩 — muduo 日誌設計
- Google SRE Book, Chapter 16 — Logging
- Honeycomb: Observability 三支柱（Logs / Metrics / Traces）
- OpenTelemetry Logging Specification
- etcd（zap）— 分散式系統日誌實作參考
- Envoy — 高頻網路庫日誌處理參考

---

## 修訂紀錄

| 日期 | 版本 | 變更 |
|------|------|------|
| 2026-05-16 | 1.0 | 初版 |
| 2026-05-16 | 1.1 | 移除 `module` 欄位，改以 `file:line` 標示發生位置 |
| 2026-05-16 | 1.2 | 檔案組織改為單一 `app.log`，跨日改名為 `YYYYMMDD_app.log`，單檔超過 500 MB 才額外切割 |
| 2026-05-16 | 1.3 | 業務 profile 簡化為 INFO 等級，每分鐘輸出一次彙總統計 |
| 2026-05-16 | 1.4 | 暫時移除安全與隱私、效能與可靠性、錯誤碼規範三節 |
