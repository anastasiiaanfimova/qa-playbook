# Review Notes

Заметки в процессе просмотра ресурсов — что смотрели, что отложили, что пропустили и почему.

---

## hesreallyhim/awesome-claude-code — THE_RESOURCES_TABLE.csv
*2026-04-15*

**Сейчас**
- [agnix](https://github.com/agent-sh/agnix) — линтер для CLAUDE.md, агентов, хуков. С 16+ агентами полезно следить за консистентностью frontmatter. Посмотреть как ставится и что проверяет.
- [TDD Guard](https://github.com/nizos/tdd-guard) — хуки, которые блокируют изменения нарушающие TDD в реальном времени. QA-паттерн, хороший пример quality gates через хуки.
- [Trail of Bits Security Skills](https://github.com/trailofbits/skills) — 15+ security skills (CodeQL, Semgrep, fix verification). Пригодится при углублении security-auditor агента или аудите проекта.

**Позже**
- [Claude Code Agents (undeadlist)](https://github.com/undeadlist/claude-code-agents) — workflow для solo разработчика с multi-auditor паттернами и micro-checkpoints. Интересно для понимания оркестрации агентов, не срочно.
- [Dippy](https://github.com/ldayton/Dippy) — авто-апрув безопасных bash-команд через AST. Уменьшает permission fatigue. Вернуться когда надоест кликать Allow.

**Пропустили**
- Всё командное, визуальная регрессия, мобильное, enterprise-инструменты — не в контексте.

---

## elizabethfuentes12/claude-code-dotfiles
*2026-04-15*

**Взяли**
- Стратегию `.gitignore` с явным allowlist — добавили в claude-configs.

**Пропустили**
- Shell-wrapper для автосинка — не нужен, один компьютер, один источник правды.

---
