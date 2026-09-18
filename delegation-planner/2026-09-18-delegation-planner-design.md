# Delegation Planner Design

**Date:** 2026-09-18  
**Status:** Approved (see "Defaults chosen" at the end for items to veto)

## Goal

A desktop app for planning and tracking work delegated across a small technical team (currently three people, must scale to more). Tasks and projects sit on a board organised by person, each person's work grouped by status. Everything is stored as readable YAML so the data can be copied to another machine, exported as a snapshot, and imported back.

## Decisions made during brainstorming

| Area | Decision |
|------|----------|
| Interface | Desktop GUI (Python, tkinter — same stack as sibling projects) |
| Storage | YAML. One file per project plus one shared config file (people, statuses, custom fields) |
| Board layout | Columns = people; inside each column, cards grouped by status |
| Counts | Strip at the top: per person, broken down by status |
| Moving work | Drag-and-drop a card to another person and/or status group |
| Project types | Two: **single-owner** (whole project has one assignee + status; sub-tasks are a checklist that never appears on the board) and **independent** (each sub-task has its own assignee + status and appears on the board as its own card) |
| Standalone tasks | Live in a built-in project called "Individual Tasks" — same file format as any project, cannot be deleted or renamed |
| Project visibility | Rollup strip above the board: one card per project showing status breakdown (independent) or checklist progress (single-owner) plus contributor avatars; sub-tasks still appear individually below |
| Clicking a project | Depends on type: independent → board filters to that project's cards; single-owner → detail panel opens with the checklist |
| Task fields | Built in: priority, due date, description, tags. Plus an in-app custom field builder (name + type) |
| Statuses, people, priorities | All three user-defined in an in-app Settings screen. Nothing hardcoded |
| Card sort order | User-selectable (priority, due date, title, created, updated) and persisted |
| First launch | Setup wizard requires at least one person, one status, and one priority before the board opens |
| Backup | Export = timestamped snapshot; Import = load a snapshot back, replacing current data |
| Platforms | Linux, macOS, and Windows are all first-class — see "Cross-platform" below |

## Architecture

```
delegation-planner/
├── src/
│   ├── main.py              # Entry point: resolve data dir, first-run check, launch App
│   ├── models.py            # Dataclasses: Person, Status, FieldDef, Task, Project, Config
│   ├── store.py             # Data dir layout, YAML load/save, id generation, atomic writes
│   ├── app_prefs.py         # Remembers the chosen data dir (tiny JSON in platform config dir)
│   ├── counts.py            # Pure functions: per-person/per-status counts, project rollups
│   ├── backup.py            # Export snapshot (zip) / import snapshot with validation
│   ├── requirements.txt     # pyyaml, pytest
│   └── gui/
│       ├── app.py           # Main window: toolbar, count strip, project strip, board, status bar
│       ├── board.py         # Person columns → status groups → cards; filter mode
│       ├── project_strip.py # Rollup cards
│       ├── dragdrop.py      # Drag controller: press/motion/release, drop-target detection
│       ├── widgets.py       # Card widgets, scrollable frame, avatar chip
│       ├── setup_wizard.py  # First-run dialog
│       ├── settings_dialog.py   # People, statuses, custom fields, data folder
│       ├── task_dialog.py   # Create/edit a task (or a single-owner project's card fields)
│       ├── project_dialog.py    # Create/edit a project; checklist editor for single-owner
│       └── project_panel.py # Detail panel opened by clicking a single-owner project
└── tests/
    ├── conftest.py          # Adds src/ to sys.path, tmp data-dir fixtures
    ├── test_models.py
    ├── test_store.py
    ├── test_counts.py
    └── test_backup.py
```

Rule: `models.py`, `store.py`, `counts.py`, `backup.py` have no tkinter imports and are fully unit-tested. The GUI calls into them and never touches YAML directly.

## Data Model

