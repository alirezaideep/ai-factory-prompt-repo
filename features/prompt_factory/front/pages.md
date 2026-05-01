# Prompt Factory — Frontend Pages

## Page: Repository Browser (`/repos/:repoId`)

### Layout
File explorer on left, file content viewer on right (IDE-like layout).

### Components
- **FileTree** — Collapsible directory tree with file type icons
- **FileViewer** — Markdown renderer with syntax highlighting for code blocks
- **VersionSelector** — Dropdown to switch between versions
- **DiffViewer** — Side-by-side diff when comparing versions
- **EditPanel** — Inline editor for manual file edits

### Data Flow
```
trpc.repos.getTree.useQuery({ repoId })
trpc.repos.getFile.useQuery({ repoId, path })
trpc.repos.updateFile.useMutation()
trpc.repos.applyDelta.useMutation()
```

---

## Page: Repository Generator (`/repos/new`)

### Layout
Wizard-style multi-step form for creating new repositories.

### Steps
1. **Product Definition** — Paste or write product description
2. **Configuration** — Select template, modules to include, naming
3. **Preview** — AI generates structure, user reviews before confirming
4. **Generation** — Progress indicator as files are created
5. **Complete** — Link to browse the new repository

---

## Page: Version History (`/repos/:repoId/versions`)

### Layout
Timeline view of all versions with expandable changelog entries.

### Features
- Visual timeline with version badges
- Expandable changelog for each version
- Diff viewer between any two versions
- Rollback button (with confirmation)
- Branch/tag visualization

---

## Page: Context Loader (`/repos/:repoId/context`)

### Purpose
Interactive tool to test and visualize the loading formula.

### Layout
- Left: Task type selector + module selector
- Right: Resulting file list with token counts and reasons
- Bottom: Total token estimate and context window utilization bar

### Features
- Test different task types to see which files load
- Override loading formula for custom contexts
- Save custom loading formulas
- Export context as a single merged document
