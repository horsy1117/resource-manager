# Resource Manager — Project Guide for Claude

## What This Project Is
A **single-file, self-contained Gantt-chart project management web app** (`index.html`, ~175 KB, ~3,817 lines). No build step, no dependencies, no external libraries. Pure vanilla HTML + CSS + JavaScript in one file. Served locally via Python HTTP server on port 8090.

## Dev Server
Defined in `.claude/launch.json`:
- Command: `python -m http.server 8090`
- Open: `http://localhost:8090`
- Use `preview_start` with name `"resource-manager"` to launch it.

## File Structure
There is only **one source file**: `index.html`. Everything — styles, markup, and all application logic — lives inside it.

| Section | Lines | Content |
|---------|-------|---------|
| CSS | 1–450 | All styling, CSS variables, dark mode overrides, component styles |
| HTML | 451–542 | Layout skeleton (sidebar, timeline, panels, modals, command palette, toast) |
| JavaScript | 543–3,815 | All application logic organized in 14 namespaces |

**Do not create additional files** unless absolutely necessary. All changes go into `index.html`.

---

## Architecture Overview

### Core Hierarchy
```
Portfolio (workspace)
  └── Program (colored grouping, called "category" in code)
        ├── Project (first-class entity: name, notes, color — NO dates, NO status)
        │     ├── Task (work item: name, type, dates, status, notes, links, deps)
        │     └── Task (milestone: zero-duration diamond, single date)
        └── Project
              └── Task
```

- **Portfolios** — top-level independent workspaces (each stored separately in localStorage)
- **Programs** (called "categories" in code) — colored groupings within a portfolio
- **Projects** — first-class entities within a program; have name, notes, color; stored in `cat.projects` array
- **Tasks** — individual work items within a project; have dates, status, type, notes, links, dependencies; linked to project via `projectId`

### Namespaces (14)
All logic is organized into namespace objects declared on line ~563:
```javascript
const Utils = {}, UI = {}, Portfolio = {}, Projects = {}, Tasks = {}, Deps = {},
      Drag = {}, Sidebar = {}, Timeline = {}, AI = {}, Cmd = {}, CSV = {}, App = {}, FileStorage = {};
```

