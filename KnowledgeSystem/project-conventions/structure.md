# Project Structure Conventions

跨語言適用的專案目錄結構原則。語言特定的結構見 `languages/` 子目錄。

<!--
📌 預期擴展：
- 其他語言的目錄結構慣例（Go、Rust、Python）
- Monorepo vs Polyrepo 的選用條件
- 第三方依賴的管理策略（vendoring vs package manager）
-->

---

## 通用原則

- 原始碼、測試、文件、腳本各自分目錄，職責不混合。
- 測試目錄結構鏡射原始碼目錄結構。
- `build/` 與 IDE 配置不進版本控制。
- README.md 是最外層入口，包含一句話描述 + 構建指令 + 文件索引。

## 標準目錄

```
{project}/
├── src/                    # 原始碼
├── tests/
│   ├── unit/               # 單元測試（鏡射 src/ 結構）
│   └── integration/        # 整合測試
├── docs/
│   ├── design/             # System Narrative + Design Doc
│   ├── adr/                # Architecture Decision Records
│   └── diagrams/           # 互動關係圖
├── scripts/                # CI/CD、分析、自動化腳本
├── README.md
└── .gitignore
```

## Application 層職責

- 應用程式入口點（main）不包含業務邏輯，僅作為啟動點。
- 存在一個「組裝層」（如 `app/`），負責元件初始化、依賴注入、生命週期管理。
- 組裝層是唯一知道所有模組存在的地方；其他模組彼此不直接建構對方。
