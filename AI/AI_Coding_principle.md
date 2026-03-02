# AI Coding 使用技巧

1. *CLAUDE.md* 是專案的核心，除了透過一開始的*init*指令產出的內容外，應該涵蓋以下幾點
   1. 代碼規範
   2. 技術棧
   3. 專案結構
   4. 每次應該做的事
   5. 每次不應該做的事
2. 每當AI犯錯，就讓它將錯誤更新到*CLAUDE.md*，避免再犯
3. *CLAUDE.md* 應該保持*精簡內容*、並*持續更新*，內容大小應控制在 2.5K tokens 左右(大約 200 行內)，官方有提供[API](https://platform.claude.com/docs/en/build-with-claude/token-counting)來測量
4. 要求AI編寫測試，並依據領域的不同，有多樣化的驗證手段(前端、後端、手機端等)
5. 需要做不同新任務時，像開發新功能或是除錯，都開啟新對話或是清除對話緩存，保護上下文邊界
6. MCP (Model Context Protocol) 能夠幫助AI與外部資源互動，像是可以連接telegram讓AI執行完任務後回報到tg上，實現遠程操控，或是讓AI輸出到Notion進行項目追蹤
7. 重複性的工作讓AI整理成*commands*
8. 隔離不同類型任務，*code review*、*design solution*、*refactor*、*implement code*，這些都應該由各自的agent操作
9.  進行重要任務，可以改用*Plan Mode*，避免AI直接改代碼
10. 養成「先計畫、後執行、必驗證」，AI生產的代碼，人需要對其內容進行檢核跟理解
11. *SKILL*可以透過AI生成，完成一次複雜任務後，跟AI說「將我們剛才的操作存為一個新技能」
12. *SKILL*中可以將更多的細節與選項放到*reference*下，同時也能在*scripts*放一些代碼讓AI執行，來產生與外部的溝通能力
    1.  假設一個會議總結的SKILL，會有以下的目錄結構
```
meeting-summary/
├── SKILL.md              ← 永久載入（Skill 主描述與指令）
├── reference/
│   ├── fiscal_policy.md  ← 按需載入（財政原則參考文件）
│   └── statistics.md     ← 按需載入（統計原則參考文件）
└── scripts/
    └── upload.py         ← 按需載入（上傳/後處理腳本）
```
1.  利用關鍵字觸發特定技能或是MCP工具
2.  利用*Hook*，可以指定AI開始前跟開始後做特定行為，像是讓AI每次寫完代碼做*Format*的動作，保持可讀性
3.  針對AI的個人偏好使用，可以讓AI記憶起來，它會存到專案下的一個*MEMORY.md*
4.  CLAUDE 有很多有用的插件，例如[Ralph Loop](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/ralph-loop)很適合複雜大型任務
5.  RAG (Retrieval-Augmented Generation) 檢索增強生成，能幫助 LLM 生成回答前，先從資料庫中檢索相關資訊，再進行回答，能有效幫助AI減少幻覺

## 參考資料

- [那些 Agent 神文没告诉你的事：照着做，系统只会更烂 【AI agent 搭建实操指南】](https://youtu.be/b_9D7T0n4RA?list=TLGGtH-HmxCStmIyMDAyMjAyNg)
- [Claude Code's Creator Does This Before Every Single Project](https://youtu.be/B-UXpneKw6M?list=TLGGyrZG6_oOcgMyMDAyMjAyNg)
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code?tab=readme-ov-file)
