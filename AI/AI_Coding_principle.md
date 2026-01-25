# AI Coding原則

1. 對話盡可能透過以下模式引導AI
   1. What: 專案是什麼、問題是什麼 (CLAUDE.md)
   2. Context: 當前的情境，有什麼限制 (Context, Rules)
   3. Who: 誰來做，專職任務的專家 (Agent)
   4. How: 具體怎麼做 (Skill)
2. 需要做不同新任務，開啟一個新的對話，保護token數量

## 功能模塊

- CLAUDE.md: 專案核心，讓AI理解這個專案
- Rule: 讓AI一定要遵守的部分
- Agents: 負責專職任務，例: 負責**code review**, 檢查**coding style**
  - Sub-agent: AI實際開啟獨立對話來處理對應的專職任務
- Skills: 參考手冊，指導AI該怎麼做的標準流程
- Commands: 將一整套SOP指令封裝成一個指令
- Hooks: 事件鉤子
- Contexts: 情境模式，目前有三種可參考，**dev**、**review**、**research**

## 參考資料

[everything-claude-code](https://github.com/affaan-m/everything-claude-code?tab=readme-ov-file)
