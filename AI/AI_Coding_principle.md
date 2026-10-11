# AI Coding 使用技巧

更新：2026-10（Claude Code 2.1.28x）。工具變得快，這份只留目前還成立的原則；版本相關的細節看官方 changelog。

## CLAUDE.md

1. 只放**導航指標**與**重複出現的失敗模式**，不放代碼規範、技術棧、禁止清單。每一行都常駐 context，行數越多每行的權重越低。
2. AI 犯錯時先分類，再決定放哪：
   - 可機械判定（格式、檔案位置、禁用 API）→ hook 或 lint，不寫規則
   - 需判斷（職責劃分、錯誤有沒有真的被處理）→ `CODING_STANDARDS.md`，只給 review 階段讀
   - 都不是 → 丟掉
3. 新專案可以先不寫 CLAUDE.md，等失敗模式出現再加。換模型時重審一次，用 `/doctor prompt-audit` 找為舊模型寫的 prompt。
4. 記憶由 Claude Code 自己管（auto-memory + Auto Dream 整理），不再手寫 MEMORY.md。`/memory` 看狀態；整理後看 diff，不看整理後的檔案。

## 流程

5. 寫碼前先把需求問清楚：要求 AI 只問不答，每題給選項；決策是人的事，查事實是 AI 的事。前面省的時間會在「不是這樣，重做」付回來。
6. 大任務用 Plan Mode 先出計畫再動手。要求 AI 在做完後列出**考慮過但沒做的方案**（implementation notes），人審的是那張清單，不是整份 diff。
7. 寫碼與審查分開：實作 agent 不讀 coding standard，review agent 才讀。可再用另一個模型做對抗式審查（「這是同事寫的，找出漏洞」）。
8. 不同任務開新對話或 `/clear`：開發、除錯、review 各自一個 context。Session 之間用 worktree 隔離。
9. 要求測試，但要指定方法：property-based、deterministic simulation、oracle 重放。AI 寫測試會偷懶，只驗「沒有 throw」的測試不算。

## 自動化

10. 有明確完成條件的任務才交給 loop：`/goal` 設判準、`/background` 脫離終端跑、`/loop` 定時重跑。沒有可機械判定的「完成」就不要 loop。
11. 長時間自主跑的四件套：計畫寫在 CLAUDE.md、`CHANGELOG.md` 記進度與**失敗過的方法**、test oracle 放在 AI 寫不到的地方、每個單位 commit。早上看 git log 和 CHANGELOG，不看最終檔案。
12. `/advisor` 讓 Claude 在關鍵時刻諮詢更強的模型，同 session 自動呼叫，不用另開。

## 擴充

13. **Skill**：完成一次複雜任務後說「把剛才的操作存成 skill」。細節放 `reference/`，要執行的放 `scripts/`，SKILL.md 本身保持短。寫 description 的只有在需要 AI 自己觸發時才寫，否則用 `disable-model-invocation: true` 省 context。
14. **Hook**：事件驅動的 shell / script，用來擋、放行、記錄。格式化、commit 前跑測試都放這裡。
15. **Mod**：在 Claude Code 程序內跑的 JS/TS，能改 tool call、畫 UI、保有狀態。比 hook 強，但沒有沙箱、用你的權限跑，裝之前先 `claude plugin validate`。
16. **MCP**：接外部系統（issue tracker、Notion、telegram 回報）。
17. **Plugin**：以上全部打包的安裝單位。Customize 頁裝的綁帳號、會同步到 Claude Code；命令列裝的只在那台機器。chat 端只載入 skills，hooks / agents / mods 只在 Claude Code 與 Cowork 生效。
18. Skill 有沒有效可以測：`claude plugin eval` 做 no-skill baseline 對照。沒測過的 skill 不知道是加分還是扣分。

## 參考資料

- [Thariq Shihipar on Latent Space（Claude Code 團隊，2026-09）](https://www.latent.space/p/thariq) — CLAUDE.md 越少越好、mods、implementation notes
- [Long-running Claude](https://www.anthropic.com/research/long-running-Claude) — CLAUDE.md + CHANGELOG + oracle + git 四件套
- [mattpocock/skills](https://github.com/mattpocock/skills) — grilling、retro、review 讀 standard 的分工
- [Claude Code docs: Mods](https://code.claude.com/docs/en/plugins/mods/overview)
- [Plugin feature support across platforms](https://claude.com/docs/plugins/platform-support)
- [那些 Agent 神文没告诉你的事](https://youtu.be/b_9D7T0n4RA)
- [Claude Code's Creator Does This Before Every Single Project](https://youtu.be/B-UXpneKw6M)