Ids are stable slugs generated once from the name (`sso-migration`, `priya`, `in-progress`). Renaming keeps the id, so renaming a status or person never orphans work. Collisions get a short numeric suffix (`priya-2`).

```python
@dataclass
class Person:
    id: str
    name: str

@dataclass
class Status:
    id: str
    name: str
    # display order = list order in Config.statuses

@dataclass
class Priority:
    id: str
    name: str
    # rank = list order in Config.priorities; first entry is the highest priority

@dataclass
class FieldDef:               # custom field definition
    id: str
    name: str
    type: str                 # "text" | "number" | "date" | "choice" | "checkbox"
    choices: list[str] = field(default_factory=list)   # type == "choice" only

@dataclass
class ChecklistItem:
    text: str
    done: bool = False

@dataclass
class Task:                   # a card on the board (independent projects only)
    id: str
    title: str
    assignee: str | None      # Person.id, None = unassigned
    status: str               # Status.id
    priority: str             # Priority.id
    due: date | None = None
    description: str = ""
    tags: list[str] = field(default_factory=list)
    fields: dict[str, object] = field(default_factory=dict)   # FieldDef.id -> value
    created: date
    updated: date

@dataclass
class Project:
    id: str
    name: str
    type: str                 # "independent" | "single"
    description: str = ""
    builtin: bool = False     # True only for Individual Tasks
    tasks: list[Task]         # independent only
    # single-owner only — the project itself is the card:
    assignee: str | None = None
    status: str | None = None
    priority: str | None = None   # Priority.id
    due: date | None = None
    tags: list[str] = field(default_factory=list)
    fields: dict[str, object] = field(default_factory=dict)
    checklist: list[ChecklistItem] = field(default_factory=list)

@dataclass
class Config:
    version: int = 1
    people: list[Person]
    statuses: list[Status]
    priorities: list[Priority]
    custom_fields: list[FieldDef]
    card_sort: str = "priority"   # see "Card sort order" below
```

A **card** is anything that appears on the board: a `Task` (from an independent project) or a single-owner `Project`. Both carry the same field set (assignee, status, priority, due, description, tags, custom fields) so the task editor dialog serves both.

## Storage Layout

```
<data-dir>/                      any local, mapped-drive, or UNC folder — see "Data folder paths"
├── config.yaml
└── projects/
    ├── individual-tasks.yaml    # builtin: true
    ├── sso-migration.yaml
    └── ...
```

**config.yaml**

```yaml
version: 1
people:
  - id: you
    name: You
  - id: priya
    name: Priya
statuses:
  - id: backlog
    name: Backlog
  - id: in-progress
    name: In Progress
  - id: done
    name: Done
priorities:            # first = highest
  - id: high
    name: High
  - id: medium
    name: Medium
  - id: low
    name: Low
card_sort: priority    # priority | due | title | created | updated
custom_fields:
  - id: client
    name: Client
    type: text
  - id: environment
    name: Environment
    type: choice
    choices: [Prod, Staging, Dev]
```

**projects/sso-migration.yaml** (independent)

```yaml
version: 1
id: sso-migration
name: SSO Migration
type: independent
description: Move all internal apps onto the new SSO provider.
tasks:
  - id: t-3f9a
    title: Migrate ticket queue to new SSO
    assignee: priya
    status: in-progress
    priority: high
    due: 2026-09-25
    description: ""
    tags: [identity]
    fields:
      client: Internal
      environment: Prod
    created: 2026-09-18
    updated: 2026-09-18
```

**projects/client-halden-onboarding.yaml** (single-owner)

```yaml
version: 1
id: client-halden-onboarding
name: Client Halden Onboarding
type: single
assignee: marcus
status: in-progress
priority: medium
due: 2026-10-02
tags: [onboarding]
fields: {}
checklist:
  - text: Kickoff call scheduled
    done: false
  - text: Access provisioning request sent
    done: true
```

