# Resource Manager — Project Guide for Claude

## What This Project Is
A **single-file, self-contained Gantt-chart project management web app** (`index.html`, ~160 KB, ~3,600 lines). No build step, no dependencies, no external libraries. Pure vanilla HTML + CSS + JavaScript in one file. Served locally via Python HTTP server on port 8090.

## Dev Server
Defined in `.claude/launch.json`:
- Command: `python -m http.server 8090`
- Open: `http://localhost:8090`
- Use `preview_start` with name `"resource-manager"` to launch it.

## File Structure
There is only **one source file**: `index.html`. Everything — styles, markup, and all application logic — lives inside it.

| Section | Lines | Content |
|---------|-------|---------|
| CSS | 1–450 | All styling, dark mode variants, component styles |
| HTML | 451–527 | Layout skeleton (sidebar, timeline, panels, modals) |
| JavaScript | 528–3,606 | All application logic |

**Do not create additional files** unless absolutely necessary. All changes go into `index.html`.

---

## Architecture Overview

### Core Hierarchy
- **Portfolios** — top-level independent workspaces (each stored separately in localStorage)
- **Programs** (called "categories" in code) — colored groupings within a portfolio
- **Projects** — named groupings of tasks within a program (tasks sharing the same `group` field appear as one sidebar row)
- **Tasks** (called "tasks" in code) — individual work items within a project; have dates, status, notes, links, dependencies

### State Object
```javascript
let state = {
  categories: [],          // Array of Program objects (see below)
  zoom: 'weeks',           // 'days' | 'weeks' | 'months' | 'year'
  viewStart: null,         // Date — left edge of timeline view
  selectedTask: null,      // { catId, taskId } | null
  aiKey: '',               // API key for AI provider
  aiProvider: 'gemini',    // 'gemini' | 'custom'
  aiCustomEndpoint: '',    // Custom LLM base URL
  aiCustomModel: '',       // Custom LLM model name
  aiMessages: [],          // [{ role: 'user'|'assistant'|'system', text }]
  projectName: 'Project Plan', // Active portfolio display name
  activePortfolioId: null, // Active portfolio localStorage key suffix
  orgName: 'Organization'  // Org name shown above portfolio title
};
```

### Program Object
```javascript
{
  id: string,        // uid() generated
  name: string,
  color: hex_string, // from COLORS constant
  collapsed: boolean,
  tasks: Task[]
}
```

### Task Object
```javascript
{
  id: string,
  name: string,
  group: string,           // Visual grouping key (defaults to task name)
  startDate: 'YYYY-MM-DD',
  endDate: 'YYYY-MM-DD',
  status: string,          // See STATUS_OPTIONS
  notes: string,
  urls: [{ title, url }],
  dependencies: string[],  // Array of predecessor task IDs
  lastUpdated: ISO_string
}
```

---

## Constants

### Colors (12 predefined)
```javascript
const COLORS = ['#3b82f6','#ef4444','#10b981','#f59e0b','#8b5cf6','#ec4899',
                '#06b6d4','#f97316','#14b8a6','#6366f1','#e11d48','#84cc16'];
```

### Zoom Configuration
```javascript
const ZOOM_CFG = {
  days:   { dayWidth: 56,  totalDays: 70,  navStep: 7 },
  weeks:  { dayWidth: 26,  totalDays: 126, navStep: 14 },
  months: { dayWidth: 14,  totalDays: 210, navStep: 30 },
  year:   { dayWidth: 3,   totalDays: 730, navStep: 90 }
};
```

### Status Options
```javascript
const STATUS_OPTIONS = [
  { value: 'not_started', label: 'Not Started', color: '#9ca3af', shape: 'hollow' },
  { value: 'in_progress', label: 'In Progress', color: '#3b82f6', shape: 'half' },
  { value: 'planning',    label: 'In Planning', color: '#f59e0b', shape: 'half' },
  { value: 'confirmed',   label: 'Confirmed',   color: '#10b981', shape: 'filled' },
  { value: 'complete',    label: 'Complete',    color: '#6366f1', shape: 'check' }
];
```

---

## localStorage Keys
| Key | Contents |
|-----|----------|
| `rm_portfolios` | JSON array of `{ id, name, createdAt, updatedAt }` |
| `rm_portfolio_[ID]` | JSON `{ categories, projectName, aiMessages }` per portfolio |
| `rm_global` | JSON `{ aiKey, aiProvider, aiCustomEndpoint, aiCustomModel, orgName }` |
| `rm_active` | Active portfolio ID string |
| `rm_darkMode` | `'1'` or `'0'` |
| `rm_state` | Legacy key — auto-migrated to multi-portfolio format on load |

---

## Key Functions

### Rendering
| Function | Purpose |
|----------|---------|
| `render()` | Master render — calls sidebar + timeline |
| `renderSidebar()` | Program/task list in left panel |
| `renderTimeline()` | Header date row + body with task bars + dependency SVG |
| `renderPanel()` | Detail slide-out panel for selected task |
| `renderAIPanel()` | AI assistant interface |

