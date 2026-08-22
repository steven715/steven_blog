# KnowledgeSystem 調整清單

第一版（v0.1.0）尚未在任何專案實測前的體檢結果，共 17 個調整點，分四個面向。
每條標明現況證據、問題、調整方向與依據來源。優先序 P1 代表「不改則後續投入會被放大浪費」。

建立日期：2026-08-16

## 現況

| 項目 | 數值 |
|------|------|
| 規模 | 20 檔 / 963 行 / 1 支 skill |
| 調整點 | 17 項（P1 7 · P2 9 · P3 1） |
| 最大結構問題 | principles 9 條中 7 條有下層副本 |
| 最大機制問題 | 規則無執行機制標註 |

## 執行機制三層（建議補上的第二軸）

現有分層軸是「變更頻率」（principles 幾乎不變 / standards 一年數次 / conventions 每專案不同）。
變更頻率決定維護成本，不決定效力。缺的是這條效力軸：

| 規則性質 | 落地機制 | 保證強度 | 適用範例 |
|----------|----------|----------|----------|
| 可機械判定 | hook / script / CI | 真保證 | build 成功、`main.cpp` ≤5 行、檔案存在、`@import` 為本地路徑、commit 格式 |
| 需判斷 | skill 提供 the tell | 高機率 | 語義斷言、單一職責、錯誤是否真的被處理 |
| 真取捨 | 人類 blocking gate | 靠流程 | 架構選型、鎖策略、錯誤處理慣例、API 契約 |

## A. 分層結構

> 層級定義本身是對的，檔案沒有實現那個定義。

| 優先 | 面向 | 現況（證據） | 問題 | 調整方向 | 依據 |
|------|------|--------------|------|----------|------|
| P1 | principles 與下層重複 | 9 條中 7 條在 `standards/`、`design-principles.md` 有逐字或同義副本（明顯失敗、知識存在於系統、過期文件更危險為逐字） | 當上層每句在下層都有更可執行的版本，讀者與 AI 永遠用下層那份，上層必然失效 | 刪掉 7 條有副本的，只留下層沒有的 | 內部 |
| P1 | principles 是陳述句不是裁決句 | 9 條中僅 3 條具「當 X 與 Y 衝突選 X」形態（約束優於指令、漸進式交付、明顯失敗優於隱性錯誤） | principles 的用途是上訴，不是描述。不能用來裁決的原則在體系中是裝飾 | 全部改寫為取捨句，總數收在 3–5 條 | 內部 |
| P1 | standards 只寫了「合適」半邊 | `code-quality.md`、`testing.md` 全是正面陳述；反例只存在於 `SKILL.md` 的 5 條 Red Flags | 自己定義 standards 要揭露何謂合適與不合適，目前缺後者，實作者無從辨認 | 每條補「the tell」——可判定的識別特徵，例如「重構後測試壞了但行為沒變」 | Pocock |
| P2 | 規則缺來源與反例欄位 | `principles.md` 開頭自稱「經典著作交叉驗證」，無任何一條標來源；註解自陳待補 | 無法區分已驗證與待驗證的規則，也無法判斷一條規則值不值得留 | 加兩欄：來源（哪本書／哪次踩坑）、反例（違反時實際發生過什麼） | 內部 / Tessl |
| P2 | 分層軸只有變更頻率 | README 分層表以「幾乎不變／一年數次／每專案不同」切分 | 變更頻率決定維護成本，不決定效力。缺的是效力軸 | 補第二軸「執行機制」（見上表），與現有頻率軸交叉 | Anthropic |

## B. 施行機制

> skill 是 prompt，prompt 沒有保證能力——這是全表最硬的約束。

| 優先 | 面向 | 現況（證據） | 問題 | 調整方向 | 依據 |
|------|------|--------------|------|----------|------|
| P1 | 禁止事項放在 CLAUDE.md | `SKILL.md` 產出的 CLAUDE.md 範本含 5 條禁令（main.cpp 含業務邏輯、測試只驗證沒 throw 等） | 官方明言絕對禁令不該放 CLAUDE.md；長 session、模糊情境、prompt injection 下規則會失效 | 可機械判定者改寫成 PreToolUse hook；其餘降級為 the tell | Anthropic / Pocock |
| P1 | 可機械檢查的驗證靠 AI 自己念 | Verification 6 條中至少 3 條完全可機械化：`main.cpp` 行數、`SNAPSHOT.md` 存在、每條 `@import` 為本地相對路徑 | 把可確定的事情交給不確定的機制執行，等於自願放棄唯一能拿到的保證 | 寫成 script，掛 CI 或 hook，skill 只負責呼叫並讀結果 | Anthropic |
| P2 | 決策權是參考表不是關卡 | `collaboration-protocol.md` 是一張 8 列決策權矩陣，內容比 Pocock 完整 | 同樣的知識，「參考資料」與「流程中的 blocking gate」效力差一個量級 | 照 Pocock 的 `tdd` 寫法：未確認就不准前進，寫進 skill 步驟而非附錄 | Pocock |
| P2 | plugin 標準不會自動載入 | Claude 只自動發現 `skills/`。標準進 context 的唯一路徑是 `start-cpp-project` 的 vendoring | 沒跑那支綠地 skill 的使用者，安裝後 principles 一行都不會生效 | 補一支 `adopt-standards`：對既有專案做 vendoring 並改寫 CLAUDE.md（推出時處理） | Anthropic |