Every file starts with `version: 1` so a future schema change can migrate on load.

Writes are atomic: write to `<file>.tmp`, then `os.replace`. Deleting a project moves its file to `<data-dir>/.trash/` rather than unlinking it.

**Where the data dir is remembered:** a tiny JSON file in the platform config location (`~/.config/delegation-planner/prefs.json` on Linux/Mac, `%APPDATA%\delegation-planner\prefs.json` on Windows) holding `{"data_dir": "..."}`. This is the only thing stored outside the data dir, and it's just a pointer — losing it means the app asks where the data is on next launch.

### Data folder paths

The data folder can be anywhere the OS can reach, typed or browsed, in the OS's own path style:

| Style | Example | Notes |
|-------|---------|-------|
| Home-relative | `~/DelegationPlanner` | The default on every OS; `~` expands to `C:\Users\<you>` on Windows |
| Windows drive | `D:\Work\DelegationPlanner` | Backslashes or forward slashes both accepted |
| Mapped drive | `P:\Team\Planner` | Treated like any drive; see "unreachable" below |
| UNC share | `\\fileserver\team\planner` | Works directly, no drive letter needed |
| Environment variables | `%OneDrive%\Planner`, `%USERPROFILE%\Planner`, `$HOME/planner` | Both `%VAR%` and `$VAR` forms are expanded on every OS |
| POSIX | `/Volumes/Team/planner`, `/mnt/share/planner` | macOS SMB mounts, Linux mounts |

Normalisation is one function, `app_prefs.normalize_data_dir(text) -> Path`: `os.path.expandvars` → `Path(...).expanduser()` → `Path.absolute()`. It deliberately does *not* call `resolve()` — resolving a UNC or mapped-drive path can block or fail when the share is offline, and following symlinks isn't wanted for a user-chosen folder. The normalised string is what goes in `prefs.json`.

Because a synced folder (OneDrive, Dropbox, a network share) is a perfectly good data folder, this is the simplest way to use the same data from two machines — but only one copy of the app should have it open at a time; the app doesn't lock files.

**Unreachable at launch.** A data folder that *doesn't exist* is ambiguous: it might be a brand-new install, or a network share that isn't connected right now. Silently running the setup wizard in the second case would create a fresh empty dataset and quietly abandon the real one. So `main.py` distinguishes them:

- `prefs.json` missing entirely → genuine first run → setup wizard.
- `prefs.json` present but the folder (or its `config.yaml`) can't be found → a dialog: *"Your data folder `\\fileserver\team\planner` isn't reachable. Connect to the network and retry, or choose a different folder."* with **Retry**, **Choose folder…**, and **Quit**. "Choose folder…" runs the wizard against the newly picked folder only if that folder has no `config.yaml`; if it does, the app simply opens it.

Project filenames are ASCII slugs, so they're valid on every filesystem. `slugify` also avoids Windows reserved device names (`con`, `prn`, `aux`, `nul`, `com1`–`com9`, `lpt1`–`lpt9`) — a project called "Con" gets the file `con-2.yaml`, since `con.yaml` cannot be created on Windows.

## Main Window

```
┌──────────────────────────────────────────────────────────────────────┐
│ [+ Task] [+ Project]           [Export…] [Import…] [Settings]         │  toolbar
├──────────────────────────────────────────────────────────────────────┤
│ You      2 Backlog · 1 In Progress · 1 Review          4 total        │  count strip
│ Priya    1 Backlog · 1 In Progress · 1 Blocked         3 total        │  (one row per person,
│ Marcus   1 In Progress · 1 Done                        2 total        │   zero-count statuses omitted)
├──────────────────────────────────────────────────────────────────────┤
│ ┌ SSO Migration ─────┐ ┌ Q3 Renewals ───────┐ ┌ Halden Onboarding ─┐ │  project strip
│ │ 1 Backlog 1 In Prog│ │ 2 Backlog 1 In Prog│ │ ▓▓░░ 1/4 done   MA │ │  (wraps into rows)
│ └────────────────────┘ └────────────────────┘ └────────────────────┘ │
├──────────────────────────────────────────────────────────────────────┤
│ Unassigned  │ You           │ Priya         │ Marcus                  │  board
│             │ ● In Progress │ ● In Progress │ ● In Progress           │  (person columns,
│  [card]     │  [card]       │  [card]       │  [single-owner card]    │   status groups
│             │ ● Backlog     │ ● Blocked     │ ● Backlog               │   inside each)
│             │  [card][card] │  [card]       │  [card]                 │
└──────────────────────────────────────────────────────────────────────┘
```

