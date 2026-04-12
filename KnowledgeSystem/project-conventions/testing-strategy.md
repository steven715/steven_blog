# Testing Strategy

跨語言適用的測試策略。具體的測試框架配置見 `languages/` 子目錄。

<!--
📌 預期擴展：
- Contract testing 策略（取代部分難以執行的 e2e 測試）
- 效能基準測試的設計與歷史比較方法
- Chaos testing / Fault injection 的適用場景
- 測試資料管理策略（fixture、factory、builder）
-->

---

## CI 階段規劃（漸進實施）

### 階段一：測試基礎設施
- 建立 `tests/unit/` 結構，引入測試框架。
- 構建配置支援獨立測試執行。
- 至少對核心業務模組撰寫單元測試。

### 階段二：Pre-commit 守門
- 配置 pre-commit hook，執行單元測試。
- 未全部通過則阻止 commit。
- 引入代碼格式化工具確保風格一致。

### 階段三：CI Pipeline
- 自動化構建 + 全量測試（unit + integration）。
- 靜態分析 + 動態分析（Sanitizer）。
- 效能基準測試，與歷史數據比較。
- 通過所有標準後才標記為可發布。
