# Delegation Planner v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Cross-reference `docs/superpowers/specs/2026-09-18-delegation-planner-design.md` for the full data model, storage layout, window mockup, and behaviour spec — this plan sequences the build and gives concrete code for the pure-logic modules; GUI tasks give structure, signatures, and key algorithms rather than full widget code, since the design doc's mockups are the source of truth for layout.

**Goal:** A tkinter desktop app for delegating and tracking technical work across a small, growing team. Board = person columns grouped by status, drag-and-drop to move cards, projects (single-owner or independent-subtask) with a rollup strip, everything stored as readable per-project YAML plus a shared config file, with zip export/import for backup.

**Architecture:** Pure-logic layer (`models.py`, `store.py`, `counts.py`, `backup.py`, `app_prefs.py`) with zero tkinter imports, fully unit tested. GUI layer (`gui/`) calls into it and never touches YAML directly. See the design doc for the full file tree.

**Tech Stack:** Python 3.11+ stdlib (`dataclasses`, `pathlib`, `zipfile`, `re`, `datetime`), `pyyaml`, tkinter/`ttk`, `pytest`.

---

## File Map

| Action | Path | Responsibility |
|--------|------|-----------------|
| Modify | `src/requirements.txt` | Add `pyyaml`, `pytest` |
| Modify | `run.sh`, `run.bat` | Check tkinter is importable, print install hint |
| Create | `src/models.py` | Dataclasses + `slugify` |
| Create | `src/store.py` | Data dir layout, YAML load/save, atomic writes, trash |
| Create | `src/app_prefs.py` | Remembers chosen data dir |
| Create | `src/counts.py` | Per-person/status counts, project rollups |
| Create | `src/backup.py` | Export/import zip snapshots |
| Create | `src/gui/app.py` | Main window |
| Create | `src/gui/widgets.py` | Card, avatar, scrollable frame widgets |
| Create | `src/gui/board.py` | Person columns / status groups, filter mode |
| Create | `src/gui/project_strip.py` | Rollup cards |
| Create | `src/gui/dragdrop.py` | Drag controller |
| Create | `src/gui/setup_wizard.py` | First-run wizard |
| Create | `src/gui/settings_dialog.py` | People / statuses / custom fields / data folder |
| Create | `src/gui/task_dialog.py` | Create/edit task or single-owner project's card fields |
| Create | `src/gui/project_dialog.py` | Create/edit project, checklist editor |
| Create | `src/gui/project_panel.py` | Single-owner project detail panel |
| Modify | `src/main.py` | Resolve data dir, first-run check, launch app |
| Create | `tests/conftest.py` | `sys.path` shim, tmp data-dir fixture |
| Create | `tests/test_models.py` | |
| Create | `tests/test_app_prefs.py` | Path normalisation |
| Create | `tests/test_store.py` | |
| Create | `tests/test_counts.py` | |
| Create | `tests/test_backup.py` | |
| Modify | `docs/CHANGELOG.md`, `README.md`, `docs/RELEASE-CHECKLIST.md` | v1 entry, version bump, checklist |

---

## Task 1: Project setup

**Files:**
- Modify: `src/requirements.txt`
- Create: `tests/conftest.py`

- [ ] **Step 1: Add dependencies**

```
pyyaml
pytest
```

- [ ] **Step 2: `tests/conftest.py`**

```python
import os
import sys
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))

import pytest


@pytest.fixture
def data_dir(tmp_path):
    """An empty data directory for store/backup tests."""
    return tmp_path / "data"
```

- [ ] **Step 3: tkinter check in launchers** — Linux distros often ship tkinter as a separate package. In `run.sh`, after the Python-version check, add:

```bash
if ! python3 -c "import tkinter" 2>/dev/null; then
    echo "ERROR: tkinter is not available for this Python."
    echo "  Debian/Ubuntu: sudo apt install python3-tk"
    echo "  Fedora:        sudo dnf install python3-tkinter"
    echo "  macOS (brew):  brew install python-tk"
    sleep 5
    exit 1
fi
```

and the equivalent `python -c "import tkinter"` guard in `run.bat` (python.org Windows installers include it; the message there just says to reinstall Python with the tcl/tk option ticked).

- [ ] **Step 4: Commit**

```bash
git add src/requirements.txt tests/conftest.py run.sh run.bat
git commit -m "chore: add pyyaml/pytest deps, test scaffold, tkinter launcher check"
```

---

## Task 2: Data models (`models.py`)

**Files:**
- Create: `src/models.py`

- [ ] **Step 1: Create `src/models.py`**

```python
from __future__ import annotations

import re
from dataclasses import dataclass, field
from datetime import date
from typing import Optional


# Filenames that Windows refuses to create, in any case and with any extension.
WINDOWS_RESERVED = frozenset(
    {"con", "prn", "aux", "nul"}
    | {f"com{i}" for i in range(1, 10)}
    | {f"lpt{i}" for i in range(1, 10)}
)


def slugify(name: str, existing: set[str] = frozenset()) -> str:
    """Turn a display name into a stable id, unique against `existing`.

    Slugs become filenames, so Windows reserved device names are treated as
    already taken: "Con" -> "con-2", never "con".
    """
    base = re.sub(r"[^a-z0-9]+", "-", name.strip().lower()).strip("-") or "item"
    taken = set(existing) | WINDOWS_RESERVED
    if base not in taken:
        return base
    n = 2
    while f"{base}-{n}" in taken:
        n += 1
    return f"{base}-{n}"


FIELD_TYPES = ("text", "number", "date", "choice", "checkbox")
PROJECT_TYPES = ("independent", "single")
SORT_OPTIONS = ("priority", "due", "title", "created", "updated")


@dataclass
class Person:
    id: str
    name: str


@dataclass
class Status:
    id: str
    name: str


@dataclass
class Priority:
    id: str
    name: str   # rank = list order in Config.priorities; first = highest


@dataclass
class FieldDef:
    id: str
    name: str
    type: str  # one of FIELD_TYPES
    choices: list[str] = field(default_factory=list)  # type == "choice" only


@dataclass
class ChecklistItem:
    text: str
    done: bool = False


@dataclass
class Task:
    id: str
    title: str
    assignee: Optional[str] = None   # Person.id
    status: str = ""                 # Status.id
    priority: str = ""               # Priority.id
    due: Optional[date] = None
    description: str = ""
    tags: list[str] = field(default_factory=list)
    fields: dict = field(default_factory=dict)   # FieldDef.id -> value
    created: date = field(default_factory=date.today)
    updated: date = field(default_factory=date.today)


@dataclass
class Project:
    id: str
    name: str
    type: str  # "independent" | "single"
    description: str = ""
    builtin: bool = False
    tasks: list[Task] = field(default_factory=list)          # independent only
    assignee: Optional[str] = None                            # single only
    status: Optional[str] = None                              # single only
    priority: Optional[str] = None                            # single only, Priority.id
    due: Optional[date] = None                                # single only
    tags: list[str] = field(default_factory=list)             # single only
    fields: dict = field(default_factory=dict)                # single only
    checklist: list[ChecklistItem] = field(default_factory=list)  # single only


@dataclass
class Config:
    version: int = 1
    people: list[Person] = field(default_factory=list)
    statuses: list[Status] = field(default_factory=list)
    priorities: list[Priority] = field(default_factory=list)
    custom_fields: list[FieldDef] = field(default_factory=list)
    card_sort: str = "priority"   # one of SORT_OPTIONS


INDIVIDUAL_TASKS_ID = "individual-tasks"
```

- [ ] **Step 2: Commit**

```bash
git add src/models.py
git commit -m "feat: add data model dataclasses and slug generation"
```

---

## Task 3: Tests for models

**Files:**
- Create: `tests/test_models.py`

- [ ] **Step 1: Create `tests/test_models.py`**