- Person columns appear in the order people are listed in Settings. The board scrolls horizontally when columns overflow.
- Status groups inside a column appear in Settings order; a group with no cards is still drawn as a thin drop target so you can drag into it.
- The count strip and project strip recompute from the in-memory model after every change; no separate cache.

### Cards

```
┌──────────────────────────────┐
│ Migrate ticket queue to SSO  │  title
│ [SSO Migration] HIGH  Sep 25 │  project badge · priority · due
│ identity                     │  tags (if any)
└──────────────────────────────┘
```

Single-owner project cards show a "Project" eyebrow and checklist progress (`▓▓░░ 1/4`) instead of a project badge. Cards in Individual Tasks show no project badge.

Overdue due dates render in the "blocked" colour. The first priority in the configured list (the highest) is emphasised; all others are quiet.

### Card sort order

Within each status group, cards are sorted by the `card_sort` setting, chosen from a "Sort" combobox in the toolbar and persisted in `config.yaml`:

| `card_sort` | Order |
|-------------|-------|
| `priority` | Highest priority first (list order in Settings), then earliest due, then title |
| `due` | Earliest due first (no due date last), then priority, then title |
| `title` | Alphabetical |
| `created` | Oldest first |
| `updated` | Most recently changed first |

The sort applies to every column at once. No manual drag-to-reorder within a group in v1.

### Drag and drop

Implemented natively in tkinter (no third-party DnD library — `tkinterdnd2` is for OS file drops, not in-app widgets).

1. `<ButtonPress-1>` on a card records the press position; nothing happens yet.
2. `<B1-Motion>` beyond a 6px threshold starts a drag: a borderless `Toplevel` ghost showing the card title follows the cursor; the source card dims.
3. On motion, `winfo_containing(x, y)` finds the widget under the cursor; walk up parents until a status group or person column header is found; highlight it as the drop target.
4. `<ButtonRelease-1>`:
   - over a status group → set card's assignee to that column's person (or `None` for Unassigned) and status to that group
   - over a column header → change assignee only, keep status
   - anywhere else → cancel
5. Any successful drop saves the affected project file immediately and re-renders the strip, counts, and board.