### State Persistence
| Function | Purpose |
|----------|---------|
| `saveState()` | Persist to localStorage + call pushUndo() |
| `loadState()` | Load from localStorage, handle migration |
| `loadPortfolio(id)` | Load a specific portfolio |
| `switchPortfolio(id)` | Switch active portfolio |
| `savePortfolioRegistry()` | Update `rm_portfolios` |

### Undo/Redo
- Snapshot-based (stores JSON of `state.categories`)
- Max 30 entries (`MAX_UNDO = 30`)
- `pushUndo()` — called by `saveState()` before mutations
- `undo()` / `redo()` — restore from stacks, re-render, re-save
- `_skipUndo` flag prevents double-push during undo/redo

### Dependency System
| Function | Purpose |
|----------|---------|
| `enforceDependency(task, predTask)` | Push task start to after predecessor end |
| `cascadeDependencies(movedTaskId, deltaDays)` | Propagate date shifts downstream |
| `wouldCreateCycle(fromId, toId)` | DFS cycle detection |

### Lane Assignment
- `assignLanes(tasks)` — Topological sort + greedy bin-packing for overlapping tasks in the same group
- Tasks in the same `group` that overlap in time get separate lanes

### Utilities
```javascript
uid()                    // Generate unique ID
toDate(s)                // 'YYYY-MM-DD' string → Date object (no timezone shift)
toStr(d)                 // Date → 'YYYY-MM-DD' string
addDays(d, n)            // Date + n days → new Date
diffDays(a, b)           // Date difference in days
clamp(v, min, max)       // Numeric clamp
showToast(msg)           // Show bottom-center toast for 2 seconds
```

---

## UI Components

### Layout
```
[Topbar: org name | portfolio title + dropdown | nav/zoom | action buttons]
[Sidebar (resizable)] | [Timeline]
                         [Header: date rows]
                         [Body: grid + task bars + dep arrows]
```

### Panels & Overlays
- **Detail Panel** (`.detail-panel`) — slides in from right, fixed width 440px, task editing
- **AI Panel** (`.ai-panel`) — slides in from right, fixed width 380px
- **Modal** (`.modal-bg` + `.modal`) — centered overlay for add/edit/confirm dialogs
- **Context Menu** (`.ctx-menu`) — right-click menus, fixed position
- **Command Palette** (`.cmd-palette`) — `Ctrl+K`, searches across all portfolios
- **Tooltip** (`.tooltip`) — hover on task bars

### Topbar Buttons (left to right)
Org name → Portfolio title/dropdown → ◄ Today ► → Zoom select → + Add → Export CSV → Import CSV → AI → Search → ? Help → Dark mode

---

## Features Summary

### Task Creation Methods
1. "+ Add task" button in sidebar
2. Drag horizontally on an empty timeline row
3. Double-click on timeline row
4. Right-click on timeline row

### Keyboard Shortcuts
| Shortcut | Action |
|----------|--------|
| `Ctrl+Z` | Undo |
| `Ctrl+Y` | Redo |
| `Ctrl+K` | Open command palette |
| `?` | Show keyboard shortcut help |
| `Esc` | Close panel/modal/palette |

### AI Assistant
- Providers: **Google Gemini** (free tier) or **Custom OpenAI-compatible endpoint**
- Project context is injected as system prompt
- Conversation history: last 10 messages sent per request
- API keys stored in `rm_global` localStorage key

### CSV Import/Export
- Export: all fields including dependencies (by task name)
- Import modes: Replace (wipe current) or Merge (append)
- Supports column name variants: Program/Category, Project/Task

### Multi-Tab Safety
- Detects changes in other browser tabs via `storage` event
- Alerts user to external modifications before overwriting

---

## Dark Mode
- Toggled by adding/removing class `dark` on `<html>` element
- All dark variants defined as CSS overrides using `html.dark .selector` pattern
- CSS variables (`--bg`, `--text`, `--border`, etc.) control theming

---

## Coding Conventions
- **No build tools** — edit `index.html` directly
- **Minified CSS** — all CSS is on single lines or minimally spaced; maintain this style
- **Inline event handlers** — `onclick="functionName()"` pattern used throughout HTML
- **DOM refs** cached at top of JS as `$elementName` constants
- **`uid()`** for all new IDs — never use sequential integers
- **Dates always as `'YYYY-MM-DD'` strings** in state; use `toDate()`/`toStr()` to convert
- **`saveState()`** must be called after any mutation to `state.categories` or state fields
- **`render()`** must be called after state changes to update the UI
- Typical mutation pattern: `pushUndo()` → mutate state → `saveState()` → `render()`

---

## Git Info
- Branch: `main`
- Recent features added: multi-portfolio, org name stacking, timeline drag/click/right-click task creation, Programs rename (was Categories)