| Namespace | Responsibility |
|-----------|---------------|
| `Utils` | ID generation, date math, DOM helpers, color manipulation |
| `UI` | Modal, context menu, confirm dialog, tooltip, color picker |
| `Portfolio` | Multi-portfolio CRUD, localStorage persistence, migration |
| `Projects` | Project CRUD modals (add/edit/delete), findProject helper |
| `Tasks` | Task CRUD modals, detail panel, task-level operations |
| `Deps` | Dependency enforcement, cycle detection, cascading |
| `Drag` | All drag states + handlers (bar move/resize, create, reorder, pan, dep-draw) |
| `Sidebar` | Sidebar rendering, program CRUD modals, context menus, reorder, row focus |
| `Timeline` | Lane assignment, group building, timeline rendering |
| `AI` | AI panel rendering, Gemini/custom LLM integration |
| `Cmd` | Command palette search + navigation |
| `CSV` | CSV export and import with project/group support |
| `App` | Init (async), render, navigate, zoom, dark mode, sample data, saveState |
| `FileStorage` | File System Access API — folder picker, data.json read/write, polling |

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
  id: string,          // uid() generated
  name: string,
  color: hex_string,   // from COLORS constant
  collapsed: boolean,
  projects: Project[], // ordered list of project entities
  tasks: Task[]        // all tasks in this program (linked to projects via projectId)
}
```

### Project Object
```javascript
{
  id: string,        // uid() generated
  name: string,
  notes: string,
  color: hex_string  // defaults to program color; from COLORS constant
}
```

### Task Object
```javascript
{
  id: string,
  name: string,
  type: string,            // 'task' | 'milestone'
  projectId: string,       // References a Project.id within the same program
  startDate: 'YYYY-MM-DD',
  endDate: 'YYYY-MM-DD',   // Same as startDate for milestones
  status: string,          // See STATUS_OPTIONS
  notes: string,
  urls: [{ title, url }],
  dependencies: string[],  // Array of predecessor task IDs (within same program)
  lastUpdated: ISO_string
}
```

---

## JavaScript Section Map

| Section | Lines (approx) | Key Functions |
|---------|-------|---------------|
| Constants | 541–558 | COLORS, MONTHS, ZOOM_CFG, STATUS_OPTIONS |
| Namespaces | 560–563 | All 14 namespace declarations |
| State | 565–588 | `state` object, Drag state vars |
| DOM Refs | 590–606 | `$sidebarBody`, `$timelineWrap`, etc. |
| Utilities | 608–790 | `uid`, `toDate`, `toStr`, `addDays`, `diffDays`, `esc`, `lightenColor`, `findCat`, `findTask`, `Projects.findProject`, project CRUD functions |
| File Storage | 792–940 | `FileStorage` — IDB handle persistence, folder picker, read/write, polling, toast |
| Persistence | 942–1070 | Portfolio load/save/switch/migrate, multi-portfolio management |
| Sample Data | 1072–1126 | `App.loadSampleData` with projects + tasks |
| Grouping | 1128–1220 | `assignLanes` (topological sort + bin-packing), `buildGroups` (by projectId) |
| Rendering | 1222–1530 | `render`, `renderSidebar`, `renderTimeline` |
| Navigation | 1531–1554 | `navigate`, `goToday`, `setZoom` |
| Scroll Sync | 1555–1582 | Timeline↔sidebar scroll sync, sticky labels |
| Sidebar Resize | 1583–1593 | Draggable sidebar width |
| Timeline Interactions | 1594–1745 | Pan, drag-to-create, dblclick, right-click on timeline |
| Category CRUD | 1746–1814 | `toggleCat`, `showAddCatModal`, `doAddCat`, `showEditCatModal`, `doEditCat`, `deleteCat` |
| Task CRUD | 1815–2010 | `showAddTaskModal`, `doAddTask`, `showEditTaskModal`, `doEditTask`, `deleteTask`, link management, modal body builder |
| Detail Panel | 2011–2181 | `openPanel`, `closePanel`, `renderPanel`, `panelUpdate`, URL/dep management |
| Modals | 2182–2210 | `openModal`, `closeModal`, `showConfirmModal` |
| Portfolio UI | 2211–2368 | Project name editing, org name, portfolio dropdown, portfolio operations |
| Add Menu | 2369–2381 | Topbar "+" add menu |
| Context Menus | 2382–2425 | `showCtxMenu`, `showCatCtx`, `showProjectCtx` |
| Sidebar Reorder | 2426–2488 | Grip drag to reorder projects and categories |
| Drag Helpers | 2489–2524 | `showDragLabel`, `calcBarPos`, `pixelToDate`, `timelinePixelX` |
| Bar Drag | 2525–2554 | `onBarMouseDown` (move/resize task bars) |
| Dependencies | 2555–2628 | `enforceDependency`, `cascadeDependencies`, `wouldCreateCycle`, dep bubble drag |
| Mouse Dispatch | 2629–2999 | Unified `mousemove`/`mouseup` handlers for all drag operations |
| Tooltip | 3000–3026 | Bar hover tooltip with project name prefix |
| Keyboard | 3027–3068 | Ctrl+K, Escape, arrow keys, command palette nav |
| Dark Mode | 3069–3094 | Toggle, icon update, init |
| AI Panel | 3095–3278 | AI panel rendering, Gemini/custom API calls, settings |
| CSV Export | 3279–3332 | Export with Program, Project, Project Color, Type columns |
| CSV Import | 3333–3504 | Import with backward-compatible Group/Project column, auto-creates project entities |
| Command Palette | 3505–3680 | Search portfolios, programs, projects, tasks; keyboard navigation |
| Init | 3681–3815 | `App.init` (async), Enter key handler for modals |

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
  { value: 'draft',     label: 'Draft',       color: '#ef4444' },
  { value: 'planning',  label: 'In Planning',  color: '#f59e0b' },
  { value: 'confirmed', label: 'Confirmed',    color: '#10b981' }
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

## Key Systems

### Rendering Pipeline
1. `App.render()` → calls `Timeline.buildGroups()` → caches result
2. `Sidebar.renderSidebar()` — iterates `cat.projects`, renders rows with color dots + task counts
3. `Timeline.renderTimeline()` — renders header, grid, task bars (using `group.project.color`), milestones, dependency SVG

### Lane Assignment
- `Timeline.assignLanes(tasks)` — Topological sort + greedy bin-packing
- Tasks in the same project that overlap in time get separate lanes
- Milestones get extended visual footprint (~120px label width converted to days based on zoom)
- `Timeline.buildGroups()` — groups tasks by `projectId` using `cat.projects` ordering

### Dependency System
| Function | Purpose |
|----------|---------|
| `Deps.enforceDependency(catId, predId, depId)` | Push task start to after predecessor end |
| `Deps.cascadeDependencies(catId, movedTaskId)` | Propagate date shifts downstream |
| `Deps.wouldCreateCycle(catId, fromId, toId)` | DFS cycle detection |
| `Deps.enforceAllDependencies(catId, taskId)` | Entry point after task date changes |

Dependencies are only allowed between tasks in the same project (enforced in dep bubble drop).

### Undo/Redo
- **Currently missing** (lost during namespace restructuring) — planned for re-implementation as patch-based system (#21)

### Data Migration
`Portfolio.migrateTaskData()` runs on every portfolio load and handles:
1. Missing fields: `dependencies`, `urls`, `lastUpdated`, `type`
2. Status rename: `tentative` → `planning`
3. **Group-to-project migration**: If `cat.projects` doesn't exist, creates project entities from unique `task.group` values, assigns `task.projectId`, deletes `task.group`

### State Persistence
| Function | Purpose |
|----------|---------|
| `App.saveState()` | Persist to localStorage + trigger `FileStorage.writeAll()` |
| `Portfolio.loadState()` | Load from localStorage, handle migration |
| `Portfolio.loadPortfolio(id)` | Load a specific portfolio |
| `Portfolio.switchPortfolio(id)` | Switch active portfolio |
| `Portfolio.savePortfolioRegistry()` | Update `rm_portfolios` |

### FileStorage System (File System Access API)
Enables shared `data.json` file persistence for multi-user use (Chrome/Edge only; falls back to localStorage on Firefox).

**data.json structure:**
```json
{
  "portfolios": [...],
  "portfolioData": { "[id]": { "categories": [], "projectName": "", "aiMessages": [] } },
  "global": { "aiKey": "", "aiProvider": "gemini", "orgName": "" },
  "activeId": "...",
  "darkMode": "0",
  "_savedAt": "ISO timestamp"
}
```

**Key functions:**
| Function | Purpose |
|----------|---------|
| `FileStorage.init()` | Restore saved directory handle from IndexedDB; uses `queryPermission` only (no user gesture needed on load) |
| `FileStorage.selectFolder()` | Open folder picker (opens in previously selected folder via `startIn` + `id`); if pending handle exists, calls `reconnect()` instead |
| `FileStorage.reconnect()` | Called via user gesture (banner button or folder icon); calls `requestPermission`, then loads data and re-renders |
| `FileStorage.writeAll()` | Collect all state, stamp `_savedAt`, write to `data.json` |
| `FileStorage.importToLocalStorage()` | Read `data.json`, populate localStorage, record `_lastSavedAt` |
| `FileStorage.startPolling()` | Start 15-second interval poll for external changes |
| `FileStorage.poll()` | Compare `_savedAt` in file vs `_lastSavedAt`; if different, call `_applyData()` |
| `FileStorage._applyData(data)` | Apply external data to state + re-render without page reload |
| `FileStorage.showToast(msg)` | Show bottom-center toast for 3 seconds |

**Permission model:**
- `requestPermission()` requires a user gesture — it **cannot** be called on page load
- `init()` uses `queryPermission()` only; if permission not yet granted, parks handle as `_pendingHandle`
- Amber reconnect banner (`#fsReconnect`) appears below topbar when a pending handle exists
- User clicks **"Load from file"** (or the folder icon) → `reconnect()` → `requestPermission()` → data loads
- This one-click-per-session is a hard browser security requirement

