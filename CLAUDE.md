# Resource Manager — Project Guide for Claude

## What This Project Is
A **single-file, self-contained Gantt-chart project management web app** (`index.html`, ~168 KB, ~3,642 lines). No build step, no dependencies, no external libraries. Pure vanilla HTML + CSS + JavaScript in one file. Served locally via Python HTTP server on port 8090.

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
| HTML | 451–538 | Layout skeleton (sidebar, timeline, panels, modals, command palette) |
| JavaScript | 540–3,642 | All application logic organized in 13 namespaces |

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

### Namespaces (13)
All logic is organized into namespace objects declared on line 561:
```javascript
const Utils = {}, UI = {}, Portfolio = {}, Projects = {}, Tasks = {}, Deps = {},
      Drag = {}, Sidebar = {}, Timeline = {}, AI = {}, Cmd = {}, CSV = {}, App = {};
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
| `Sidebar` | Sidebar rendering, program CRUD modals, context menus, reorder |
| `Timeline` | Lane assignment, group building, timeline rendering |
| `AI` | AI panel rendering, Gemini/custom LLM integration |
| `Cmd` | Command palette search + navigation |
| `CSV` | CSV export and import with project/group support |
| `App` | Init, render, navigate, zoom, dark mode, sample data, saveState |

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

| Section | Lines | Key Functions |
|---------|-------|---------------|
| Constants | 541–558 | COLORS, MONTHS, ZOOM_CFG, STATUS_OPTIONS |
| Namespaces | 560–561 | All 13 namespace declarations |
| State | 563–586 | `state` object, Drag state vars |
| DOM Refs | 588–604 | `$sidebarBody`, `$timelineWrap`, etc. |
| Utilities | 606–780 | `uid`, `toDate`, `toStr`, `addDays`, `diffDays`, `esc`, `lightenColor`, `findCat`, `findTask`, `Projects.findProject`, project CRUD functions |
| Persistence | 782–1011 | Portfolio load/save/switch/migrate, multi-portfolio management |
| Sample Data | 1013–1067 | `App.loadSampleData` with projects + tasks |
| Grouping | 1069–1161 | `assignLanes` (topological sort + bin-packing), `buildGroups` (by projectId) |
| Rendering | 1163–1464 | `render`, `renderSidebar`, `renderTimeline` |
| Navigation | 1465–1488 | `navigate`, `goToday`, `setZoom` |
| Scroll Sync | 1489–1516 | Timeline↔sidebar scroll sync, sticky labels |
| Sidebar Resize | 1517–1527 | Draggable sidebar width |
| Timeline Interactions | 1528–1586 | Pan, drag-to-create, dblclick, right-click on timeline |
| Category CRUD | 1587–1655 | `toggleCat`, `showAddCatModal`, `doAddCat`, `showEditCatModal`, `doEditCat`, `deleteCat` |
| Task CRUD | 1656–1946 | `showAddTaskModal`, `doAddTask`, `showEditTaskModal`, `doEditTask`, `deleteTask`, link management, modal body builder |
| Detail Panel | 1947–2117 | `openPanel`, `closePanel`, `renderPanel`, `panelUpdate`, URL/dep management |
| Modals | 2118–2145 | `openModal`, `closeModal`, `showConfirmModal` |
| Portfolio UI | 2146–2303 | Project name editing, org name, portfolio dropdown, portfolio operations |
| Add Menu | 2304–2316 | Topbar "+" add menu |
| Context Menus | 2317–2360 | `showCtxMenu`, `showCatCtx`, `showProjectCtx` |
| Sidebar Reorder | 2361–2423 | Grip drag to reorder projects and categories |
| Drag Helpers | 2424–2459 | `showDragLabel`, `calcBarPos`, `pixelToDate`, `timelinePixelX` |
| Bar Drag | 2460–2489 | `onBarMouseDown` (move/resize task bars) |
| Dependencies | 2490–2563 | `enforceDependency`, `cascadeDependencies`, `wouldCreateCycle`, dep bubble drag |
| Mouse Dispatch | 2564–2934 | Unified `mousemove`/`mouseup` handlers for all drag operations |
| Tooltip | 2935–2961 | Bar hover tooltip with project name prefix |
| Keyboard | 2962–3003 | Ctrl+K, Escape, arrow keys, command palette nav |
| Dark Mode | 3004–3029 | Toggle, icon update, init |
| AI Panel | 3030–3213 | AI panel rendering, Gemini/custom API calls, settings |
| CSV Export | 3214–3267 | Export with Program, Project, Project Color, Type columns |
| CSV Import | 3268–3439 | Import with backward-compatible Group/Project column, auto-creates project entities |
| Command Palette | 3440–3615 | Search portfolios, programs, projects, tasks; keyboard navigation |
| Init | 3616–3642 | `App.init`, Enter key handler for modals |

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
| `App.saveState()` | Persist to localStorage |
| `Portfolio.loadState()` | Load from localStorage, handle migration |
| `Portfolio.loadPortfolio(id)` | Load a specific portfolio |
| `Portfolio.switchPortfolio(id)` | Switch active portfolio |
| `Portfolio.savePortfolioRegistry()` | Update `rm_portfolios` |

---

## UI Components

### Layout
```
[Topbar: org name | portfolio title + dropdown | nav/zoom | action buttons]
[Sidebar (resizable)] | [Timeline]
                         [Header: date rows]
                         [Body: grid + task bars + milestones + dep arrows]
```

### Panels & Overlays
- **Detail Panel** (`.detail-panel`) — slides in from right, fixed width 440px, task editing
- **AI Panel** (`.ai-panel`) — slides in from right, fixed width 380px
- **Modal** (`.modal-bg` + `.modal`) — centered overlay for add/edit/confirm dialogs
- **Context Menu** (`.ctx-menu`) — right-click menus, fixed position
- **Command Palette** (`.cmd-palette`) — `Ctrl+K`, searches projects, programs, tasks, portfolios
- **Tooltip** (`.tooltip`) — hover on task bars, shows "Project > Task | Status | Dates"

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
Org name → Portfolio title/dropdown → ◄ Today ► → Zoom select → + Add → Export CSV → Import CSV → AI → Search → ? Help → Dark mode

---

## Features Summary

### Project & Task Creation
- **Projects**: "+ Add Project" link in sidebar, "+" button in topbar add menu, or right-click program header
- **Tasks**: "+" button on a project row, drag on timeline row, double-click timeline row, or right-click timeline row
- Projects have: name, notes, color (NO dates, NO status)
- Tasks have: name, type (task/milestone), dates, status, notes, links, dependencies
- When adding a task without a project context, a new project is auto-created with the task name

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

### Multi-Tab Safety
- Detects changes in other browser tabs via `storage` event
- Alerts user to external modifications before overwriting

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

---

## Known Issues / Pending Work
1. **#20 — Virtualize timeline DOM**: Large portfolios create many DOM elements; needs virtual rendering
2. **#21 — Patch-based undo**: Undo/redo system is currently missing; needs reimplementation as patch-based system (not snapshot-based)
3. **#22 — Visual export PNG/PDF**: Add ability to export timeline as image or PDF

---

## Git Info
- Repository: `https://github.com/horsy1117/resource-manager`
- Branch: `main`
- 13 namespaces: Utils, UI, Portfolio, Projects, Tasks, Deps, Drag, Sidebar, Timeline, AI, Cmd, CSV, App
- Recent features: multi-portfolio, org name stacking, timeline drag/click/right-click task creation, milestone task type, Projects as first-class entities (decoupled from Tasks)
