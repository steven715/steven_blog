---
name: start-cpp-project
description: 為全新的 C++ 專案建立標準骨架與專案憲法 CLAUDE.md。當使用者要開新的 C++ 專案、初始化 C++ repo、建立專案結構、bootstrap 一個 C++ 工程、或需要產生一份引用 KnowledgeSystem 工程規範的 CLAUDE.md 時使用。依 KnowledgeSystem 的啟動清單產出:目錄結構、三-target CMake、thin-main 入口、首個測試、文件骨架,以及一份 @import 工程標準、宣告決策權與優先序的 CLAUDE.md。這是 /init 的綠地對應版——/init 從既有代碼逆推 CLAUDE.md,本 skill 在零代碼時正向生成專案與 CLAUDE.md。
---

# Start C++ Project

把一個空目錄,變成一個符合 KnowledgeSystem 規範、且 AI 與人類都能立刻接手的 C++ 專案。
本 skill 的最終產物是一份**專案憲法 CLAUDE.md**:它 `@import` 你的工程標準,讓任何通用 AI 工具(Superpowers、agent-skills、Claude Code 內建 review)都對著「你的標準」執行,而非它們的預設。

## When to Use

- 開一個全新的 C++ 專案 / repo,目錄是空的或只有 README。
- 需要一份引用 KnowledgeSystem、而非 `/init` 自動掃描產生的 CLAUDE.md。
- 想讓新專案一開始就具備可測試、可接手、可被 AI 工具正確約束的結構。

**不適用**:既有大型 codebase 想補 CLAUDE.md → 改用 `/init`(它會掃描現有代碼)。

## 前置:定位 KnowledgeSystem(plugin 隨附)

本 skill 是 `knowledge-system` plugin 的一部分,規範檔與本 SKILL.md 一起被打包安裝。
規範檔位於 plugin 根目錄(即本檔 `skills/start-cpp-project/SKILL.md` 的上兩層),記為 `{KS}`:

- `{KS}/principles.md`、`{KS}/standards/`、`{KS}/project-conventions/`、`{KS}/templates/`、`{KS}/ai/`

執行前先確認能讀到 `{KS}/principles.md`。若讀不到(例如以非 plugin 方式執行),向使用者詢問 KnowledgeSystem 根路徑。

**關鍵原則:不要讓專案 `@import` 指向 plugin 內部。** plugin 安裝後被複製到每台機器各異的快取路徑(`~/.claude/plugins/cache/...`),且 `@import` 無法用 `${CLAUDE_PLUGIN_ROOT}`。因此步驟 9 採**就地落地(vendoring)**:把需要的規範**複製一份快照**進新專案,CLAUDE.md 只 `@import` 專案內的本地相對路徑。這同時帶來「每專案釘住一版標準」的好處——日後你演進 KnowledgeSystem,舊專案不會被動破壞。

## 決策權:先問人類,再動手

本 skill 嚴格遵循 KnowledgeSystem 的人機決策權矩陣(`{KS}/ai/collaboration-protocol.md`)。
以下決策**必須先向使用者確認**,不可由 AI 擅自決定;在得到答案前不要產生對應檔案:

- **專案名稱與一句話定義**(System Narrative 的根)。
- **架構選型 / 系統邊界**:這專案的核心職責、上下游、有無背景線程。
- **併發模型**(若涉及):是否有效能關鍵路徑、鎖策略傾向。AI 在併發上容易產出不一致方案(見 `{KS}/ai/quality-gates.md`)。
- **錯誤處理慣例**:`Expected<T,E>` vs error code vs exception。

把這些答案當作後續所有產出的輸入。AI 負責「在這些約束內生成骨架」,人類負責「定義約束」。

## Process

逐步執行。每步完成後做該步的 checkpoint,未過不前進(漸進式交付,見 `{KS}/principles.md`)。

### 1. 目錄結構
依 `{KS}/project-conventions/structure.md` 建立:
```
src/  tests/unit/  tests/integration/  docs/design/  docs/adr/  docs/diagrams/  scripts/
```
**Checkpoint**:目錄存在,`tests/unit/` 結構鏡射 `src/`。

### 2. 三-Target 構建
依 `{KS}/project-conventions/languages/cpp/build.md` 寫頂層 `CMakeLists.txt`,定義 `{name}_core`(library,除 main 外全部原始碼)+ `{name}`(executable,僅 main.cpp)+ `tests`。
**Checkpoint**:`cmake` configure 成功,三個 target 都出現。

### 3. Thin-Main 入口
建 `src/main.cpp`(≤5 行,僅啟動)與 `src/app/application.h/.cpp`(組裝層:初始化、依賴注入、生命週期)。main 不含任何業務邏輯。
**Checkpoint**:`main.cpp` ≤5 行;business 物件由 `app/` 組裝,彼此不直接 new 對方。

### 4. System Narrative(人類決策的產物)
用 `{KS}/templates/system-narrative-template.md` 在 `docs/design/` 起草,填入前面跟使用者確認的系統定義、物件分類、線程模型。
**Checkpoint**:讀者能在 5 分鐘內掌握系統全貌;物件分類符合 業務/資源/資料 三類。

### 5. Design Doc(至少 Context + Goals/Non-goals)
用 `{KS}/templates/design-doc-template.md` 起草核心子系統設計,明確 Non-goals。
**Checkpoint**:有可量化的 Goals 與明確排除的 Non-goals。

### 6. 首個測試
依 `{KS}/standards/testing.md` 為第一個核心模組寫 `{原始檔名}_test.cpp`,確認測試框架可獨立運行。測試須含**語義斷言**,不可只驗證「沒 throw」。
**Checkpoint**:測試可被 `ctest` 獨立執行並通過,且至少一個語義斷言。

