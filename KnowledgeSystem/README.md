# KnowledgeSystem

個人軟體工程知識體系。從實戰經驗中提煉，供人類閱讀與 AI 開發共用。

## 結構總覽

```
KnowledgeSystem/
├── TODO.md                              # 待調整清單：第一版體檢結果與執行順序
├── skills/                              # 程序性 skill 層：可觸發的 workflow
│   └── start-cpp-project/SKILL.md       # 綠地 C++ 專案 + CLAUDE.md 生成
├── principles.md                        # 工程信念：不隨專案變化的價值判斷
├── standards/                           # 工程標準：跨專案適用的可執行規則
│   ├── code-quality.md
│   ├── testing.md
│   └── documentation.md
├── templates/                           # 文件模板：標準化的產出格式
│   ├── system-narrative-template.md
│   ├── design-doc-template.md
│   ├── adr-template.md
│   └── review-checklist.md
├── ai/                                  # AI 協作：人機協作契約與工具配置
│   ├── collaboration-protocol.md
│   ├── quality-gates.md
│   └── tooling/
│       ├── claude-code.md
│       └── prompting-patterns.md
└── project-conventions/                 # 專案慣例：因專案而異的具體決策
    ├── structure.md
    ├── testing-strategy.md
    ├── design-principles.md
    └── languages/
        └── cpp/
            ├── conventions.md
            ├── build.md
            ├── concurrency.md
            └── checklist.md
```

## 分層邏輯

| 層級 | 變更頻率 | 受眾 | 範例 |
|------|----------|------|------|
| principles | 幾乎不變 | 所有人 | 「未經測試的代碼是未完成的代碼」 |
| standards | 一年數次 | 工程師 + AI | 「公開 API 必須有錯誤處理」 |
| project-conventions | 每個專案不同 | 該專案的開發者 + AI | 「本專案用 Expected<T,E> 而非 exception」 |

## 使用方式

**人類閱讀**：從 `principles.md` 開始，了解核心信念，再按需進入 standards 或 project-conventions。

**AI 開發**：在專案的 CLAUDE.md 中用 `@import` 引用所需的層級。典型配置：
```markdown
@path/to/KnowledgeSystem/standards/code-quality.md
@path/to/KnowledgeSystem/standards/testing.md
@path/to/KnowledgeSystem/project-conventions/languages/cpp/conventions.md
```

## skills 層

`skills/` 是「動詞」——可被 Claude 自動觸發的程序性 workflow。
其餘層級（principles / standards / project-conventions / templates）是「名詞」——skill 引用的工程標準。

與通用工具（Superpowers / agent-skills）的分工：本體系不重做 TDD / brainstorm / review 等通用 workflow，只提供領域知識與少數領域 skill。通用引擎跑在本體系產出的 CLAUDE.md 之上，於是對著自己的標準執行。

> 打包成 Claude Code plugin 對外發佈的部分暫緩，待 `TODO.md` 的調整完成後再處理。

## 演進原則

- 每條規範在至少一個實際專案中驗證過才寫入
- 技巧在三個以上場景反覆使用後，升級為規範
- 內容精煉優先於覆蓋全面——寧可少而準，不要多而泛
