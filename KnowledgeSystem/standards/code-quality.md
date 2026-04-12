# Code Quality Standards

從 principles 推導出的具體可執行規則。跨專案適用。

<!--
📌 預期擴展：
- 函數複雜度上限（cyclomatic complexity）
- 註解規範：何時該寫、何時不該寫
- 錯誤處理的具體 pattern 分類（error code vs Expected<T,E> vs exception 的選用條件）
- 命名的通用規則（與語言特定規則區分開）
- 禁止清單：magic numbers、深層巢狀、god class 等
-->

---

## 函數

- 一個函數只做一件事。如果需要用「然後」來描述它的行為，考慮拆分。
- 函數長度不超過 40 行。超過時提取子函數，以有意義的名稱取代註解。
- 參數數量不超過 4 個。超過時用 struct 封裝為參數物件。

## 模組

- 每個模組（class / 檔案）遵守單一職責原則。
- 模組間透過介面（callback、抽象基底類別、事件）協作，不直接建構或持有對方實例。
- 公開 API 必須有錯誤處理，不允許未處理的失敗路徑。

## 共享狀態

- 所有共享狀態必須有明確的鎖策略，並在宣告處以註解標明保護方式。
- 寫入操作限制為單一維度——一次寫入只修改一個業務物件的狀態。

## 版本控制

- Commit message 遵循 conventional commits 格式。
- 不使用 magic numbers；使用命名常數。