### 7. README
依 `{KS}/standards/documentation.md`:一句話描述 + 構建指令 + 文件索引。
**Checkpoint**:陌生工程師照 README 能成功 build。

### 8. 版本控制
寫 `.gitignore`(排除 `build/`、IDE 配置),首次 commit(conventional commits 格式)。
**Checkpoint**:`build/` 不入庫;首個 commit 完成。

### 9. 落地標準快照 + CLAUDE.md(本 skill 的核心產物)

先 **vendoring**:把下列 `{KS}` 規範複製進專案 `docs/engineering/`,保留相對結構:
```
docs/engineering/principles.md
docs/engineering/standards/{code-quality,testing,documentation}.md
docs/engineering/project-conventions/languages/cpp/{conventions,build,concurrency}.md
docs/engineering/ai/collaboration-protocol.md
```
在 `docs/engineering/` 放一個 `SNAPSHOT.md`,記下來源 plugin 版本(`knowledge-system` 的 `version`)與複製日期,標明「這是釘住的快照,如需升級重跑本 skill 或手動同步」。

再產出下方 CLAUDE.md:它 `@import` **專案內**的 `docs/engineering/...`(本地相對路徑,跨機器不壞),宣告**衝突時的優先序**與**決策權**,讓外部 AI 工具對著你的標準跑。
**Checkpoint**:`docs/engineering/` 下每個被 `@import` 的檔案都實際存在;`SNAPSHOT.md` 記錄了版本與日期;優先序與禁止事項明確;domain 規則已填入步驟 4 的決策。

## CLAUDE.md 產出範本

> 用步驟 1–8 的實際結果填空。`@import` 一律指向專案內 `docs/engineering/`(步驟 9 vendoring 落地的本地副本)。

```markdown
# {Project Name}

## 專案概覽
{一句話描述 —— 取自 System Narrative}

## 規範優先序(衝突時的裁決)
本檔與 docs/engineering/ 下的工程標準為最高優先。
當外部工具(Superpowers / agent-skills / 內建 review)的通用建議與此處牴觸時,以本檔為準。
標準快照來源見 docs/engineering/SNAPSHOT.md。

## 工程標準(專案內快照)
@docs/engineering/principles.md
@docs/engineering/standards/code-quality.md
@docs/engineering/standards/testing.md
@docs/engineering/standards/documentation.md

## C++ 規範
@docs/engineering/project-conventions/languages/cpp/conventions.md
@docs/engineering/project-conventions/languages/cpp/build.md
@docs/engineering/project-conventions/languages/cpp/concurrency.md

## 人機決策權
@docs/engineering/ai/collaboration-protocol.md
架構選型、API 契約、鎖策略由人類決定;AI 在約束內實作。
涉及這些範疇的變更,先提方案待確認,不可直接落地。

## Domain 規則(本專案特有)
- {核心業務規則 —— 取自步驟 4/5}
- 效能預算:{若有,如 hot path p99 延遲}
- 併發模型:{若有,取自前置決策}
- 錯誤處理慣例:{Expected<T,E> / error code / exception 擇一}

## 禁止事項
- main.cpp 含業務邏輯
- 共享狀態未在宣告處標明鎖策略
- 測試只驗證「沒 throw」而無語義斷言
- 模組間直接 new / 持有對方實例(須經介面 / callback / EventBus)
- {其他專案特有禁止項}
```

## Red Flags(出現以下藉口時,停下並糾正)

KnowledgeSystem 觀察到 AI 常以「形式正確但語義空洞」的方式跳步(`{KS}/ai/quality-gates.md`)。以下是本 skill 常見的偷懶與反駁:

- 「先把代碼寫一寫,CLAUDE.md 之後再補」→ 否。CLAUDE.md 是約束的載體,缺它則後續所有產出失去裁決依據。它是步驟 9,但其輸入(決策權那段)在動手前就要確認。
- 「測試之後一起補」→ 否。未經測試的代碼是未完成的代碼;步驟 6 必須在宣稱完成前通過。
- 「架構我先猜一個」→ 否。架構選型與鎖策略屬人類決策,先問。
- 「README/文件不影響功能,跳過」→ 否。文件是系統的一部分;產品化標準要求陌生工程師能接手。
- 「`@import` 直接指向 plugin 安裝目錄比較省事,不用複製」→ 否。plugin 被複製到各機器各異的快取路徑,換機器即壞。必須 vendoring 到專案內再用本地相對路徑 `@docs/engineering/...`。

## Verification(沒有以下證據,不得宣稱完成)

- `cmake` configure + build 成功的輸出。
- `ctest` 跑過首個測試的輸出(看到 PASS,且該測試含語義斷言)。
- `main.cpp` 行數 ≤5 的實際內容。
- `docs/engineering/` 下被 `@import` 的每個檔案都實際存在,且 `SNAPSHOT.md` 記錄了來源版本與日期。
- CLAUDE.md 的每條 `@import` 都是 `@docs/engineering/...` 本地相對路徑,無指向 plugin 快取的路徑。
- `git log` 顯示首個 commit。

## 與其他工具的關係

- **`/init`**:既有專案用 `/init` 逆推;綠地用本 skill 正向生成。兩者不重疊。
- **Superpowers / agent-skills**:本 skill 只負責「立專案 + 立憲法」。後續的 brainstorm / TDD / review workflow 交給那些通用引擎——它們會自動讀取本 skill 產出的 CLAUDE.md,於是對著你的標準執行。