```python
from models import slugify, Task, Project


def test_slugify_basic():
    assert slugify("SSO Migration") == "sso-migration"
    assert slugify("  Q3   Client Renewals!! ") == "q3-client-renewals"


def test_slugify_collision_suffix():
    existing = {"priya"}
    assert slugify("Priya", existing) == "priya-2"
    existing.add("priya-2")
    assert slugify("Priya", existing) == "priya-3"


def test_slugify_empty_falls_back():
    assert slugify("!!!") == "item"


def test_slugify_avoids_windows_reserved_names():
    # These would be uncreatable as con.yaml / lpt1.yaml on Windows
    assert slugify("Con") == "con-2"
    assert slugify("LPT1") == "lpt1-2"
    assert slugify("Console") == "console"  # only exact matches are reserved


def test_task_defaults():
    t = Task(id="t-1", title="Do the thing")
    assert t.assignee is None
    assert t.priority == ""
    assert t.tags == []


def test_project_independent_vs_single_defaults():
    indep = Project(id="p-1", name="SSO Migration", type="independent")
    assert indep.tasks == []
    single = Project(id="p-2", name="Halden Onboarding", type="single")
    assert single.checklist == []
```

- [ ] **Step 2: Run and commit**

```bash
tests/test.sh
git add tests/test_models.py
git commit -m "test: cover slugify and model defaults"
```

---

## Task 4: Storage layer (`store.py`)

**Files:**
- Create: `src/store.py`

Handles the on-disk layout described in the design doc: `config.yaml` + `projects/*.yaml`, atomic writes, soft-delete to `.trash/`. YAML's built-in timestamp resolution means `yaml.safe_load`/`safe_dump` round-trip `datetime.date` values with no custom representer needed — confirm this in tests rather than assuming.

- [ ] **Step 1: Create `src/store.py`**

```python
from __future__ import annotations

import os
import shutil
from dataclasses import asdict
from datetime import datetime
from pathlib import Path

import yaml

from models import (
    ChecklistItem, Config, FieldDef, Person, Priority, Project, Status, Task,
    INDIVIDUAL_TASKS_ID,
)

DEFAULT_STATUSES = ["Backlog", "In Progress", "Blocked", "Review", "Done"]
DEFAULT_PRIORITIES = ["High", "Medium", "Low"]   # first = highest


class StoreError(Exception):
    """Raised for a specific file's load/save failure; carries the path."""

    def __init__(self, path: Path, message: str):
        super().__init__(f"{path}: {message}")
        self.path = path
        self.message = message


# ---------- paths ----------

def config_path(data_dir: Path) -> Path:
    return data_dir / "config.yaml"


def projects_dir(data_dir: Path) -> Path:
    return data_dir / "projects"


def project_path(data_dir: Path, project_id: str) -> Path:
    return projects_dir(data_dir) / f"{project_id}.yaml"


def trash_dir(data_dir: Path) -> Path:
    return data_dir / ".trash"


def is_initialized(data_dir: Path) -> bool:
    return config_path(data_dir).exists()


# ---------- atomic write ----------

def _atomic_write(path: Path, text: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    tmp = path.with_suffix(path.suffix + ".tmp")
    tmp.write_text(text, encoding="utf-8")
    os.replace(tmp, path)


# ---------- config ----------

def _config_to_dict(config: Config) -> dict:
    return {
        "version": config.version,
        "people": [asdict(p) for p in config.people],
        "statuses": [asdict(s) for s in config.statuses],
        "priorities": [asdict(p) for p in config.priorities],
        "card_sort": config.card_sort,
        "custom_fields": [asdict(f) for f in config.custom_fields],
    }


def _config_from_dict(data: dict) -> Config:
    return Config(
        version=data.get("version", 1),
        people=[Person(**p) for p in data.get("people", [])],
        statuses=[Status(**s) for s in data.get("statuses", [])],
        priorities=[Priority(**p) for p in data.get("priorities", [])],
        custom_fields=[FieldDef(**f) for f in data.get("custom_fields", [])],
        card_sort=data.get("card_sort", "priority"),
    )


def load_config(data_dir: Path) -> Config:
    path = config_path(data_dir)
    try:
        data = yaml.safe_load(path.read_text(encoding="utf-8")) or {}
    except Exception as e:
        raise StoreError(path, f"could not parse: {e}") from e
    return _config_from_dict(data)


def save_config(data_dir: Path, config: Config) -> None:
    path = config_path(data_dir)
    _atomic_write(path, yaml.safe_dump(_config_to_dict(config), sort_keys=False, allow_unicode=True))


# ---------- projects ----------

def _task_to_dict(t: Task) -> dict:
    d = asdict(t)
    return d


def _task_from_dict(data: dict) -> Task:
    return Task(
        id=data["id"],
        title=data["title"],
        assignee=data.get("assignee"),
        status=data.get("status", ""),
        priority=data.get("priority", ""),
        due=data.get("due"),
        description=data.get("description", ""),
        tags=list(data.get("tags", [])),
        fields=dict(data.get("fields", {})),
        created=data.get("created") or datetime.now().date(),
        updated=data.get("updated") or datetime.now().date(),
    )


def _project_to_dict(p: Project) -> dict:
    base = {
        "version": 1,
        "id": p.id,
        "name": p.name,
        "type": p.type,
        "description": p.description,
        "builtin": p.builtin,
    }
    if p.type == "independent":
        base["tasks"] = [_task_to_dict(t) for t in p.tasks]
    else:
        base.update({
            "assignee": p.assignee,
            "status": p.status,
            "priority": p.priority,
            "due": p.due,
            "tags": p.tags,
            "fields": p.fields,
            "checklist": [asdict(i) for i in p.checklist],
        })
    return base


def _project_from_dict(path: Path, data: dict) -> Project:
    ptype = data.get("type")
    if ptype not in ("independent", "single"):
        raise StoreError(path, f"unknown project type {ptype!r}")
    p = Project(
        id=data["id"],
        name=data["name"],
        type=ptype,
        description=data.get("description", ""),
        builtin=bool(data.get("builtin", False)),
    )
    if ptype == "independent":
        p.tasks = [_task_from_dict(t) for t in data.get("tasks", [])]
    else:
        p.assignee = data.get("assignee")
        p.status = data.get("status")
        p.priority = data.get("priority")
        p.due = data.get("due")
        p.tags = list(data.get("tags", []))
        p.fields = dict(data.get("fields", {}))
        p.checklist = [ChecklistItem(**i) for i in data.get("checklist", [])]
    return p


def list_project_ids(data_dir: Path) -> list[str]:
    pdir = projects_dir(data_dir)
    if not pdir.exists():
        return []
    return sorted(f.stem for f in pdir.glob("*.yaml"))


def load_project(data_dir: Path, project_id: str) -> Project:
    path = project_path(data_dir, project_id)
    try:
        data = yaml.safe_load(path.read_text(encoding="utf-8")) or {}
    except Exception as e:
        raise StoreError(path, f"could not parse: {e}") from e
    return _project_from_dict(path, data)


def load_all_projects(data_dir: Path) -> tuple[list[Project], list[StoreError]]:
    """Loads every project file; a broken file is skipped and reported, not raised."""
    projects, errors = [], []
    for pid in list_project_ids(data_dir):
        try:
            projects.append(load_project(data_dir, pid))
        except StoreError as e:
            errors.append(e)
    return projects, errors


def save_project(data_dir: Path, project: Project) -> None:
    path = project_path(data_dir, project.id)
    _atomic_write(path, yaml.safe_dump(_project_to_dict(project), sort_keys=False, allow_unicode=True))


def delete_project(data_dir: Path, project_id: str) -> None:
    path = project_path(data_dir, project_id)
    if not path.exists():
        return
    tdir = trash_dir(data_dir)
    tdir.mkdir(parents=True, exist_ok=True)
    stamp = datetime.now().strftime("%Y%m%d%H%M%S")
    shutil.move(str(path), str(tdir / f"{project_id}-{stamp}.yaml"))


# ---------- first-run bootstrap ----------

def initialize(data_dir: Path, people: list[Person], statuses: list[Status],
               priorities: list[Priority]) -> None:
    """Called once by the setup wizard. Creates config.yaml and Individual Tasks."""
    data_dir.mkdir(parents=True, exist_ok=True)
    save_config(data_dir, Config(people=people, statuses=statuses,
                                 priorities=priorities, custom_fields=[]))
    builtin = Project(
        id=INDIVIDUAL_TASKS_ID,
        name="Individual Tasks",
        type="independent",
        description="Standalone tasks that aren't part of a bigger project.",
        builtin=True,
    )
    save_project(data_dir, builtin)
```

