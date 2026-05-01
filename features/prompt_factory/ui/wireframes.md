# Prompt Factory — UI Wireframes

## Repository Browser
```
┌─────────────────────────────────────────────────────────────┐
│ [Logo] AI Factory    Tasks  Repos  Approvals  Analytics     │
├─────────────────────────────────────────────────────────────┤
│ Project: Glass Factory  │  Version: v0.3.0 ▼  │  [+ New]   │
├──────────────────┬──────────────────────────────────────────┤
│                  │                                          │
│  📁 shared/      │  # Order Management — Data Model        │
│    backend.md    │                                          │
│    frontend.md   │  ## Tables                               │
│    design.md     │                                          │
│    app.md        │  ### orders                              │
│    personas.md   │  | Column    | Type        | Nullable | │
│    versioning.md │  |-----------|-------------|----------| │
│                  │  | id        | UUID        | NOT NULL | │
│  📁 features/    │  | customer  | UUID (FK)   | NOT NULL | │
│    📁 orders/    │  | status    | enum(...)   | NOT NULL | │
│      brief.md    │  | priority  | enum(...)   | NOT NULL | │
│      📁 back/    │  | total     | DECIMAL     | NOT NULL | │
│        ▶ data... │  | created   | TIMESTAMP   | NOT NULL | │
│        ▶ api_... │                                          │
│      📁 front/   │  ### order_items                         │
│      📁 ui/      │  | Column    | Type        | Nullable | │
│      📁 workflow/│  | ...       | ...         | ...      | │
│                  │                                          │
│  📁 context.../  │  [Edit] [History] [Diff with prev]      │
│  📁 changes/     │                                          │
│                  │                                          │
└──────────────────┴──────────────────────────────────────────┘
```

## Delta Apply Interface
```
┌─────────────────────────────────────────────────────────────┐
│ Apply Delta Update                              [Cancel] [✓]│
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Target: features/orders/back/data_model.md                 │
│  Operation: [Add ▼]                                         │
│                                                             │
│  Anchor (find this text):                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ## Columns                                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Position: (●) After  ( ) Before  ( ) Replace              │
│                                                             │
│  New Content:                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ | priority | enum('low','normal','high') | NOT NULL |│   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Commit Message: feat(orders): add priority field           │
│  Version Bump: [Patch ▼]                                    │
│                                                             │
│  Preview:                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ## Columns                                         │   │
│  │  | id | UUID | NOT NULL |                           │   │
│  │+ | priority | enum('low','normal','high') | NOT NULL│   │
│  │  | status | enum(...) | NOT NULL |                  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