**Folder picker behaviour:**
- `showDirectoryPicker({ id: 'rm-data-folder', startIn: _pendingHandle })` — opens in the previously selected folder every time
- `id` key lets the browser remember location independently of the handle

**Polling behaviour:**
- Polls every **15 seconds**
- Skips poll if a modal is open (avoids disrupting user mid-edit)
- On external change detected: applies data directly to state, calls `App.render()`, shows toast "↻ Updated by another user"
- Own saves set `_lastSavedAt` so they never trigger a self-reload

**Topbar indicator:**
- Folder icon button (`.fs-btn`) with status dot (`.fs-dot`)
- Gray dot = localStorage only; Amber dot = pending permission; Green dot = connected

**`App.init` is now `async`** — awaits `FileStorage.init()` and `FileStorage.importToLocalStorage()` before calling `Portfolio.loadState()`.

---

## UI Components

### Layout
```
[Topbar: org name | portfolio title + dropdown | nav/zoom | action buttons]
[Sidebar (resizable)] | [Timeline]
                         [Header: date rows]
                         [Body: grid + task bars + milestones + dep arrows]
[Toast: bottom-center, fades after 3s]
```

### Panels & Overlays
- **Detail Panel** (`.detail-panel`) — slides in from right, fixed width 440px, task editing
- **AI Panel** (`.ai-panel`) — slides in from right, fixed width 380px
- **Modal** (`.modal-bg` + `.modal`) — centered overlay for add/edit/confirm dialogs
- **Context Menu** (`.ctx-menu`) — right-click menus, fixed position
- **Command Palette** (`.cmd-palette`) — `Ctrl+K`, searches projects, programs, tasks, portfolios
- **Tooltip** (`.tooltip`) — hover on task bars, shows "Project > Task | Status | Dates"
- **Toast** (`.fs-toast` `#fsToast`) — bottom-center notification for file sync events