- [ ] **Step 2: Commit**

```bash
git add src/store.py
git commit -m "feat: add YAML storage layer with atomic writes and soft delete"
```

---

## Task 5: Tests for storage layer

**Files:**
- Create: `tests/test_store.py`

- [ ] **Step 1: Create `tests/test_store.py`**

```python
from datetime import date

import store
from models import Config, Person, Status, Priority, Project, Task, ChecklistItem


def make_config():
    return Config(
        people=[Person(id="you", name="You"), Person(id="priya", name="Priya")],
        statuses=[Status(id="backlog", name="Backlog"), Status(id="done", name="Done")],
        priorities=[Priority(id="high", name="High"), Priority(id="low", name="Low")],
        custom_fields=[],
        card_sort="due",
    )


def test_initialize_creates_config_and_individual_tasks(data_dir):
    cfg = make_config()
    store.initialize(data_dir, cfg.people, cfg.statuses, cfg.priorities)
    assert store.is_initialized(data_dir)
    assert "individual-tasks" in store.list_project_ids(data_dir)
    proj = store.load_project(data_dir, "individual-tasks")
    assert proj.builtin is True
    assert proj.type == "independent"


def test_config_roundtrip(data_dir):
    cfg = make_config()
    store.save_config(data_dir, cfg)
    loaded = store.load_config(data_dir)
    assert loaded == cfg
    assert loaded.card_sort == "due"
    assert [p.id for p in loaded.priorities] == ["high", "low"]  # order preserved


def test_independent_project_roundtrip_with_date(data_dir):
    task = Task(id="t-1", title="Migrate SSO", assignee="priya", status="backlog", due=date(2026, 9, 25))
    proj = Project(id="sso-migration", name="SSO Migration", type="independent", tasks=[task])
    store.save_project(data_dir, proj)
    loaded = store.load_project(data_dir, "sso-migration")
    assert loaded.tasks[0].due == date(2026, 9, 25)
    assert isinstance(loaded.tasks[0].due, date)


def test_single_owner_project_roundtrip(data_dir):
    proj = Project(
        id="halden-onboarding", name="Client Halden Onboarding", type="single",
        assignee="priya", status="backlog",
        checklist=[ChecklistItem(text="Kickoff call", done=False)],
    )
    store.save_project(data_dir, proj)
    loaded = store.load_project(data_dir, "halden-onboarding")
    assert loaded.checklist[0].text == "Kickoff call"


def test_atomic_write_leaves_no_tmp_file(data_dir):
    store.save_config(data_dir, make_config())
    tmp_files = list(data_dir.glob("*.tmp"))
    assert tmp_files == []


def test_malformed_project_reported_not_raised(data_dir):
    (data_dir / "projects").mkdir(parents=True)
    (data_dir / "projects" / "broken.yaml").write_text("not: valid: yaml: [", encoding="utf-8")
    projects, errors = store.load_all_projects(data_dir)
    assert projects == []
    assert len(errors) == 1
    assert "broken.yaml" in str(errors[0].path)


def test_delete_moves_to_trash(data_dir):
    proj = Project(id="temp-project", name="Temp", type="independent")
    store.save_project(data_dir, proj)
    store.delete_project(data_dir, "temp-project")
    assert "temp-project" not in store.list_project_ids(data_dir)
    assert list((data_dir / ".trash").glob("temp-project-*.yaml"))
```

- [ ] **Step 2: Run and commit**

```bash
tests/test.sh
git add tests/test_store.py
git commit -m "test: cover storage roundtrips, atomic writes, trash, malformed files"
```

---

## Task 6: App-level data dir pointer (`app_prefs.py`)

**Files:**
- Create: `src/app_prefs.py`
- Create: `tests/test_app_prefs.py`

