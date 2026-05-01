# Design System

## Overview

The AI Software Factory uses a professional, data-dense design system optimized for technical users managing complex workflows. The design prioritizes clarity, information density, and rapid decision-making.

## Design Philosophy

- **Information-first:** Maximize useful data per viewport without clutter
- **Progressive disclosure:** Show summary → expand for detail
- **Status-driven:** Every entity shows its current state via color + icon
- **Keyboard-first:** Power users navigate without mouse
- **Dark mode default:** Reduces eye strain for extended sessions

## Color System

### Semantic Colors
| Token | Light | Dark | Usage |
|-------|-------|------|-------|
| `--background` | `#FFFFFF` | `#0A0A0F` | Page background |
| `--foreground` | `#0F172A` | `#F8FAFC` | Primary text |
| `--card` | `#F8FAFC` | `#141420` | Card surfaces |
| `--muted` | `#F1F5F9` | `#1E1E2E` | Disabled, secondary |
| `--border` | `#E2E8F0` | `#2D2D3F` | Borders, dividers |

### Status Colors
| Status | Color | Hex | Usage |
|--------|-------|-----|-------|
| Success | Green | `#10B981` | Completed, approved |
| Warning | Amber | `#F59E0B` | Pending review, at risk |
| Error | Red | `#EF4444` | Failed, rejected |
| Info | Blue | `#3B82F6` | In progress, informational |
| Neutral | Gray | `#6B7280` | Draft, inactive |

### Agent Colors (unique per agent type)
| Agent | Color | Hex |
|-------|-------|-----|
| Planner | Purple | `#8B5CF6` |
| Prompt Factory | Emerald | `#059669` |
| Code Agent | Blue | `#2563EB` |
| Test Agent | Cyan | `#06B6D4` |
| Review Agent | Orange | `#EA580C` |
| DevOps Agent | Pink | `#DB2777` |

## Typography

| Element | Font | Size | Weight | Line Height |
|---------|------|------|--------|-------------|
| H1 (Page title) | Inter | 28px | 700 | 1.2 |
| H2 (Section) | Inter | 22px | 600 | 1.3 |
| H3 (Card title) | Inter | 18px | 600 | 1.4 |
| Body | Inter | 14px | 400 | 1.6 |
| Small | Inter | 12px | 400 | 1.5 |
| Code | JetBrains Mono | 13px | 400 | 1.5 |
| Monospace data | JetBrains Mono | 12px | 500 | 1.4 |

## Spacing System

Based on 4px grid:
- `xs`: 4px
- `sm`: 8px
- `md`: 16px
- `lg`: 24px
- `xl`: 32px
- `2xl`: 48px
- `3xl`: 64px

## Component Patterns

### Cards
- Rounded corners: 8px (standard), 12px (featured)
- Shadow: `0 1px 3px rgba(0,0,0,0.1)` (light), none (dark, use border)
- Padding: 16px (compact), 24px (standard)
- Always show status indicator (colored left border or badge)

### Tables
- Striped rows in light mode, hover highlight in dark mode
- Sticky headers for scrollable tables
- Inline actions (icon buttons) on row hover
- Sortable columns with visual indicator
- Pagination: 25/50/100 items per page

### Forms
- Label above input (not floating)
- Error messages below field (red, with icon)
- Required fields marked with asterisk
- Submit button disabled until valid
- Auto-save for long forms (debounced)

### Modals & Dialogs
- Centered, max-width 640px
- Overlay: semi-transparent dark
- Close via X button, Escape key, or overlay click
- Confirmation dialogs for destructive actions

### Navigation
- Left sidebar (collapsible): primary navigation
- Top bar: breadcrumbs + search + user menu
- Tabs: for sub-navigation within a page
- Command palette (Cmd+K): quick navigation

## DAG Visualization (ReactFlow)

### Node Types
| Type | Shape | Color | Content |
|------|-------|-------|---------|
| Task | Rounded rect | Agent color | Title + status |
| Decision | Diamond | Amber | Condition text |
| Start/End | Circle | Green/Gray | Label |
| Parallel | Double-border rect | Blue | Group label |

### Edge Types
- Default: Bezier curve, gray
- Active: Animated dash, blue
- Error: Red, thicker
- Conditional: Dashed

## Loading States

- Skeleton screens for initial page load
- Spinner for actions (button loading state)
- Progress bar for multi-step operations
- Streaming text for LLM outputs (typewriter effect)

## Notification System

| Type | Position | Duration | Action |
|------|----------|----------|--------|
| Success | Bottom-right toast | 3s auto-dismiss | None |
| Error | Bottom-right toast | Persistent | Retry/Dismiss |
| Warning | Top banner | Persistent | Acknowledge |
| Info | Bottom-right toast | 5s auto-dismiss | View detail |
| Approval request | Top-right slide-in | Persistent | Review/Dismiss |

## Responsive Behavior

| Breakpoint | Layout Change |
|-----------|--------------|
| < 768px | Sidebar collapses to hamburger, single-column |
| 768-1024px | Sidebar icons only, 2-column where needed |
| 1024-1280px | Full sidebar, standard layout |
| > 1280px | Full sidebar + detail panel side-by-side |

## Iconography

- Primary: Lucide Icons (consistent with shadcn/ui)
- Custom: Agent avatars (SVG illustrations)
- Size: 16px (inline), 20px (button), 24px (navigation), 32px (feature)