### Sidebar Project Row Behaviour
- **Click anywhere on row** → focuses row (`Sidebar.focusRow(el)`): adds `.row-focused` to both the `.s-task` sidebar element AND the matching `.t-row[data-project]` timeline element, highlighting the full horizontal band
- **Click project name text** (`.s-task-name-link`) → opens edit modal (`Projects.showEditProjectModal`)
- **Action buttons** (+, pencil, ×) → `stopPropagation`, work independently
- `.s-task.row-focused` CSS: `rgba(59,130,246,.07)` tint + `inset 2px 0 0 #3b82f6` left accent
- `.t-row.task-row.row-focused` CSS: `rgba(59,130,246,.05)` tint only

### Context Menus
| Menu | Trigger | Items |
|------|---------|-------|
| Program | Right-click program header | Add Project, Edit Program, Delete Program |
| Project | Right-click project row in sidebar | Add Task, Edit Project, Delete Project |
| Timeline row | Right-click empty area on timeline | Add Task here |
| Dependency arrow | Right-click dep arrow on timeline | Remove dependency |

### Modals
| Modal | Fields | Called by |
|-------|--------|----------|
| New/Edit Program | Name, Color | Sidebar + context menus |
| New/Edit Project | Name, Notes, Color (NO dates, NO status) | Sidebar + context menus |
| New/Edit Task | Name, Type, Start/End Date, Status, Notes, Links | Sidebar "+" button, timeline interactions |
| Confirm | Message + Cancel/Delete | Delete operations |

### Topbar Buttons (left to right)
Org name → Portfolio title/dropdown → ◄ Today ► → Zoom select → + Add → Export CSV → Import CSV → AI → Search → Folder (FileStorage) → Dark mode

---

## Features Summary

