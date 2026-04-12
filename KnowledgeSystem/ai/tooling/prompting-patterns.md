# Prompting Patterns

反覆驗證有效的 prompt 模式。當某個 pattern 在三個以上場景穩定有效時，考慮升級為規範。

<!--
📌 預期擴展：
- 依任務類型分類（代碼生成、code review、文件撰寫、debug、架構分析）
- 每個 pattern 附上「有效場景」與「失效場景」
- 與 collaboration-protocol.md 的決策權歸屬對照
- Anti-patterns：常見但效果差的 prompt 方式
-->

---

## 結構原則

- **Context → Constraint → Task**：先給背景，再給約束，最後給任務。AI 需要知道邊界才能做出好的判斷。
- **用約束而非指令**：「不允許在 hot path 上做 heap allocation」比「請用 stack allocation」更有持久價值。
- **提供正例與反例**：光說「好的代碼」不夠，附上具體的 good example 和 bad example。

## 已驗證的 Pattern

### Incremental Implementation
先讓 AI 產出骨架（介面 + 空實作），確認結構正確後再逐模組填入實作。
避免一次產出整個系統導致結構偏離預期。

### Constraint-Driven Review
在 code review 時，不問「這段代碼好不好」，而是問「這段代碼是否違反了以下約束：{列出約束}」。
具體的約束比模糊的「好」更能得到有用的回饋。

### Spec-First Development
先用 OpenSpec 或類似流程產出 proposal → design → tasks，確認規格後再讓 AI 實作。
規格是人類的決策產物，實作是 AI 的執行產物。