## C. 可驗證性

> 反思迭代的直覺是對的，但迴圈必須掛外部訊號。

| 優先 | 面向 | 現況（證據） | 問題 | 調整方向 | 依據 |
|------|------|--------------|------|----------|------|
| P1 | 「讓 AI 自我反思升級」不可行 | README 演進原則隱含「用久了自然升級」的路徑，無外部驗證環節 | self-generated skills 平均 −1.3pp（GPT-5.2 −5.6pp）：模型無法可靠地寫出自己受益於閱讀的程序性知識 | 反思迴圈必須掛外部 verifier／eval，不能是 AI 自省 | SkillsBench |
| P1 | 演進原則無法被檢查 | README 寫「每條規範在至少一個實際專案中驗證過才寫入」，但無任何條目標記在哪驗證 | 這條自我約束目前等同自我判斷，是「可驗證」的直接缺口 | 選一個 bounded 任務跑 no-skill baseline 對照，把結果寫回規則旁 | OpenHands |
| P2 | 沒有 eval 迴圈 | 目前無任何量測；體系全靠結構推理成立 | 無法知道哪條規則有效、哪條在拖累。實測案例中 skill 可讓通過率 0%→100%，也可能是負值 | 三件套：bounded deterministic 任務 + pass/fail verifier、no-skill baseline、runtime／token／tool-call 次數 | OpenHands / Tessl |
| P2 | 規則數遠大於驗證數 | 963 行規則對 6 條 verification | Skill Coverage（test suite 對 skill 規則的 exercise 程度）未被測量，比值本身就是缺的指標 | 追蹤「被驗證規則／總規則」，把未覆蓋的規則列為待刪或待驗 | SkillCoverage |

## D. 規模與定位

> 「更全面」在實證上是負分，這與體系直覺相反。

| 優先 | 面向 | 現況（證據） | 問題 | 調整方向 | 依據 |
|------|------|--------------|------|----------|------|
| P1 | 體系偏向全面 | 20 檔 963 行，四層加 templates 與 ai 兩支 | comprehensive skills 比 focused 差 −2.9pp；最佳配置是 2–3 支 skill 而非大集合 | 執行 README 自己寫的「精煉優先於覆蓋」——目前尚未執行 | SkillsBench |
| P2 | skill 流程過重 | `start-cpp-project`：9 步 + 每步 checkpoint + 6 條 verification | 效率退化中 Excessive Procedure 佔 62.6%，其中「過度驗證」67 例為最大宗 | 減步驟或合併 checkpoint，並實測 token／runtime 對照 | Harmful |
| P2 | 看似相關的 skill 才是主凶 | 體系假設「規則越貼近領域越安全」 | 307 個失敗中，功能性失敗 68.8% 來自 Task-Implementation Fault——不是不相關的 skill，是看似相關的 skill 讓 agent 填錯或漏填必要元素 | 在 review checklist 加一項：這條規則有沒有可能誘導出錯誤或缺漏的實作 | Harmful |
| P3 | 軟體工程是增益最低的領域 | KS 專攻軟體工程與 C++ | curated skills 在軟體工程僅 +4.5pp，healthcare 達 +51.9pp（總平均 +16.2pp） | 不是放棄，是認清差異化來源：hook、verifier、人類 gate，不是更多 markdown | SkillsBench |

## 執行順序建議

1. **在每條規則旁標註執行機制**（hook / tell / gate）。成本低，且會揭露有多少條規則根本沒有落地路徑——那份清單就是後續刪減的客觀依據。
2. **收斂 principles 到 3–5 條取捨句**，刪掉下層有副本的。
3. **standards 每條補 the tell**（不合適的識別特徵）。
4. **可機械化的 verification 改寫成 script**。
5. **選一個 bounded C++ 任務跑 no-skill baseline 對照**，記錄通過與否、runtime、token。這是把「驗證過才寫入」從自我判斷變成可檢查的唯一辦法。

前四項在動第一個專案前改比較便宜；C 面向的 eval 相關與 D 面向的流程過重，本來就需要真實專案才驗得了。

## 依據來源

| 代號 | 來源 |
|------|------|
| 內部 | 對 KS 全部 963 行做的結構分析，非外部引用 |
| Anthropic | [Steering Claude Code](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more) 官方指引（CLAUDE.md／skills／hooks 分工） |
| Pocock | [mattpocock/skills](https://github.com/mattpocock/skills) 實際 repo 內容，特別是 `tdd` 與 `git-guardrails` |
| SkillsBench | [SkillsBench](https://huggingface.co/papers/2602.12670)：84 tasks／11 domains／7 configs／7,308 trajectories |
| Harmful | [Agent Skills Can Be Harmful](https://arxiv.org/html/2608.11888v1)：307 個確認的 skill-induced failure |
| OpenHands | [How to Evaluate Agent Skills](https://www.openhands.dev/blog/evaluating-agent-skills)：評測方法論與三組實測對照 |
| Tessl | [Skills for agents, by an agent](https://tessl.io/blog/skills-for-agents-by-an-agent/)：雙迴圈優化實例（79%→99% 等四支 skill） |
| SkillCoverage | [Skill Coverage](https://arxiv.org/pdf/2606.20659)：測試充分性指標 |