The data folder must accept Windows drive letters, mapped drives, UNC shares, `%VAR%`/`$VAR`, `~`, and POSIX paths (see the design doc's "Data folder paths"). `normalize_data_dir` is the single place that happens. It must never call `Path.resolve()` — resolving an offline UNC path can block or raise — and must never raise on a path that doesn't exist yet.

`get_data_dir()` returns `None` when `prefs.json` is missing so `main.py` can tell "first run" apart from "folder configured but unreachable".

- [ ] **Step 1: Create `src/app_prefs.py`**

```python
from __future__ import annotations

import json
import os
from pathlib import Path
from typing import Optional

APP_NAME = "delegation-planner"
DEFAULT_DATA_DIR = Path.home() / "DelegationPlanner"


def _prefs_path() -> Path:
    if os.name == "nt":
        base = Path(os.environ.get("APPDATA", Path.home()))
    else:
        base = Path(os.environ.get("XDG_CONFIG_HOME", Path.home() / ".config"))
    return base / APP_NAME / "prefs.json"


def normalize_data_dir(text: str) -> Path:
    """Expand %VAR%, $VAR and ~, then make absolute. Never resolves symlinks or touches disk.

    Accepts "D:\\Work\\Planner", "\\\\server\\share\\planner", "P:/Team/Planner",
    "%OneDrive%\\Planner", "$HOME/planner", "~/DelegationPlanner", "/mnt/share/planner".
    """
    expanded = os.path.expandvars(text.strip())
    return Path(expanded).expanduser().absolute()


def get_data_dir() -> Optional[Path]:
    """The configured data dir, or None if the app has never been set up on this machine."""
    path = _prefs_path()
    try:
        data = json.loads(path.read_text(encoding="utf-8"))
        return Path(data["data_dir"])
    except Exception:
        return None


def set_data_dir(data_dir: Path) -> None:
    path = _prefs_path()
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps({"data_dir": os.fspath(data_dir)}), encoding="utf-8")
```

- [ ] **Step 2: Create `tests/test_app_prefs.py`**

```python
import os
import sys
from pathlib import Path

import pytest

import app_prefs


def test_normalize_expands_tilde():
    p = app_prefs.normalize_data_dir("~/DelegationPlanner")
    assert p == (Path.home() / "DelegationPlanner").absolute()
    assert "~" not in os.fspath(p)


def test_normalize_expands_dollar_var(monkeypatch, tmp_path):
    monkeypatch.setenv("PLANNER_HOME", os.fspath(tmp_path))
    p = app_prefs.normalize_data_dir("$PLANNER_HOME/data")
    assert p == (tmp_path / "data").absolute()


def test_normalize_expands_percent_var(monkeypatch, tmp_path):
    # %VAR% is expanded by os.path.expandvars on every OS, not only Windows
    monkeypatch.setenv("PLANNER_HOME", os.fspath(tmp_path))
    p = app_prefs.normalize_data_dir("%PLANNER_HOME%/data")
    assert p == (tmp_path / "data").absolute()


def test_normalize_does_not_raise_for_missing_path():
    p = app_prefs.normalize_data_dir("/definitely/not/here/planner")
    assert p.is_absolute()


def test_normalize_strips_whitespace():
    assert app_prefs.normalize_data_dir("  ~/x  ") == app_prefs.normalize_data_dir("~/x")


@pytest.mark.skipif(sys.platform != "win32", reason="Windows path semantics")
def test_normalize_windows_styles():
    assert app_prefs.normalize_data_dir(r"D:\Work\Planner") == Path(r"D:\Work\Planner")
    assert app_prefs.normalize_data_dir("D:/Work/Planner") == Path(r"D:\Work\Planner")
    unc = app_prefs.normalize_data_dir(r"\\server\share\planner")
    assert os.fspath(unc) == r"\\server\share\planner"


def test_get_data_dir_none_when_prefs_missing(monkeypatch, tmp_path):
    monkeypatch.setattr(app_prefs, "_prefs_path", lambda: tmp_path / "nope" / "prefs.json")
    assert app_prefs.get_data_dir() is None


def test_set_then_get_roundtrip(monkeypatch, tmp_path):
    monkeypatch.setattr(app_prefs, "_prefs_path", lambda: tmp_path / "prefs.json")
    app_prefs.set_data_dir(tmp_path / "data")
    assert app_prefs.get_data_dir() == tmp_path / "data"
```

- [ ] **Step 3: Run and commit**

```bash
tests/test.sh
git add src/app_prefs.py tests/test_app_prefs.py
git commit -m "feat: data dir prefs with Windows/UNC/env-var path normalisation"
```

---

## Task 7: Counts module (`counts.py`)

**Files:**
- Create: `src/counts.py`

- [ ] **Step 1: Create `src/counts.py`**

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import date
from typing import Optional

from models import Config, Project, Task


@dataclass
class Card:
    """A unified view over an independent-project task or a single-owner project."""
    kind: str               # "task" | "single_project"
    id: str                 # Task.id or Project.id
    project_id: str         # owning project's id
    project_name: str
    title: str
    assignee: Optional[str]
    status: Optional[str]
    priority: Optional[str]  # Priority.id
    due: Optional[date]
    created: Optional[date]  # None for single-owner projects (no created field in v1)
    updated: Optional[date]


def cards_for(projects: list[Project]) -> list[Card]:
    cards: list[Card] = []
    for p in projects:
        if p.type == "independent":
            for t in p.tasks:
                cards.append(Card("task", t.id, p.id, p.name, t.title, t.assignee, t.status,
                                  t.priority, t.due, t.created, t.updated))
        else:
            cards.append(Card("single_project", p.id, p.id, p.name, p.name, p.assignee, p.status,
                              p.priority, p.due, None, None))
    return cards


FAR_FUTURE = date.max


def sort_cards(cards: list[Card], config: Config) -> list[Card]:
    """Applies config.card_sort. Unknown priorities rank after all known ones."""
    rank = {p.id: i for i, p in enumerate(config.priorities)}
    unknown = len(rank)

    def prio(c: Card) -> int:
        return rank.get(c.priority, unknown)

    def due(c: Card) -> date:
        return c.due or FAR_FUTURE

    def title(c: Card) -> str:
        return c.title.lower()

    mode = config.card_sort
    if mode == "due":
        key = lambda c: (due(c), prio(c), title(c))
    elif mode == "title":
        key = lambda c: (title(c),)
    elif mode == "created":
        key = lambda c: (c.created or FAR_FUTURE, title(c))
    elif mode == "updated":
        return sorted(cards, key=lambda c: (c.updated or date.min, title(c)), reverse=True)
    else:  # "priority" and any unrecognised value
        key = lambda c: (prio(c), due(c), title(c))
    return sorted(cards, key=key)


def person_status_counts(cards: list[Card]) -> dict[Optional[str], dict[str, int]]:
    """{person_id or None: {status_id: n}} — zero-count entries omitted."""
    counts: dict[Optional[str], dict[str, int]] = {}
    for c in cards:
        by_status = counts.setdefault(c.assignee, {})
        by_status[c.status] = by_status.get(c.status, 0) + 1
    return counts


@dataclass
class Rollup:
    project_id: str
    project_name: str
    project_type: str
    status_counts: dict[str, int]   # independent only
    contributors: list[str]          # independent only, person ids in first-seen order
    checklist_done: int              # single only
    checklist_total: int             # single only
    owner: Optional[str]             # single only


def project_rollup(project: Project) -> Rollup:
    if project.type == "independent":
        status_counts: dict[str, int] = {}
        contributors: list[str] = []
        for t in project.tasks:
            status_counts[t.status] = status_counts.get(t.status, 0) + 1
            if t.assignee and t.assignee not in contributors:
                contributors.append(t.assignee)
        return Rollup(project.id, project.name, "independent", status_counts, contributors, 0, 0, None)
    total = len(project.checklist)
    done = sum(1 for i in project.checklist if i.done)
    return Rollup(project.id, project.name, "single", {}, [], done, total, project.assignee)
```

- [ ] **Step 2: Commit**

```bash
git add src/counts.py
git commit -m "feat: add pure counting, rollup, and card sorting functions"
```

---

## Task 8: Tests for counts

**Files:**
- Create: `tests/test_counts.py`

- [ ] **Step 1: Create `tests/test_counts.py`**

```python
from datetime import date

from counts import cards_for, person_status_counts, project_rollup, sort_cards
from models import Config, Priority, Project, Task, ChecklistItem


def test_cards_for_mixes_task_and_single_project():
    indep = Project(id="sso", name="SSO Migration", type="independent", tasks=[
        Task(id="t1", title="A", assignee="priya", status="backlog"),
        Task(id="t2", title="B", assignee="marcus", status="in-progress"),
    ])
    single = Project(id="halden", name="Halden Onboarding", type="single", assignee="marcus", status="in-progress")
    cards = cards_for([indep, single])
    assert len(cards) == 3
    assert {c.kind for c in cards} == {"task", "single_project"}


def test_person_status_counts_groups_and_omits_zero():
    indep = Project(id="sso", name="SSO", type="independent", tasks=[
        Task(id="t1", title="A", assignee="priya", status="backlog"),
        Task(id="t2", title="B", assignee="priya", status="backlog"),
        Task(id="t3", title="C", assignee=None, status="backlog"),
    ])
    counts = person_status_counts(cards_for([indep]))
    assert counts["priya"]["backlog"] == 2
    assert counts[None]["backlog"] == 1
    assert "in-progress" not in counts["priya"]


def test_rollup_independent_counts_and_contributors():
    proj = Project(id="sso", name="SSO", type="independent", tasks=[
        Task(id="t1", title="A", assignee="priya", status="backlog"),
        Task(id="t2", title="B", assignee="marcus", status="blocked"),
        Task(id="t3", title="C", assignee="priya", status="backlog"),
    ])
    r = project_rollup(proj)
    assert r.status_counts == {"backlog": 2, "blocked": 1}
    assert r.contributors == ["priya", "marcus"]


def test_rollup_single_owner_checklist_progress():
    proj = Project(id="halden", name="Halden", type="single", assignee="marcus",
                    checklist=[ChecklistItem("a", True), ChecklistItem("b", False)])
    r = project_rollup(proj)
    assert (r.checklist_done, r.checklist_total) == (1, 2)
    assert r.owner == "marcus"


def _sort_fixture():
    config = Config(priorities=[Priority("high", "High"), Priority("low", "Low")])
    proj = Project(id="p", name="P", type="independent", tasks=[
        Task(id="a", title="Zeta", priority="low", due=date(2026, 1, 1), created=date(2026, 1, 3)),
        Task(id="b", title="Alpha", priority="high", due=None, created=date(2026, 1, 1)),
        Task(id="c", title="Mid", priority="does-not-exist", due=date(2026, 1, 2), created=date(2026, 1, 2)),
    ])
    return config, cards_for([proj])


def test_sort_priority_ranks_by_config_order_unknown_last():
    config, cards = _sort_fixture()
    config.card_sort = "priority"
    assert [c.id for c in sort_cards(cards, config)] == ["b", "a", "c"]


def test_sort_due_puts_no_due_last():
    config, cards = _sort_fixture()
    config.card_sort = "due"
    assert [c.id for c in sort_cards(cards, config)] == ["a", "c", "b"]


def test_sort_title_and_created():
    config, cards = _sort_fixture()
    config.card_sort = "title"
    assert [c.id for c in sort_cards(cards, config)] == ["b", "c", "a"]
    config.card_sort = "created"
    assert [c.id for c in sort_cards(cards, config)] == ["b", "c", "a"]


def test_sort_unrecognised_mode_falls_back_to_priority():
    config, cards = _sort_fixture()
    config.card_sort = "banana"
    assert [c.id for c in sort_cards(cards, config)] == ["b", "a", "c"]
```

- [ ] **Step 2: Run and commit**

```bash
tests/test.sh
git add tests/test_counts.py
git commit -m "test: cover card unification, counts, rollups, and every sort mode"
```

---

## Task 9: Backup module (`backup.py`)

**Files:**
- Create: `src/backup.py`

- [ ] **Step 1: Create `src/backup.py`**

```python
from __future__ import annotations

import shutil
import zipfile
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path

import yaml

import store
from models import Config


class BackupError(Exception):
    """Raised with a user-facing reason; import must be refused, nothing written."""


@dataclass
class SnapshotSummary:
    people: int
    statuses: int
    projects: int
    tasks: int


def default_export_name() -> str:
    return f"delegation-planner-{datetime.now().strftime('%Y-%m-%d-%H%M')}.zip"


def export_snapshot(data_dir: Path, dest_zip: Path) -> None:
    dest_zip.parent.mkdir(parents=True, exist_ok=True)
    with zipfile.ZipFile(dest_zip, "w", zipfile.ZIP_DEFLATED) as zf:
        zf.write(store.config_path(data_dir), arcname="config.yaml")
        for pid in store.list_project_ids(data_dir):
            zf.write(store.project_path(data_dir, pid), arcname=f"projects/{pid}.yaml")


def _validate_and_summarize(extracted: Path) -> SnapshotSummary:
    cfg_path = extracted / "config.yaml"
    if not cfg_path.exists():
        raise BackupError("config.yaml is missing from the snapshot")
    try:
        config = store.load_config(extracted)
    except store.StoreError as e:
        raise BackupError(f"config.yaml is invalid: {e.message}") from e

    task_count = 0
    project_count = 0
    for path in sorted((extracted / "projects").glob("*.yaml")) if (extracted / "projects").exists() else []:
        try:
            project = store._project_from_dict(path, yaml.safe_load(path.read_text(encoding="utf-8")) or {})
        except Exception as e:
            raise BackupError(f"{path.name} is invalid: {e}") from e
        project_count += 1
        task_count += len(project.tasks)

    return SnapshotSummary(len(config.people), len(config.statuses), project_count, task_count)


def preview_snapshot(zip_path: Path, tmp_extract_dir: Path) -> SnapshotSummary:
    """Extracts to tmp_extract_dir and validates. Caller shows the summary before import_snapshot()."""
    if tmp_extract_dir.exists():
        shutil.rmtree(tmp_extract_dir)
    with zipfile.ZipFile(zip_path) as zf:
        zf.extractall(tmp_extract_dir)
    return _validate_and_summarize(tmp_extract_dir)


def import_snapshot(data_dir: Path, validated_extract_dir: Path) -> None:
    """Call only after preview_snapshot() succeeded on the same extract dir."""
    trash = store.trash_dir(data_dir)
    trash.mkdir(parents=True, exist_ok=True)
    stamp = datetime.now().strftime("%Y%m%d%H%M%S")
    if store.is_initialized(data_dir):
        export_snapshot(data_dir, trash / f"pre-import-{stamp}.zip")

    if store.projects_dir(data_dir).exists():
        shutil.rmtree(store.projects_dir(data_dir))
    shutil.copy(validated_extract_dir / "config.yaml", store.config_path(data_dir))
    if (validated_extract_dir / "projects").exists():
        shutil.copytree(validated_extract_dir / "projects", store.projects_dir(data_dir))
```

- [ ] **Step 2: Commit**

```bash
git add src/backup.py
git commit -m "feat: add zip export/import with validation and pre-import backup"
```

---

## Task 10: Tests for backup

**Files:**
- Create: `tests/test_backup.py`

- [ ] **Step 1: Create `tests/test_backup.py`**

```python
import zipfile

import pytest

import store
import backup
from models import Person, Status, Priority, Project, Task

PEOPLE = [Person("you", "You")]
STATUSES = [Status("backlog", "Backlog")]
PRIORITIES = [Priority("high", "High")]


def seed(data_dir):
    store.initialize(data_dir, PEOPLE, STATUSES, PRIORITIES)
    store.save_project(data_dir, Project(
        id="sso-migration", name="SSO Migration", type="independent",
        tasks=[Task(id="t1", title="Migrate", assignee="you", status="backlog")],
    ))


def test_export_contains_config_and_projects_excludes_trash(data_dir, tmp_path):
    seed(data_dir)
    store.delete_project(data_dir, "sso-migration")  # populates .trash
    store.save_project(data_dir, Project(id="q3", name="Q3 Renewals", type="independent"))
    dest = tmp_path / "out.zip"
    backup.export_snapshot(data_dir, dest)
    with zipfile.ZipFile(dest) as zf:
        names = zf.namelist()
    assert "config.yaml" in names
    assert "projects/q3.yaml" in names
    assert not any(".trash" in n for n in names)


def test_import_rejects_zip_missing_config(data_dir, tmp_path):
    bad_zip = tmp_path / "bad.zip"
    with zipfile.ZipFile(bad_zip, "w") as zf:
        zf.writestr("projects/x.yaml", "id: x\nname: X\ntype: independent\n")
    with pytest.raises(backup.BackupError, match="config.yaml"):
        backup.preview_snapshot(bad_zip, tmp_path / "extract")


def test_import_rejects_bad_project_file(data_dir, tmp_path):
    bad_zip = tmp_path / "bad.zip"
    with zipfile.ZipFile(bad_zip, "w") as zf:
        zf.writestr("config.yaml", "version: 1\npeople: []\nstatuses: []\ncustom_fields: []\n")
        zf.writestr("projects/broken.yaml", "type: not-a-real-type\n")
    with pytest.raises(backup.BackupError, match="broken.yaml"):
        backup.preview_snapshot(bad_zip, tmp_path / "extract")


def test_successful_roundtrip_writes_pre_import_backup(data_dir, tmp_path):
    seed(data_dir)
    export_dest = tmp_path / "snapshot.zip"
    backup.export_snapshot(data_dir, export_dest)

    other_dir = tmp_path / "other-machine"
    store.initialize(other_dir, PEOPLE, STATUSES, PRIORITIES)
    extract = tmp_path / "extract"
    summary = backup.preview_snapshot(export_dest, extract)
    assert summary.projects == 2  # individual-tasks (builtin) + sso-migration
    backup.import_snapshot(other_dir, extract)

    assert "sso-migration" in store.list_project_ids(other_dir)
    assert list((other_dir / ".trash").glob("pre-import-*.zip"))
```

- [ ] **Step 2: Run and commit**

```bash
tests/test.sh
git add tests/test_backup.py
git commit -m "test: cover export contents, import validation, pre-import safety backup"
```

---

## Task 11: GUI skeleton and app bootstrap

**Files:**
- Create: `src/gui/__init__.py` (empty)
- Create: `src/gui/app.py`
- Modify: `src/main.py`

Wires the pure-logic layer to a window that opens but doesn't render the board yet (Task 12 adds it). This task establishes the `App` object every later GUI task extends.

- [ ] **Step 1: `src/gui/app.py`** — a `tk.Tk` subclass `App` holding:
  - `self.data_dir: Path`, `self.config: Config`, `self.projects: list[Project]` (loaded via `store.load_all_projects`)
  - `self.selected_filter: str | None` (project id currently filtered, or None)
  - `reload()` — re-reads everything from disk and re-renders; called after every save
  - `save_project(project)` — wraps `store.save_project` + `reload()`
  - Menu/toolbar frame (placeholders: `+ Task`, `+ Project`, `Export…`, `Import…`, `Settings` buttons wired to `command=lambda: None` for now)
  - A status bar `tk.Label` at the bottom for load-error warnings (Task 16 wires it up)
  - If `store.load_all_projects` returns errors, show them in a `messagebox.showwarning` listing each file

- [ ] **Step 2: `src/main.py`** — three launch cases, per the design doc's "Unreachable at launch": never set up → wizard; set up but folder missing → Retry / Choose / Quit dialog (so an offline network share doesn't get replaced by a fresh empty dataset); set up and present → open.

```python
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parent))

