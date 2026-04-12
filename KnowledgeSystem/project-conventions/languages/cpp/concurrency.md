# C++ Concurrency Conventions

效能敏感路徑與併發模型的規範。

<!--
📌 預期擴展：
- SeqLock 的具體實作規範與使用場景
- Lock-free queue 的選型標準
- 欄位寫入權歸屬模型（immutable / trade-mutable / calc-mutable）
- TOCTOU 問題的處理策略
- 變更通知機制的選擇（polling + version check vs eventfd）
- ThreadSanitizer / AddressSanitizer 的 CI 配置
- std::atomic 的 memory order 選用指引
-->

---

## 效能關鍵路徑

在效能敏感的路徑上（行情處理、風控計算）：

- 偏好 lock-free 資料結構與 zero-allocation 策略。
- 使用 `std::atomic`、SeqLock 等低開銷同步原語。
- 避免在熱路徑上做序列化（BSON/JSON encode）或同步 I/O。
- 熱路徑上禁止使用 `std::map`，偏好 `flat_map` 或 `unordered_map`。

## 共享狀態規範

- 所有共享狀態在宣告處標明保護方式（註解標明鎖類型）。
- 讀寫分離：多讀少寫場景用讀寫鎖或 SeqLock。
- 浮點數的共享寫入使用 `std::atomic<double>`，不依賴讀鎖保護寫入操作。

## 分析工具

- `perf` + flame graph：效能瓶頸定位
- ThreadSanitizer：data race 檢測
- AddressSanitizer：記憶體錯誤檢測
