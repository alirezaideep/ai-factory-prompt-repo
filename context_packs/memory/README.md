# Context Pack: Memory

## Purpose
Project memory — decisions made, conventions adopted, known issues, and glossary. This is the institutional knowledge that persists across all sessions and agents.

## Files

| File | Description | Load When |
|------|-------------|-----------|
| `decisions_log.md` | Architecture/design decisions with rationale | Before making new decisions |
| `conventions.md` | Code style, naming, file organization rules | Every coding session |
| `glossary.md` | Domain terms, abbreviations, definitions | When unclear on terminology |
| `known_issues.md` | Known bugs, limitations, workarounds | Before starting related work |
| `tech_debt.md` | Technical debt register with priority | Sprint planning, refactoring |

## Update Rules
1. Any architectural decision → add to `decisions_log.md`
2. Any new convention agreed → add to `conventions.md`
3. Any new domain term → add to `glossary.md`
4. Any known bug/limitation → add to `known_issues.md`
5. Any shortcut taken → add to `tech_debt.md`
