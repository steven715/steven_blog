# C++ New Project Checklist

新建 C++ 專案時的啟動清單。各項引用 KnowledgeSystem 中的對應規範。

<!--
📌 預期擴展：
- 加入各步驟的預估時間
- 加入「驗收條件」欄位（怎樣算完成）
- 加入 AI 輔助標記（哪些步驟適合委託 AI、哪些必須人類完成）
-->

---

## 啟動順序

1. **目錄結構** — 建立 `src/`, `tests/`, `docs/`, `scripts/`
   → 參考 `project-conventions/structure.md`

2. **構建配置** — 撰寫頂層 `CMakeLists.txt`，定義三個 target（core + exe + tests）
   → 參考 `languages/cpp/build.md`

3. **應用程式入口** — 建立 `src/main.cpp`（thin entry）和 `src/app/application.h/.cpp`
   → 參考 `languages/cpp/build.md`

4. **System Narrative** — 撰寫系統全景敘事
   → 使用 `templates/system-narrative-template.md`

5. **Design Doc** — 撰寫核心子系統的設計文件，至少包含 Context、Goals/Non-goals
   → 使用 `templates/design-doc-template.md`

6. **首個測試** — 為第一個核心模組撰寫 `_test.cpp`，確認測試框架運行正常
   → 參考 `standards/testing.md`

7. **README** — 包含一句話描述、構建指令、文件索引
   → 參考 `standards/documentation.md`

8. **版本控制** — 配置 `.gitignore`（排除 `build/`、IDE 配置檔），首次 commit

9. **CLAUDE.md** — 配置 AI 開發規範，引用 KnowledgeSystem 相關文件
   → 參考 `ai/tooling/claude-code.md`
