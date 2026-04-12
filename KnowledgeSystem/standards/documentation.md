# Documentation Standards

文件撰寫的跨專案規範。具體模板見 `templates/`。

<!--
📌 預期擴展：
- 圖的繪製工具與格式規範（Mermaid / Draw.io / PlantUML 的選用）
- 文件 review 流程（誰 review、多久更新一次）
- API 文件的自動生成策略
- 變更日誌（CHANGELOG）的撰寫規範
- 文件與代碼同步的機制（CI 檢查文件是否過期）
-->

---

## 文件層級

每個專案至少包含以下文件，各層覆蓋不同的認知需求：

| 文件類型 | 回答的問題 | 位置 |
|----------|-----------|------|
| System Narrative | 系統是什麼、怎麼運作 | `docs/design/` |
| Design Doc | 為什麼這樣設計 | `docs/design/` |
| ADR | 單一決策的 why | `docs/adr/` |
| 互動圖 | 誰跟誰溝通 | `docs/diagrams/` |
| README | 怎麼開始 | 專案根目錄 |

## 撰寫原則

- 圖表達關係（who talks to whom），敘事表達職責（who does what）。兩者互補，不重複。
- 每張圖只服務一個縮放層級。Container 圖不混 Component 細節。
- ADR 一旦寫入不修改內容，僅更新狀態（Accepted → Superseded / Deprecated）。
- 過期的文件比沒有文件更危險。如果無法維護，明確標註「⚠️ 可能過期」。