Both task cards and single-owner project cards drag the same way (for a single-owner project, the drop updates the project's own assignee/status).

### Clicking things

| Target | Single click | Double click / Enter |
|--------|--------------|----------------------|
| Task card | Select | Open task editor |
| Single-owner project card (on board) | Select | Open project detail panel |
| Project rollup card, independent | Toggle board filter to this project | — |
| Project rollup card, single-owner | Open project detail panel | — |
| Project rollup card, Individual Tasks | Toggle board filter | — |

Right-click on any card opens a context menu: Edit, Move to status ▸, Reassign to ▸, Delete. (Keyboard-friendly fallback for drag-and-drop.)

### Filter mode

When an independent project is filtered, every card not in that project dims (stays visible, stays draggable). A banner replaces the toolbar's left side:

```
Showing: SSO Migration    [Edit project] [+ Task in this project] [× Clear]
```

Clicking the same rollup card again, pressing Escape, or clicking Clear exits filter mode.

### Project detail panel (single-owner)

Modal dialog: project name, owner, status, priority, due, then the checklist with checkboxes (toggling saves immediately), an entry to add an item, and Edit/Delete buttons for items. "Edit details…" opens the full card editor.

### Task editor

One dialog for tasks and for single-owner projects' card fields: Title, Project (combobox — independent projects + Individual Tasks; hidden when editing a single-owner project), Assignee (people + Unassigned), Status, Priority, Due (text entry `YYYY-MM-DD` with validation; no calendar widget in v1), Tags (comma-separated), Description (multi-line), then one row per custom field rendered by type (Entry / Entry with number validation / date entry / Combobox / Checkbutton). Save / Cancel / Delete.

Moving a task to a different project via the editor moves it between YAML files.

### Project dialog

Name, Type (radio: independent / single-owner — locked after creation), Description. For single-owner: assignee, status, priority, due, tags, and the checklist editor. Deleting an independent project with tasks asks whether to delete the tasks or move them to Individual Tasks.

## Settings

Notebook with five tabs. Every change is written to `config.yaml` on Save.

**People** — list with Add / Rename / Move up / Move down / Remove. Removing a person who has cards assigned prompts: "Reassign their N cards to: [combobox of other people or Unassigned]" — removal is refused without a choice.

**Statuses** — same list controls. Removing a status in use prompts to pick a replacement status; refused without one. At least one status must remain.

**Priorities** — same list controls; list order is rank, top = highest. Removing a priority in use prompts to pick a replacement; refused without one. At least one priority must remain. Renaming keeps the id so existing cards are unaffected.

**Custom fields** — list of definitions with Add / Edit / Remove. Add/Edit asks for name and type; type `choice` also asks for the options (one per line). Removing a field asks for confirmation and strips that key from every card's `fields` map. Changing a field's type is not supported in v1 (remove and re-add).

**Data** — an editable entry showing the current data folder (so a UNC path or `%OneDrive%\…` can be typed or pasted directly), a "Browse…" button, and an "Open folder" button. A status line under the entry updates as you type: "folder found, contains N projects" / "folder doesn't exist — will be created" / "not reachable". "Apply" normalises the path, writes `prefs.json`, and reloads from the new folder; if that folder has no `config.yaml` it offers to run the setup wizard against it rather than silently creating an empty config.

## First run

If `prefs.json` is missing or points at a folder without `config.yaml`, the setup wizard opens instead of the main window:

1. **Data folder** — editable entry pre-filled with the OS-native form of `~/DelegationPlanner` (`C:\Users\<you>\DelegationPlanner` on Windows), plus Browse…. Accepts every path style in "Data folder paths"; the same live status line as the Settings Data tab.
2. **People** — a list pre-seeded with one empty row labelled "Your name". Add/Remove. At least one required.
3. **Statuses** — pre-seeded with a suggested list (Backlog, In Progress, Blocked, Review, Done) that can be edited, reordered, or deleted. At least one required.
4. **Priorities** — pre-seeded with High, Medium, Low (top = highest); same editing. At least one required.
5. **Finish** — writes `config.yaml` and `projects/individual-tasks.yaml`, then opens the main window.

The wizard cannot be dismissed without finishing (closing it exits the app).

## Export / Import

**Export** — file-save dialog defaulting to `delegation-planner-YYYY-MM-DD-HHMM.zip` in the user's Documents folder. The zip contains `config.yaml` and `projects/*.yaml` (not `.trash/`). Because the data is several files, a zip is the single-file snapshot the user asked for.

**Import** — file-open dialog for a `.zip`. Steps:

1. Extract to a temp dir and validate: `config.yaml` present and parses into a `Config`; every project file parses into a `Project`; no unknown `version`.
2. Show a summary: "This snapshot has N people, M statuses, K projects (T tasks). Importing replaces your current data. Continue?"
3. On confirm, export the current data to `<data-dir>/.trash/pre-import-YYYY-MM-DD-HHMM.zip` first, then replace `config.yaml` and `projects/` with the snapshot contents.
4. Reload the app.

A snapshot that fails validation is refused with the specific error (file name and reason) — nothing is touched.

## Counts and rollups (`counts.py`)

```python
def cards_for(config, projects) -> list[Card]           # every task + every single-owner project
def person_status_counts(config, projects) -> dict[str | None, dict[str, int]]
    # {person_id or None: {status_id: n}} — only non-zero entries
def project_rollup(project) -> Rollup
    # independent: {status_id: n} + contributor ids
    # single: checklist done/total + owner
def sort_cards(cards, config) -> list[Card]
    # applies config.card_sort using config.priorities order for rank
```

Pure functions over the model, unit tested, and the only place counting and sorting logic lives.

## Cross-platform

Linux, macOS, and Windows are all supported targets, and none is "the main one". Concretely:

- **Stack** — Python 3.11+, tkinter (stdlib), PyYAML. No platform-specific packages. On Debian/Ubuntu tkinter is a separate package (`python3-tk`); `run.sh` checks `import tkinter` and prints the install hint instead of a traceback.
- **Launchers** — `run.sh` (Linux/macOS) and `run.bat` (Windows) already follow the workspace convention; both do the same Python-version / venv / deps checks.
- **Paths** — `pathlib` everywhere; never string-concatenated separators. Prefs file: `%APPDATA%\delegation-planner\` on Windows, `$XDG_CONFIG_HOME` or `~/.config/delegation-planner/` elsewhere. The data folder accepts Windows drive letters, mapped drives, UNC shares, `%VAR%`/`$VAR`, `~`, and POSIX paths — see "Data folder paths" — and an unreachable folder at launch gets a Retry / Choose / Quit dialog, never the wizard. Export default folder is `~/Documents` if it exists, else `~`. Paths are displayed with `os.fspath()` so Windows users see backslashes.
- **Mouse** — right-click is `<Button-3>` on Windows/Linux but `<Button-2>` on macOS (and Ctrl-click is the trackpad convention there); bind all three. Mouse wheel is `<MouseWheel>` with `delta` in ±120 steps on Windows and ±1 on macOS, but `<Button-4>`/`<Button-5>` events on Linux/X11; the scroll helper in `widgets.py` normalises these behind one callback.
- **Drag ghost** — the `overrideredirect` Toplevel works on all three; on macOS it must be created *after* the first motion event (not on press) or it can steal focus and drop the button-release. The design's 6px threshold already does this.
- **"Open folder"** — `os.startfile` (Windows), `subprocess.run(["open", path])` (macOS), `subprocess.run(["xdg-open", path])` (Linux), dispatched on `sys.platform`.
- **Fonts and theme** — use ttk's default theme per platform (`aqua` / `vista` / `clam`) rather than forcing one; size type with relative `font` tuples, never pixel-tuned to one OS. Emoji and non-ASCII in card titles must render — write YAML with `allow_unicode=True` (already in the store).
- **Line endings** — YAML is written with `\n` and read in text mode; git's `.gitattributes` isn't needed since data files live outside the repo.
- **Testing** — the pure-logic test suite runs unchanged on all three. The manual release checklist gets a "run on each OS" note for the GUI items; anything platform-specific found is a TODO under `### Tests`.

## Error handling

- Malformed YAML in a project file: that project is skipped, an error dialog lists the file and the parse error, the rest of the board loads. The file is never overwritten while it's in that state.
- `config.yaml` malformed: the app shows the error and offers to open the setup wizard (which would overwrite it) or quit.
- A card referencing an unknown status or person (e.g. hand-edited YAML): rendered under a synthetic "Unknown" group / the Unassigned column, in the "blocked" colour, and the status bar shows a warning naming the file. Editing the card and saving fixes it.
- A card referencing an unknown priority: rendered with a muted "?" priority tag and sorted after all known priorities; same status-bar warning.
- Save failure (permissions, disk): error dialog with the path; the in-memory change is kept so retrying works.
- Data folder unreachable at launch (network share offline, drive unplugged): Retry / Choose folder… / Quit dialog — see "Data folder paths". Unreachable *while running* (share drops mid-session): the next save fails per the bullet above, and the status bar says the folder is unreachable; nothing in memory is discarded.

## Testing

Automated (`tests/test.sh`):
- `models`: slug generation, collision suffixing, Windows reserved names get a suffix, date round-tripping
- `app_prefs`: `normalize_data_dir` expands `~`, `%VAR%`, `$VAR`, accepts forward and back slashes, and never raises on a non-existent path
- `store`: create data dir, save/load config and both project types, atomic write leaves no `.tmp` on success, malformed file is reported not raised, delete moves to `.trash/`
- `counts`: per-person/per-status counts with unassigned cards, rollups for both project types, empty data, each `card_sort` mode including unknown-priority and no-due-date ordering
- `backup`: export contains expected files and excludes `.trash/`, import validation rejects a zip missing `config.yaml` or with a bad project file, successful import writes the pre-import backup first

Manual (release checklist):
- First run wizard on an empty folder → board opens with Individual Tasks only
- Drag a task to another person's status group → counts, rollup, and YAML file all update
- Drag a single-owner project card → project file's assignee/status update
- Click an independent rollup card → filter; click a single-owner rollup card → panel
- Remove an in-use status/person/priority → replacement prompt; refuse without a choice
- Reorder priorities in Settings → cards re-sort and the emphasised tag follows the new top entry
- Change the toolbar Sort → every column re-sorts; relaunch → the choice is remembered
- Add a `choice` custom field → it appears in the task editor with the right options
- Run the wizard, drag-and-drop, right-click menu, mouse-wheel scroll, and "Open folder" on each of Linux, macOS, Windows
- Windows: point the data folder at a UNC share (`\\server\share\...`) and at a mapped drive; type `%OneDrive%\Planner` and confirm it expands; create a project named "Con" and confirm the file is `con-2.yaml`
- Windows: with the data folder on a network share, disconnect the network and launch → Retry / Choose / Quit dialog, not the wizard; reconnect and Retry → board opens
- Export, delete the data folder, Import → identical board
- Hand-edit a YAML file to reference a missing status → Unknown group appears with a warning

## Defaults chosen — veto if wrong

These weren't asked during brainstorming; sensible defaults were picked so the plan could be written. Any of them can be changed before implementation starts.

1. **An "Unassigned" column** always appears at the far left of the board, so work can exist before it's delegated. Assignee is optional.
2. **Export is a `.zip`**, since the data is multiple files. Import accepts only zips produced by Export.
3. **Data folder defaults to `~/DelegationPlanner/`** (`C:\Users\<you>\DelegationPlanner` on Windows) — visible, easy to copy, changeable in Settings and the setup wizard to any local, mapped, UNC, or synced folder.
4. **Setup wizard pre-seeds suggested statuses and priorities** (Backlog, In Progress, Blocked, Review, Done; High, Medium, Low) as an editable starting point rather than a blank list.
5. **Custom fields apply to all cards** (tasks and single-owner projects alike), not per-project.
6. **Done isn't special** — no auto-hiding of finished work. A "hide done" toggle is a natural TODO once the status list is real.
7. **Due date entry is a plain `YYYY-MM-DD` field** in v1; a calendar picker would add a dependency.
8. **Sort is global**, one setting for every column, chosen from the toolbar. Per-column or manual drag-to-reorder within a group is not in v1.

Reviewed 2026-09-18: priority editability and configurable sort were originally defaults here; the user asked for both to be user-configurable, for Linux/macOS/Windows to all be first-class, and for the data folder to accept Windows-style paths (drive letters, mapped drives, UNC shares) — all now covered in the body above.
