# Known Issues

## Format
```
### ISSUE-XXX: [Title]
- **Severity:** Critical | High | Medium | Low
- **Component:** [Which module]
- **Discovered:** YYYY-MM-DD
- **Status:** Open | Workaround Available | Fixed in vX.Y.Z
- **Description:** [What happens]
- **Workaround:** [How to avoid/mitigate]
- **Root Cause:** [If known]
```

---

### ISSUE-001: LLM Context Window Overflow on Large Projects

- **Severity:** High
- **Component:** Orchestrator / Context Assembly
- **Discovered:** 2025-02-01
- **Status:** Workaround Available
- **Description:** Projects with >50 files can exceed context window when assembling agent context, causing truncation of important information.
- **Workaround:** Use loading formula strictly — never load more than 5 files per step. Summarize large files before including.
- **Root Cause:** No automatic summarization of large file contents before context assembly.

---

### ISSUE-002: Race Condition in Concurrent Step Completion

- **Severity:** Medium
- **Component:** Orchestrator / State Machine
- **Discovered:** 2025-02-10
- **Status:** Open
- **Description:** When two parallel steps complete simultaneously, the fan-in detection may miss one completion event, leaving the downstream step in `pending` state.
- **Workaround:** Implement polling check every 30s for steps that have been `pending` longer than expected.
- **Root Cause:** Event-driven completion notification without idempotent state reconciliation.

---

### ISSUE-003: Git Merge Conflicts on Concurrent Repo Updates

- **Severity:** Medium
- **Component:** Prompt Factory / Repository Management
- **Discovered:** 2025-02-15
- **Status:** Workaround Available
- **Description:** When multiple executions update the same repository file simultaneously, Git merge conflicts occur.
- **Workaround:** Serialize writes to the same file using a file-level lock. Queue concurrent updates.
- **Root Cause:** No file-level locking in the repository write path.

---

*No more known issues at this time. Add new issues as they are discovered.*
