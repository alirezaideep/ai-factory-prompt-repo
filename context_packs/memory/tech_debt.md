# Technical Debt Register

## Priority Levels
- **P0 (Critical):** Blocks development or causes data loss. Fix immediately.
- **P1 (High):** Significant impact on velocity or reliability. Fix within current sprint.
- **P2 (Medium):** Causes friction but has workaround. Schedule for next sprint.
- **P3 (Low):** Nice to fix, no urgency. Address during refactoring sprints.

---

## Active Debt Items

| ID | Priority | Component | Description | Impact | Estimated Effort |
|----|----------|-----------|-------------|--------|-----------------|
| TD-001 | P2 | Orchestrator | No automatic context summarization — manual loading formula required | Limits scalability to large projects | 3-5 days |
| TD-002 | P2 | Knowledge Base | Embedding index not incrementally updated — full reindex on changes | Slow for large codebases (>10k files) | 2-3 days |
| TD-003 | P3 | Frontend | No offline support — all operations require network | Poor UX on unstable connections | 5-8 days |
| TD-004 | P1 | Orchestrator | No dead letter queue implementation — failed steps just log errors | Lost work, no recovery path | 2-3 days |
| TD-005 | P3 | All | No i18n framework — UI is English-only | Limits adoption in non-English teams | 5-10 days |

---

## Resolved Debt

| ID | Resolved Date | Resolution |
|----|--------------|------------|
| — | — | No resolved items yet |

---

## Adding New Debt

When taking a shortcut or identifying existing debt, add a row to the Active table with:
1. Sequential ID (TD-XXX)
2. Priority based on impact
3. Component affected
4. Clear description of what's wrong
5. Impact on users/developers
6. Rough effort estimate

When resolving debt, move the row to Resolved with the date and brief resolution description.