import app_prefs
import store
from gui.app import App
from gui.setup_wizard import run_setup_wizard, ask_unreachable


def main():
    data_dir = app_prefs.get_data_dir()

    if data_dir is None:
        # First run on this machine.
        data_dir = run_setup_wizard(app_prefs.DEFAULT_DATA_DIR)
        if data_dir is None:
            return
        app_prefs.set_data_dir(data_dir)

    while not store.is_initialized(data_dir):
        # Configured, but config.yaml can't be found: offline share, unplugged drive, or deleted.
        action, chosen = ask_unreachable(data_dir)
        if action == "quit":
            return
        if action == "retry":
            continue
        # action == "choose": chosen is a normalised Path
        if store.is_initialized(chosen):
            data_dir = chosen
        else:
            data_dir = run_setup_wizard(chosen)
            if data_dir is None:
                return
        app_prefs.set_data_dir(data_dir)

    App(data_dir).mainloop()


if __name__ == "__main__":
    main()
```

(`run_setup_wizard` and `ask_unreachable` are built in Task 14; stub both to return `None` / `("quit", None)` so `main.py` is committable now, then remove the stubs in Task 14.)

- [ ] **Step 3: Commit**

```bash
git add src/gui/__init__.py src/gui/app.py src/main.py
git commit -m "feat: add app bootstrap and main window skeleton"
```

---

## Task 12: Card and layout widgets (`widgets.py`)

**Files:**
- Create: `src/gui/widgets.py`

- [ ] **Step 1: Cross-platform input helpers** — two small functions the rest of the GUI uses instead of binding raw events:

```python
import sys

