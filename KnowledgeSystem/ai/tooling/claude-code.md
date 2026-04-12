# Claude Code 配置與 CLAUDE.md 範本

工具層：隨工具演進而更新。

<!--
📌 預期擴展：
- Superpowers plugin 的配置慣例與自訂 workflow
- Code review plugin 的使用策略
- Sub-agent 的設計模式（何時用、怎麼拆、model 選擇）
- Custom slash commands 的設計原則
- Hooks 的實用配置（pre-commit、post-apply）
- MCP server 的整合模式
-->

---

## CLAUDE.md 分層架構

```
~/.claude/CLAUDE.md                    ← 全域：跨專案的工程標準
project-root/CLAUDE.md                 ← 專案根：domain 規則 + @import
  sub-module/CLAUDE.md                 ← 子模組：模組特有規則
```

規則越泛用越往上放，越 domain-specific 越往下放。
用 `@path/to/file` 引用外部規範，讓 `/init` 不會覆蓋自訂規則。

## CLAUDE.md 通用範本

```markdown
# {Project Name}

## 專案概覽
{一句話描述}

## 工程標準（引用 KnowledgeSystem）
@path/to/KnowledgeSystem/standards/code-quality.md
@path/to/KnowledgeSystem/standards/testing.md

## 語言規範
@path/to/KnowledgeSystem/project-conventions/languages/cpp/conventions.md

## Domain 規則
- {專案特有的業務規則}
- {效能預算}
- {併發模型}

## 禁止事項
- {明確列出不允許的做法}
```

## 配置保護策略

- 自訂規則放在獨立檔案中，用 `@import` 引入 CLAUDE.md
- `/init` 產生的內容與自訂規則以註解標記區隔
- CLAUDE.md 納入 git 版本控制
