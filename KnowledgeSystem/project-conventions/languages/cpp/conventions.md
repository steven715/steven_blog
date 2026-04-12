# C++ Conventions

C++ 特有的型別設計、命名與模組通訊規範。

<!--
📌 預期擴展：
- 錯誤處理的具體模式：Expected<T,E> vs error code 的選用條件
- RAII 的具體實踐規範（no raw owning pointers）
- Smart pointer 選用規則（unique_ptr vs shared_ptr 的判斷）
- Template / Generic programming 的使用邊界
- C++17/20 特性的採用策略
- Header include 順序規範
-->

---

## struct vs class

| 特徵 | struct | class |
|------|--------|-------|
| 用途 | 貧血資料模型 | 有行為的業務物件 |
| 成員 | 全 public 資料欄位 | private 資料 + public 介面 |
| 生命週期 | 由外部 owner 管理 | 自身管理或由明確 owner 管理 |

- struct 名稱使用資料語意後綴：`Data`, `Info`, `Snapshot`, `Config`, `Entry`
- class 名稱使用行為語意：`Manager`, `Evaluator`, `Processor`, `Publisher`

## 命名慣例

- 變數與函數：`snake_case`
- 類別與結構：`PascalCase`
- 常數：`kPascalCase` 或 `UPPER_SNAKE_CASE`
- 私有成員：尾綴下劃線 `member_`
- 測試檔名：`{原始檔名}_test.cpp`
- ADR 檔名：`NNNN-{簡短標題}.md`

## 模組間通訊

- 模組間互動僅透過 callback 或 EventBus，不允許直接建構或持有另一模組的實例。
- 資料擁有者提供唯讀存取介面（`const&` 或 `const*`）。
- 中介軟體（EventBus、網路連線）的生命週期由 Application 統一管理。