IS_MAC = sys.platform == "darwin"
IS_WIN = sys.platform.startswith("win")


def bind_right_click(widget, callback):
    """Right-click is Button-3 on Windows/Linux, Button-2 on macOS; Ctrl-click is the mac trackpad idiom."""
    widget.bind("<Button-3>", callback)
    if IS_MAC:
        widget.bind("<Button-2>", callback)
        widget.bind("<Control-Button-1>", callback)


def bind_mouse_wheel(widget, on_scroll):
    """Calls on_scroll(steps) with a small signed int on every platform."""
    if IS_MAC:
        widget.bind("<MouseWheel>", lambda e: on_scroll(-e.delta), add="+")          # delta is ±1..±n
    elif IS_WIN:
        widget.bind("<MouseWheel>", lambda e: on_scroll(-e.delta // 120), add="+")   # delta is ±120 multiples
    else:
        widget.bind("<Button-4>", lambda e: on_scroll(-1), add="+")                  # X11
        widget.bind("<Button-5>", lambda e: on_scroll(1), add="+")
```

- [ ] **Step 2: `VerticalScrollFrame`** — a `ttk.Frame` containing a `Canvas` + inner frame + `ttk.Scrollbar`, wheel-bound via `bind_mouse_wheel` (bind on the canvas *and* recursively on children added later, since Linux/X11 wheel events don't bubble the way `<MouseWheel>` does), used for each board column's card stack. (Standard tkinter scrollable-frame pattern; no third-party dep.)

- [ ] **Step 3: `Avatar`** — small `tk.Label`-based circle-ish chip showing up to 2 initials from a person's name.

- [ ] **Step 4: `CardWidget`** — a `ttk.Frame` rendering one `counts.Card`:
  - Title label (wraps at column width)
  - Meta row: project badge (hidden for Individual Tasks and for a single-owner project's own card), priority tag (name looked up from `config.priorities`; emphasised style only when it's the first entry; a muted "?" if the id isn't in the list), due date (recolored if `due < date.today()`), tags
  - For `kind == "single_project"`: replace project badge + due with an eyebrow "Project" label and a checklist progress bar (`counts.project_rollup(project).checklist_done/total`)
  - Takes `on_click`, `on_double_click`, `on_right_click` callbacks (right-click bound through `bind_right_click`; wired in later tasks) and exposes `self.card: Card` for the drag controller to read

- [ ] **Step 5: Commit**

```bash
git add src/gui/widgets.py
git commit -m "feat: add cross-platform input helpers, scrollable frame, avatar, card widgets"
```

---

## Task 13: Board rendering (`board.py`)

**Files:**
- Create: `src/gui/board.py`
- Modify: `src/gui/app.py` — mount the board below the toolbar/count-strip area

Implements the person-columns/status-groups layout from the design doc's mockup, including the "Unassigned" column.

- [ ] **Step 1: `BoardView(ttk.Frame)`**
  - Built from `self.config.people` (Unassigned first, then people in config order) and `self.config.statuses` (in config order)
  - For each person, a column; for each status inside it, a `VerticalScrollFrame` region with a header (`status.name`) and the `CardWidget`s whose `assignee == person.id and status == status.id`, ordered by `counts.sort_cards(group_cards, app.config)` — never a local sort
  - A status group with zero cards still renders (empty droppable region) rather than collapsing
  - Horizontal scroll: the whole column row sits in a `Canvas`+`Scrollbar` (reuse the `VerticalScrollFrame` pattern rotated, or a small `HorizontalScrollFrame` twin in `widgets.py`)
  - `render(cards: list[Card])` — clears and rebuilds; `app.py` calls this from `reload()`
  - `set_filter(project_id: str | None)` — when set, apply a `filtering`-equivalent dim by lowering non-matching cards' relief/foreground (tkinter has no CSS opacity; approximate with a muted style) rather than hiding them

- [ ] **Step 2: Count strip** — a `ttk.Frame` above the board, one row per person (+ Unassigned only if it has cards) built from `counts.person_status_counts`; a "N total" label per row per the mockup

- [ ] **Step 3: Sort combobox** — a readonly `ttk.Combobox` on the right of the toolbar labelled "Sort:", values from `models.SORT_OPTIONS` shown with friendly names (`priority` → "Priority", `due` → "Due date", `title` → "Title", `created` → "Created", `updated` → "Last updated"). Selecting writes `app.config.card_sort`, calls `store.save_config`, and `app.reload()`

- [ ] **Step 4: Wire into `app.py`** — `App.reload()` now calls `counts.cards_for(self.projects)`, updates the count strip, and calls `self.board.render(cards)`

- [ ] **Step 5: Commit**

```bash
git add src/gui/board.py src/gui/app.py
git commit -m "feat: render person/status board with count strip and sort selector"
```

---

## Task 14: Setup wizard

**Files:**
- Create: `src/gui/setup_wizard.py`
- Modify: `src/main.py` — remove the Task 11 stub import path if changed

- [ ] **Step 1: `run_setup_wizard(default_data_dir: Path) -> Path | None`**
  - A `tk.Tk` root (this runs *before* `App` exists) hosting a `ttk.Notebook` or simple paged `Frame` swap for: Data folder → People → Statuses → Finish, per the design doc
  - Data folder page: an *editable* entry pre-filled with `os.fspath(default_data_dir)` (so Windows users see `C:\Users\...`), a "Browse…" button using `filedialog.askdirectory`, and a live status label driven by a `StringVar` trace: run `app_prefs.normalize_data_dir(text)` and report "folder found, contains N projects" / "folder doesn't exist — will be created" / "not reachable" (an `OSError` from `.exists()` on an offline share). Next is disabled while the entry is blank. Typed UNC paths, mapped drives, and `%VAR%` forms must all work here — this is the primary way a network share gets chosen
  - Extract that entry + Browse + status trio as `DataFolderPicker(ttk.Frame)` — the Settings Data tab (Task 19) and `ask_unreachable` reuse it
  - People page: an editable `Listbox` (or repeated `Entry` rows) starting with one empty row labeled "Your name"; Add/Remove buttons; Next disabled until at least one non-empty name exists
  - Statuses page: same list pattern, pre-seeded with `store.DEFAULT_STATUSES`; Next disabled if the list is empty
  - Priorities page: same list pattern, pre-seeded with `store.DEFAULT_PRIORITIES`, with a caption "Top of the list is the highest priority"; Next disabled if empty
  - Finish page: summary, "Create" button calls `store.initialize(data_dir, people, statuses, priorities)` then destroys the wizard and returns `data_dir`
  - Because the People / Statuses / Priorities pages are the same widget with different seed data and captions, build one `EditableListPage` class and instantiate it three times — Settings (Task 19) reuses it too
  - Closing the window (WM_DELETE_WINDOW) before Finish returns `None`

- [ ] **Step 2: `ask_unreachable(data_dir: Path) -> tuple[str, Path | None]`** — a small `tk.Tk`-rooted dialog (also runs before `App` exists) with the message from the design doc naming the folder, three buttons: **Retry** → `("retry", None)`, **Choose folder…** → shows a `DataFolderPicker`, returns `("choose", normalized_path)`, **Quit** → `("quit", None)`. Closing the window counts as Quit

- [ ] **Step 3: Commit**

```bash
git add src/gui/setup_wizard.py
git commit -m "feat: add first-run setup wizard and unreachable-folder dialog"
```

- [ ] **Step 4: Manual check** — delete `prefs.json`, run `./run.sh`, confirm the wizard appears and creates a working board with only "Individual Tasks". Then rename the data folder (leaving `prefs.json`) and relaunch: the Retry / Choose / Quit dialog must appear, *not* the wizard; rename it back and Retry → board opens

---

## Task 15: Task editor dialog

**Files:**
- Create: `src/gui/task_dialog.py`

- [ ] **Step 1: `open_task_dialog(app, project: Project | None, task: Task | None) -> None`**
  - Modal `Toplevel`. `task=None` means creating; `project=None` while creating means the Project combobox starts unset (must be chosen before Save enables)
  - Fields per the design doc: Title, Project (combobox: all `independent` projects + Individual Tasks; hidden when editing a single-owner project's own card — see Task 17), Assignee, Status, Priority (readonly combobox of `config.priorities` names in list order, defaulting to the first for new cards), Due (`Entry` validated against `YYYY-MM-DD` on focus-out, red border + disabled Save if invalid and non-empty), Tags (comma-separated `Entry`, split/joined on save), Description (`Text` widget), then one row per `app.config.custom_fields` rendered by `.type` (`text`→Entry, `number`→Entry with numeric validation, `date`→Entry same as Due, `choice`→readonly Combobox from `.choices`, `checkbox`→Checkbutton)
  - Save: builds/updates a `Task`, sets `updated = date.today()`, moves it between `Project.tasks` lists if the Project field changed, calls `app.save_project(...)` for the source and (if different) destination project, closes
  - Delete (edit mode only): confirm, remove from `project.tasks`, save, close

- [ ] **Step 2: Wire "+ Task" toolbar button** in `app.py` to `open_task_dialog(app, None, None)`

- [ ] **Step 3: Wire card double-click** in `board.py`'s `CardWidget` creation to open the editor for `kind == "task"` cards

- [ ] **Step 4: Commit**

```bash
git add src/gui/task_dialog.py src/gui/app.py src/gui/board.py
git commit -m "feat: add task create/edit dialog with custom field rendering"
```

---

## Task 16: Project dialog and rollup strip

**Files:**
- Create: `src/gui/project_dialog.py`
- Create: `src/gui/project_strip.py`
- Modify: `src/gui/app.py`

- [ ] **Step 1: `open_project_dialog(app, project: Project | None) -> None`**
  - Name, Type (radio, disabled/locked when editing an existing project), Description
  - Type == single: also Assignee, Status, Priority, Due, Tags, and a checklist editor (list of `ChecklistItem` rows with a checkbox, text `Entry`, remove button, plus an "Add item" entry+button)
  - Save calls `store.save_project`; new project ids via `models.slugify(name, existing={p.id for p in app.projects})`
  - Delete (edit mode, non-builtin only): if `type == "independent"` and `tasks` is non-empty, ask "Delete its N tasks, or move them to Individual Tasks?" (per design doc) before calling `store.delete_project`
  - Individual Tasks (`builtin=True`) cannot be deleted or renamed — Delete button hidden, Name field disabled

- [ ] **Step 2: `ProjectStripView(ttk.Frame)`**
  - One `project.Rollup`-driven card per project, in a wrapping grid (`grid` with a computed column count based on frame width, recalculated on `<Configure>`) per the "wraps into rows" behavior validated during design
  - Independent: status breakdown chips (first 3, then "+N more" — reuse the truncation shown in the design mockup), contributor avatars
  - Single: checklist progress bar + owner avatar
  - Individual Tasks: visually muted (per design doc) — lighter border style
  - Click routes through `on_project_click(project)` passed in from `app.py`

- [ ] **Step 3: Wire in `app.py`**
  - Mount `ProjectStripView` between the count strip and the board
  - `on_project_click`: if `project.type == "independent"`, toggle `self.board.set_filter(project.id if not already filtering this one else None)` and show/hide the filter banner (label + "Clear" button replacing the toolbar's left side, per the design doc); if `project.type == "single"`, call `open_project_panel(app, project)` (Task 18)
  - Wire "+ Project" toolbar button to `open_project_dialog(app, None)`

- [ ] **Step 4: Commit**

```bash
git add src/gui/project_dialog.py src/gui/project_strip.py src/gui/app.py
git commit -m "feat: add project dialog, rollup strip, and board filter-by-project"
```

---

## Task 17: Single-owner project detail panel

**Files:**
- Create: `src/gui/project_panel.py`
- Modify: `src/gui/board.py` — a single-owner project's own card on the board opens this panel instead of `task_dialog`
- Modify: `src/gui/task_dialog.py` — when called for a single-owner project's fields (from the panel's "Edit details…"), hide the Project combobox per the design doc

- [ ] **Step 1: `open_project_panel(app, project: Project) -> None`**
  - Modal `Toplevel`: name, owner, status, priority, due, then the checklist — each row a `Checkbutton` (toggling calls `store.save_project` immediately, per the design doc's "toggling saves immediately") plus an entry+button to add a new item, and per-item remove buttons
  - "Edit details…" button opens a reduced task-dialog-style form (reuse `task_dialog`'s field-rendering helpers rather than duplicating them — extract a shared `render_common_fields(parent, values, config)` if `task_dialog.py`'s implementation makes that natural) for assignee/status/priority/due/tags/custom fields, saving back to the same `Project`

- [ ] **Step 2: Wire card interactions**
  - In `board.py`, a `CardWidget` with `card.kind == "single_project"` on double-click calls `open_project_panel`, not `open_task_dialog`
  - From `project_strip.py`'s click handler (Task 16), single-owner clicks already route here

- [ ] **Step 3: Commit**

```bash
git add src/gui/project_panel.py src/gui/board.py src/gui/task_dialog.py
git commit -m "feat: add single-owner project detail panel with inline checklist"
```

---

## Task 18: Drag and drop

**Files:**
- Create: `src/gui/dragdrop.py`
- Modify: `src/gui/board.py` — bind cards through the drag controller instead of/alongside click handlers

Implements the five-step algorithm from the design doc natively (no third-party DnD lib — `tkinterdnd2` is for OS file drops).

- [ ] **Step 1: `DragController`**
  - `attach(card_widget, card: Card)` binds `<ButtonPress-1>`, `<B1-Motion>`, `<ButtonRelease-1>` on the card
  - Press: record `(x, y)`; do nothing else
  - Motion: if not yet dragging and movement exceeds 6px, start drag — create a borderless `Toplevel` (`overrideredirect(True)`) showing the card's title, positioned at the cursor; dim the source `CardWidget` (e.g. reduced-contrast style). On every motion event, move the ghost window and call `winfo_containing(event.x_root, event.y_root)`, then walk `.master` up until hitting a widget tagged as a status-group drop target or a column header (tag via a widget attribute set by `board.py`, e.g. `widget.drop_role = "status_group"` / `"column_header"`), highlighting/un-highlighting targets as the cursor moves over them
  - Release: resolve the same way; if over a status group, call `on_drop(card, assignee=<that column's person id>, status=<that group's status id>)`; if over a column header only, call `on_drop(card, assignee=<person id>, status=<card's current status>)`; otherwise no-op. Always destroy the ghost window and restore the source widget's style
  - `on_drop` callback is provided by `app.py`: updates the underlying `Task`/single-owner `Project`, calls `save_project`, triggers `reload()`

- [ ] **Step 2: Wire in `board.py`** — when building each `CardWidget`, call `drag_controller.attach(widget, card)` in addition to the click bindings from Tasks 15–17 (a genuine drag — motion past the threshold — must suppress the subsequent click-open so a drag doesn't also open the editor; track this with a flag cleared on release)

- [ ] **Step 3: Commit**

```bash
git add src/gui/dragdrop.py src/gui/board.py
git commit -m "feat: add native drag-and-drop for reassigning and moving cards"
```

- [ ] **Step 4: Manual check** — drag a task to another person's status group; confirm the YAML file, count strip, and rollup strip all update. Drag a single-owner project's card; confirm its own file updates.

---

## Task 19: Settings dialog

**Files:**
- Create: `src/gui/settings_dialog.py`
- Modify: `src/gui/app.py` — wire the "Settings" toolbar button

- [ ] **Step 1: `open_settings_dialog(app) -> None`** — modal `Toplevel` with a `ttk.Notebook`, five tabs. The People / Statuses / Priorities tabs are three instances of the `EditableListPage` from Task 14 plus an in-use guard, so write the guard once as `remove_with_replacement(kind, item_id, in_use_count, candidates) -> replacement_id | None`:
  - **People**: `Listbox` of `config.people`, Add/Rename/Move up/Move down/Remove. Remove, when the person has any assigned cards (check via `counts.cards_for`), opens a small sub-dialog: "Reassign N cards to:" combobox of other people + "Unassigned"; Remove is refused (button stays disabled / shows an inline message) until a target is chosen. On confirm, walk all projects reassigning matching `assignee` fields, save each changed project, then remove the person from config
  - **Statuses**: same list pattern; same in-use guard reassigning to a chosen replacement status; refuse removal if it would leave zero statuses
  - **Priorities**: same list pattern with a caption "Top of the list is the highest priority"; same in-use guard (walk `Task.priority` and single-owner `Project.priority`); refuse removal if it would leave zero priorities. Reordering saves immediately and `app.reload()` re-sorts the board
  - **Custom fields**: list of `FieldDef` with Add/Edit/Remove. Add/Edit opens a small form (name, type dropdown, and a multi-line "choices" box shown only for `type == "choice"`). Remove asks for confirmation, then strips that field's id from every `Task.fields`/`Project.fields` across all projects and saves each changed one. Editing a field's type is out of scope for v1 — the type dropdown is disabled when editing an existing field
  - **Data**: the `DataFolderPicker` from Task 14 (editable entry showing `os.fspath(app.data_dir)`, Browse…, live status) plus an **Apply** button and an **Open folder** button. Apply: `normalize_data_dir`, then if the folder is initialized → `app_prefs.set_data_dir` and reload from it; if it doesn't exist or has no `config.yaml` → ask "Set up a new data folder here?" and run the wizard against it on Yes; if it's unreachable → refuse with the status message. Open folder: `os.startfile` (Windows), `subprocess.run(["open", ...])` (macOS), `subprocess.run(["xdg-open", ...])` (Linux) by `sys.platform`
  - Save on any tab writes `config.yaml` via `store.save_config` and calls `app.reload()`

- [ ] **Step 2: Commit**

```bash
git add src/gui/settings_dialog.py src/gui/app.py
git commit -m "feat: add settings dialog for people, statuses, priorities, custom fields, data folder"
```

- [ ] **Step 3: Manual check** — remove an in-use status without picking a replacement (refused); pick one (all matching cards move); move a priority to the top and confirm the emphasised tag and sort order follow it; add a `choice` custom field and confirm it appears correctly typed in the task editor

---

## Task 20: Export / Import wiring

**Files:**
- Modify: `src/gui/app.py`

- [ ] **Step 1: Export** — "Export…" toolbar button opens `filedialog.asksaveasfilename` defaulting to `backup.default_export_name()` in the user's Documents folder, then calls `backup.export_snapshot(app.data_dir, chosen_path)`; confirmation via a brief status-bar message

- [ ] **Step 2: Import** — "Import…" opens `filedialog.askopenfilename` filtered to `*.zip`, calls `backup.preview_snapshot` into a temp dir; on success shows the `SnapshotSummary` in a confirm dialog ("This snapshot has N people, M statuses, K projects (T tasks). Importing replaces your current data. Continue?"); on confirm calls `backup.import_snapshot` then `app.reload()`. A `BackupError` from preview shows the specific file/reason in an error dialog and does not touch existing data

- [ ] **Step 3: Commit**

```bash
git add src/gui/app.py
git commit -m "feat: wire export/import buttons to backup module"
```

- [ ] **Step 4: Manual check** — Export, delete the data folder, relaunch (wizard appears), Import the snapshot into the fresh folder, confirm the board matches exactly

---

## Task 21: Error handling polish

**Files:**
- Modify: `src/gui/app.py`, `src/gui/board.py`

- [ ] **Step 1:** On `reload()`, if `store.load_all_projects` returns `errors`, show them in the status bar (not a blocking dialog on every reload — only the first time each file is seen as broken in this session) rather than only at startup

- [ ] **Step 2:** A card whose `status` or `assignee` doesn't match any current `Status`/`Person` id (e.g. hand-edited YAML) renders in a synthetic "Unknown" status group / the Unassigned column, styled in the "blocked" colour, per the design doc. Add a `sort_key`/`render` branch in `board.py` for this case

- [ ] **Step 3:** Wrap `store.save_project` / `store.save_config` calls from the GUI in a `try/except OSError` showing the path in an error dialog; keep the in-memory change so the user can retry Save

- [ ] **Step 4: Commit**

```bash
git add src/gui/app.py src/gui/board.py
git commit -m "fix: surface load errors without blocking, handle unknown status/assignee refs"
```

- [ ] **Step 5: Manual check** — hand-edit a project YAML file to set `status: does-not-exist`, relaunch, confirm it appears in the Unknown group with a warning rather than crashing the app

---

## Task 22: Docs and release checklist

**Files:**
- Modify: `docs/CHANGELOG.md`
- Modify: `README.md`
- Modify: `docs/RELEASE-CHECKLIST.md`

- [ ] **Step 1: `docs/CHANGELOG.md`** — add `## 0.2.0 (<today>)` above the 0.1.0 entry:
  - Board: person columns grouped by status, drag-and-drop reassignment, per-person/status count strip, selectable card sort (priority / due / title / created / updated)
  - Projects: single-owner and independent-subtask types, rollup strip, filter-by-project, single-owner detail panel with checklist
  - Configurable people, statuses, priorities, and custom fields via Settings; first-run setup wizard
  - YAML storage (one file per project + config), zip export/import for backup
  - Runs on Linux, macOS, and Windows

- [ ] **Step 2: `README.md`** — add a `0.2.0` row to the version history table; fill in the Key Features section (currently `_TBD_`); expand Requirements to "Python 3.11+ with tkinter — on Debian/Ubuntu install `python3-tk`, on Fedora `python3-tkinter`; python.org installers for macOS and Windows include it"

- [ ] **Step 3: `docs/RELEASE-CHECKLIST.md`** — replace the 0.1.0 checklist with a fresh `0.2.0 (MINOR)` one: one checkbox per CHANGELOG bullet above (mark automated items — models/store/counts/backup are covered by `tests/test.sh`; GUI interaction items are manual), plus the full manual regression list drawn from each task's "Manual check" steps in this plan (setup wizard, drag-and-drop, settings guards, sort selector, export/import roundtrip, malformed-file handling). Add a sub-heading **Per-OS** with a three-column tick grid (Linux / macOS / Windows) for the GUI-interaction items — wizard, drag-and-drop, right-click menu, mouse-wheel scroll, Open folder — since those are where platform differences actually bite. Under Windows only, add: data folder on a UNC share, on a mapped drive, typed as `%OneDrive%\…`; a project named "Con" saves as `con-2.yaml`; disconnect the network with a share-hosted data folder and launch → Retry/Choose/Quit, not the wizard

- [ ] **Step 4: Commit**

```bash
git add docs/CHANGELOG.md README.md docs/RELEASE-CHECKLIST.md
git commit -m "docs: write up 0.2.0 changelog, README, and release checklist"
```

---

## Summary

22 tasks: 5 pure-logic modules with full unit coverage (models, store, counts, backup, app_prefs), then 11 GUI tasks building the window up from a skeleton through board rendering, dialogs, drag-and-drop, settings, and backup wiring, finishing with docs. Each task is independently committable and testable — `tests/test.sh` should pass after every task through Task 10, and the app should be launchable (even if visually incomplete) from Task 11 onward.

Cross-platform is a property of every task, not a task of its own: the input helpers in Task 12 are the only place OS-specific event names appear, `app_prefs.normalize_data_dir` (Task 6) is the only place a user-typed path is interpreted — so drive letters, UNC shares, mapped drives, and env vars are handled once — `pathlib` is used throughout, and the per-OS grid in the release checklist (Task 22) is where the GUI gets verified on each platform.