### Project & Task Creation
- **Projects**: "+ Add Project" link in sidebar, "+" button in topbar add menu, or right-click program header
- **Tasks**: "+" button on a project row, drag on timeline row, double-click timeline row, or right-click timeline row
- Projects have: name, notes, color (NO dates, NO status)
- Tasks have: name, type (task/milestone), dates, status, notes, links, dependencies
- When adding a task without a project context, a new project is auto-created with the task name
- **Drag-to-create prefills dates**: `Tasks.showAddTaskModal(catId, projectId, endDate, startDate)` — 4th param is `prefillStart` when 2nd param is a projectId

### `Tasks.showAddTaskModal` Signature
```javascript
Tasks.showAddTaskModal(catId, projectIdOrStart, prefillEnd, prefillStart2)
```
- If `projectIdOrStart` starts with `'id_'` → it's a projectId; `prefillStart2` is the start date
- Otherwise → `projectIdOrStart` is the start date, `prefillEnd` is the end date

### Milestones
- Task type `'milestone'` — zero-duration, rendered as diamond shape on timeline
- Single date (startDate === endDate)
- Move-only drag (no resize)
- Extended visual footprint in lane assignment to prevent label overlap

### Keyboard Shortcuts
| Shortcut | Action |
|----------|--------|
| `Ctrl+K` | Open command palette |
| `Esc` | Close panel/modal/palette |
| `←` / `→` | Navigate timeline |
| `T` | Go to today |

### AI Assistant
- Providers: **Google Gemini** (free tier) or **Custom OpenAI-compatible endpoint**
- Project context is injected as system prompt
- Conversation history: last 10 messages sent per request
- API keys stored in `rm_global` localStorage key

### CSV Import/Export
- **Export columns**: Program, Program Color, Type, Task Name, Project, Project Color, Start Date, End Date, Duration, Status, Notes, Links, Dependencies
- **Import modes**: Replace (wipe current) or Merge (append)
- Supports backward-compatible column names: Program/Category, Project/Group
- Import auto-creates Project entities from the Project/Group column

### Multi-User File Sync
- Folder connected via File System Access API (Chrome/Edge only)
- All users open `index.html` from the same SharePoint/shared folder
- Each save writes `data.json` with a `_savedAt` ISO timestamp
- Every 15 seconds, each client polls `data.json`; if `_savedAt` changed and wasn't caused by self → auto-apply + toast
- Modal-guard: poll skips if a modal is open

---

## Dark Mode
- Toggled by adding/removing class `dark` on `<html>` element
- CSS variables defined in `:root` (light) and `html.dark` (dark) blocks
- Key variables: `--bg`, `--text`, `--border`, `--blue`, `--surface-elevated`, etc.

---

## Coding Conventions
- **No build tools** — edit `index.html` directly
- **Minified CSS** — all CSS is on single lines or minimally spaced; maintain this style
- **Inline event handlers** — `onclick="functionName()"` pattern used throughout HTML
- **Namespace pattern** — all functions assigned as `Namespace.functionName = function() { ... }`
- **DOM refs** cached at top of JS as `$elementName` constants
- **`uid()`** for all new IDs — format: `id_` + random + timestamp; never use sequential integers
- **Dates always as `'YYYY-MM-DD'` strings** in state; use `toDate()`/`toStr()` to convert
- **`App.saveState()`** must be called after any mutation to `state.categories` or state fields
- **`App.render()`** must be called after state changes to update the UI
- Typical mutation pattern: mutate state → `App.render()` → `App.saveState()`
- **`App.init` is async** — do not make it synchronous

---

## Known Issues / Pending Work
1. **#20 — Virtualize timeline DOM**: Large portfolios create many DOM elements; needs virtual rendering
2. **#21 — Patch-based undo**: Undo/redo system is currently missing; needs reimplementation as patch-based system (not snapshot-based)
3. **#22 — Visual export PNG/PDF**: Add ability to export timeline as image or PDF

---

## Git Info
- Repository: `https://github.com/horsy1117/resource-manager`
- Branch: `main`
- 14 namespaces: Utils, UI, Portfolio, Projects, Tasks, Deps, Drag, Sidebar, Timeline, AI, Cmd, CSV, App, FileStorage
- Recent features: FileStorage (data.json + polling), multi-user sync, reconnect banner, folder picker memory, sidebar row focus highlight, drag-to-create date prefill fix
