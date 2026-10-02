# Week 3: A real database

Last week your Tasks API kept its tasks in a Python dict, and every restart threw them away. This week the same API keeps them in **PostgreSQL** (a database server: a program whose only job is to store data safely on disk and answer questions about it). You also add projects, link tasks to them, and let clients filter and search. Everything here is synchronous, the simplest form; the async version comes later in the internship. Same rules as week 2: work in order, commit after every step (`git add -A && git commit -m "feat: step N"`), type every command yourself, and read every error twice before you search. Written against SQLAlchemy 2.1, Alembic 1.20, psycopg 3.3, PostgreSQL 17.

## What you will have by Friday
- [ ] PostgreSQL 17 running in Docker Compose, with its data in a named volume.
- [ ] `db.py` and `models.py`: SQLAlchemy talks to the database for you.
- [ ] Two Alembic migrations that create the `tasks` and `projects` tables.
- [ ] The five week-2 endpoints reading and writing PostgreSQL. Tasks survive a restart.
- [ ] Three project endpoints, plus `GET /tasks?done=true&q=milk` filtering and search.
- [ ] Four passing tests against the real database, and `ruff` clean.

## Before you start
```bash
docker --version && docker compose version   # Docker 27 or newer, Compose v2
cd ~/projects/tasks-api && uv run pytest     # week 2 still passes: 2 passed
git status                                   # clean: everything from week 2 is committed
```

### Step 1: Start PostgreSQL
Goal: a database server runs on your laptop. You wrote a Compose file in week 1; this one is smaller. `postgres:17` is the official image, the `healthcheck` asks PostgreSQL every five seconds if it is ready, and the named volume `pgdata` keeps the data when the container is removed. The password is `secret` because this database only runs on your laptop. Create `compose.yaml` in the project folder:
```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: tasks
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d tasks"]
      interval: 5s
      retries: 10

volumes:
  pgdata:
```
Your code needs to know where the database is. A **connection string** is one line with everything needed to connect: driver (`postgresql+psycopg`), user, password, host, port and database name. You put it in an **environment variable** (a named value the operating system hands to a program), and the variable lives in `.env`. `.env` holds secrets, so git must never see it; `.env.example` is the committed copy that shows teammates what to fill in.
```bash
echo 'DATABASE_URL=postgresql+psycopg://app:secret@localhost:5432/tasks' > .env
cp .env .env.example
echo '.env' >> .gitignore
docker compose up -d && docker compose ps
```
Check: `docker compose ps` shows `tasks-api-db-1` with `Up ... (healthy)`. If it says `(health: starting)`, wait five seconds and run it again. Commit.

### Step 2: Talk SQL to it
Goal: see a table with your own eyes before any Python touches it. Before you type this, understand: a **table** is like one sheet of a spreadsheet with a fixed set of **columns** (each has a name and a type, like `title text`); every **row** is one record, here one task. The **primary key** is the column that identifies a row; `id serial PRIMARY KEY` means the database picks the next free number itself. **SQL** is the language for asking a database for data or changing it. `psql` is the SQL terminal that ships with PostgreSQL; `\dt` lists tables and `\d tasks` describes one (these two are `psql` commands, not SQL, so no `;`). You drop the table at the end because from Step 4 your Python code owns the tables, not your fingers. Run `docker compose exec db psql -U app -d tasks`; the prompt changes to `tasks=#`. Every SQL statement ends with `;`. Type these one by one and read every answer:
```sql
CREATE TABLE tasks (id serial PRIMARY KEY, title text NOT NULL, done boolean NOT NULL DEFAULT false);
INSERT INTO tasks (title) VALUES ('Buy milk'), ('Learn SQL');
SELECT * FROM tasks;
UPDATE tasks SET done = true WHERE id = 1;
DELETE FROM tasks WHERE id = 2;
SELECT * FROM tasks;
\dt
\d tasks
DROP TABLE tasks;
\q
```
Check: the second `SELECT` shows one row, `Buy milk`, with `done` = `t`. `\d tasks` shows three columns and `"tasks_pkey" PRIMARY KEY`. Nothing to commit.

### Step 3: SQLAlchemy
Goal: Python classes that mirror tables. Before you type this, understand: an **ORM** (object-relational mapper) is a library that turns rows into Python objects and back, so you write Python instead of SQL strings. **SQLAlchemy** is our ORM. Its **engine** is the object that holds the connection string and opens connections. A **session** is one short conversation with the database: you add or change objects, then `commit()` saves everything at once. **psycopg** is the driver, the low-level piece that speaks PostgreSQL's network language. `python-dotenv` loads `.env` into environment variables, and `os.environ["DATABASE_URL"]` reads one and crashes early if it is missing, which is what you want. Alembic is for Step 4. Run `uv add "sqlalchemy" "psycopg[binary]" python-dotenv alembic`, then create `db.py`:
```python
import os
from collections.abc import Iterator

from dotenv import load_dotenv
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker

load_dotenv()
engine = create_engine(os.environ["DATABASE_URL"])
SessionLocal = sessionmaker(engine)


class Base(DeclarativeBase):
    pass


def get_db() -> Iterator[Session]:
    with SessionLocal() as db:
        yield db
```
`SessionLocal()` makes a new session. `get_db` opens one, hands it out with `yield`, and closes it when the request is over; Step 5 plugs it into FastAPI. `Base` is the class every table class starts from: one class is one table, one attribute is one column. `Mapped[int]` gives the Python type and `mapped_column(...)` adds database details. The class is called `TaskRow`, not `Task`, because `Task` is already your Pydantic model for JSON, and the two have different jobs: one shape for the database, one shape for the client. Create `models.py`:
```python
from sqlalchemy import String
from sqlalchemy.orm import Mapped, mapped_column

from db import Base


class TaskRow(Base):
    __tablename__ = "tasks"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    done: Mapped[bool] = mapped_column(default=False)
```
Check: `uv run python -c "import models; print(list(models.Base.metadata.tables))"` prints `['tasks']`. Commit.

### Step 4: Alembic creates the table
Goal: the `tasks` table exists again, built from `models.py`. Before you type this, understand: a **migration** is a small script that changes the database structure (create a table, add a column) and lives in git next to your code. Everyone who runs `alembic upgrade head` gets the same tables in the same order, on a laptop or on a server. **Alembic** writes most of these scripts for you by comparing your models with the real database; this is called autogenerate. The first command below makes `alembic.ini` and a `migrations/` folder; the second tells `ruff` to skip that folder, because Alembic's generated files have their own style.
```bash
uv run alembic init migrations
printf '\n[tool.ruff]\nextend-exclude = ["migrations"]\n' >> pyproject.toml
```
Open `migrations/env.py`, find the line `target_metadata = None` and replace it with these four lines. `Base.metadata` is the list of every table your models describe. The URL comes from the environment (`models` imports `db`, which already loaded `.env`), so no password lands in `alembic.ini`.
```python
import os
from models import Base
config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
target_metadata = Base.metadata
```
Generate the first migration, run it, and look at the result:
```bash
uv run alembic revision --autogenerate -m "create tasks"
uv run alembic upgrade head
docker compose exec db psql -U app -d tasks -c '\d tasks' -c 'SELECT * FROM alembic_version'
```
Check: open the new file in `migrations/versions/`: it has `op.create_table('tasks', ...)` with three columns. `\d tasks` shows the table again, now with `character varying(200)`. `alembic_version` has one row: that is how Alembic remembers which migrations already ran. Commit, including the `migrations/` folder.

### Step 5: Create and list from the database
Goal: `POST /tasks` writes a row, `GET /tasks` reads rows, and a restart loses nothing. Before you type this, understand: `Depends(get_db)` is FastAPI's way to say "before this function runs, call `get_db` and give me what it yields"; `Annotated[Session, Depends(get_db)]` packs the type and that instruction into one name, `DbSession`, so you write it once. Replace the two import lines at the top of `main.py` with this block (keep two empty lines between it and `class TaskCreate`):
```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import select
from sqlalchemy.orm import Session

from db import get_db
from models import TaskRow

DbSession = Annotated[Session, Depends(get_db)]
```
`Task.model_validate(row)` builds the Pydantic `Task` from a `TaskRow` object; `from_attributes=True` allows reading from an object's attributes instead of a dict. `db.add(row)` tells the session about the new object, `db.commit()` runs the `INSERT` and saves it for good, and `db.refresh(row)` reads the row back so `row.id`, chosen by the database, is filled in. `select(TaskRow)` is the Python form of `SELECT * FROM tasks`, and `db.scalars(...)` runs it and hands you `TaskRow` objects. Three edits: add `model_config = ConfigDict(from_attributes=True)` as the first line inside `class Task` (leave one empty line before `id: int`); move `class TaskUpdate` up so it sits directly below `class Task`; and replace `create_task` and `list_tasks` with:
```python
@app.post("/tasks", status_code=201)
def create_task(body: TaskCreate, db: DbSession) -> Task:
    row = TaskRow(**body.model_dump())
    db.add(row)
    db.commit()
    db.refresh(row)
    return Task.model_validate(row)


@app.get("/tasks")
def list_tasks(db: DbSession) -> list[Task]:
    rows = db.scalars(select(TaskRow).order_by(TaskRow.id))
    return [Task.model_validate(row) for row in rows]
```
Get, update and delete still use the dict; Step 6 fixes them. Start the server, create two tasks, then stop the server with `Ctrl+C` and start it again:
```bash
uv run fastapi dev main.py
curl -s -X POST localhost:8000/tasks -H 'content-type: application/json' -d '{"title":"Buy milk"}'
curl -s -X POST localhost:8000/tasks -H 'content-type: application/json' -d '{"title":"Learn SQL","done":true}'
curl -s localhost:8000/tasks
```
Check: after the restart, `curl -s localhost:8000/tasks` still returns both tasks. Last week this list came back empty. Commit.

### Step 6: Get, update and delete
Goal: every endpoint uses the database, and the dict is gone. `db.get(TaskRow, task_id)` fetches one row by primary key, or `None`. `setattr(row, "done", True)` is `row.done = True` with the attribute name coming from a variable, so one loop handles whatever fields the client sent. `db.delete(row)` marks a row for deletion and `db.commit()` runs the `DELETE`. `find_task` is a plain helper, not an endpoint: it returns the `TaskRow` so `update_task` can change it, and raises the `404` in one place. Delete the lines `tasks: dict[int, Task] = {}` and `next_id = 1`, then replace `get_task`, `update_task` and `delete_task` with:
```python
def find_task(db: Session, task_id: int) -> TaskRow:
    row = db.get(TaskRow, task_id)
    if row is None:
        raise HTTPException(status_code=404, detail="task not found")
    return row


@app.get("/tasks/{task_id}")
def get_task(task_id: int, db: DbSession) -> Task:
    return Task.model_validate(find_task(db, task_id))


@app.patch("/tasks/{task_id}")
def update_task(task_id: int, body: TaskUpdate, db: DbSession) -> Task:
    row = find_task(db, task_id)
    for name, value in body.model_dump(exclude_unset=True).items():
        setattr(row, name, value)
    db.commit()
    db.refresh(row)
    return Task.model_validate(row)


@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int, db: DbSession) -> None:
    db.delete(find_task(db, task_id))
    db.commit()
```
Check: run the three `curl` commands from week 2, Step 6. `PATCH` returns the task with `done` true, the first `DELETE` returns `204`, the second `404`. `grep -c next_id main.py` prints `0`. `uv run ruff check .` passes. Commit.

### Step 7: A second table
Goal: projects exist, a task may belong to one, and you can list a project's tasks. Before you type this, understand: a **foreign key** is a column whose value must be the primary key of a row in another table; the database refuses a `project_id` that points to no project. This links the tables in a **relationship**: one project has many tasks and each task has at most one project, so it is called one-to-many. `Mapped[int | None]` makes the column nullable: a task without a project is allowed. `name="fk_tasks_project"` names the rule so a later migration can find it. An **index** is a sorted copy of one column that the database keeps so that searching by that column is fast; `index=True` adds one on `project_id`. `relationship(...)` is Python-side only: it gives you `project.tasks` as a list and `task.project` as an object, and SQLAlchemy runs the `SELECT` when you touch them. The quotes in `"ProjectRow | None"` let you name a class that Python has not read yet. Change the two import lines of `models.py` to `from sqlalchemy import ForeignKey, String` and `from sqlalchemy.orm import Mapped, mapped_column, relationship`, then add this at the end of the file; the first four lines go inside `TaskRow`, right after `done`, indented as shown:
```python
    project_id: Mapped[int | None] = mapped_column(
        ForeignKey("projects.id", name="fk_tasks_project"), index=True
    )
    project: Mapped["ProjectRow | None"] = relationship(back_populates="tasks")


class ProjectRow(Base):
    __tablename__ = "projects"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    tasks: Mapped[list["TaskRow"]] = relationship(back_populates="project")
```
Run `uv run alembic revision --autogenerate -m "add projects"`, read the new file, then `uv run alembic upgrade head`. Now `main.py`: change the import to `from models import ProjectRow, TaskRow`; add `project_id: int | None = None` as the last field of `TaskCreate`; in `create_task`, add two lines before `row = TaskRow(...)`: `if body.project_id is not None:` and, indented under it, `find_project(db, body.project_id)`. Then add this at the end of the file:
```python
class ProjectCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)


class Project(ProjectCreate):
    model_config = ConfigDict(from_attributes=True)

    id: int


def find_project(db: Session, project_id: int) -> ProjectRow:
    row = db.get(ProjectRow, project_id)
    if row is None:
        raise HTTPException(status_code=404, detail="project not found")
    return row


@app.post("/projects", status_code=201)
def create_project(body: ProjectCreate, db: DbSession) -> Project:
    row = ProjectRow(**body.model_dump())
    db.add(row)
    db.commit()
    db.refresh(row)
    return Project.model_validate(row)


@app.get("/projects")
def list_projects(db: DbSession) -> list[Project]:
    rows = db.scalars(select(ProjectRow).order_by(ProjectRow.id))
    return [Project.model_validate(row) for row in rows]


@app.get("/projects/{project_id}/tasks")
def list_project_tasks(project_id: int, db: DbSession) -> list[Task]:
    project = find_project(db, project_id)
    return [Task.model_validate(row) for row in project.tasks]
```
```bash
curl -s -X POST localhost:8000/projects -H 'content-type: application/json' -d '{"name":"Home"}'
curl -s -X POST localhost:8000/tasks -H 'content-type: application/json' -d '{"title":"Clean kitchen","project_id":1}'
curl -i -X POST localhost:8000/tasks -H 'content-type: application/json' -d '{"title":"Ghost","project_id":999}'
curl -s localhost:8000/projects/1/tasks
```
Check: the migration file has `op.create_table('projects', ...)`, `op.add_column(...)` and `op.create_foreign_key('fk_tasks_project', ...)`. The project comes back with `"id":1`, the task with `"project_id":1`, the ghost task gets `404` with `{"detail":"project not found"}`, and the last command lists only `Clean kitchen`. Commit.

### Step 8: Filter and search
Goal: `GET /tasks?done=true&q=milk` returns only done tasks whose title contains "milk". Before you type this, understand: a **query parameter** is a `name=value` pair after the `?` in a URL; several are joined by `&`. In FastAPI, any function parameter that is not in the path becomes a query parameter, and `bool | None = None` makes it optional. You build the `select` step by step: each `.where(...)` adds a condition, like `WHERE` in SQL. `ilike` is a text match that ignores upper and lower case, and `%` means "anything here". Replace `list_tasks`:
```python
@app.get("/tasks")
def list_tasks(
    db: DbSession, done: bool | None = None, q: str | None = None
) -> list[Task]:
    stmt = select(TaskRow).order_by(TaskRow.id)
    if done is not None:
        stmt = stmt.where(TaskRow.done == done)
    if q is not None:
        stmt = stmt.where(TaskRow.title.ilike(f"%{q}%"))
    return [Task.model_validate(row) for row in db.scalars(stmt)]
```
```bash
curl -s 'localhost:8000/tasks?done=true'
curl -s 'localhost:8000/tasks?q=KITCHEN'
curl -i 'localhost:8000/tasks?done=maybe'
```
Check: the first returns only `Learn SQL`, the second returns `Clean kitchen` (capital letters do not matter), the third returns `422`. Open http://127.0.0.1:8000/docs: `GET /tasks` now shows two optional parameters. Commit.

### Step 9: Tests against the real database
Goal: `uv run pytest` proves everything above, and `ruff` is clean. The tests use the same database as the server (a separate test database comes later in the internship), so every test must start from empty tables. A pytest **fixture** is a function that runs before a test; `autouse=True` runs it before every test without being asked. It deletes tasks first, then projects, because a task points to a project and the database refuses to delete a project that still has tasks. The two week-2 tests stay, with one new line in the first; two tests are new. Replace `test_main.py`:
```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import delete

from db import SessionLocal
from main import app
from models import ProjectRow, TaskRow

client = TestClient(app)


@pytest.fixture(autouse=True)
def clean_tables() -> None:
    with SessionLocal() as db:
        db.execute(delete(TaskRow))
        db.execute(delete(ProjectRow))
        db.commit()


def test_create_then_get() -> None:
    r = client.post("/tasks", json={"title": "Learn FastAPI"})
    assert r.status_code == 201
    task_id = r.json()["id"]
    r = client.get(f"/tasks/{task_id}")
    assert r.status_code == 200
    assert r.json()["title"] == "Learn FastAPI"
    assert len(client.get("/tasks").json()) == 1


def test_get_missing_is_404() -> None:
    assert client.get("/tasks/999").status_code == 404


def test_project_lists_only_its_tasks() -> None:
    project_id = client.post("/projects", json={"name": "Home"}).json()["id"]
    client.post("/tasks", json={"title": "Buy milk", "project_id": project_id})
    client.post("/tasks", json={"title": "No project"})
    r = client.get(f"/projects/{project_id}/tasks")
    assert [t["title"] for t in r.json()] == ["Buy milk"]


def test_filter_by_done() -> None:
    client.post("/tasks", json={"title": "Done one", "done": True})
    client.post("/tasks", json={"title": "Open one"})
    r = client.get("/tasks", params={"done": True})
    assert [t["title"] for t in r.json()] == ["Done one"]
```
Check: `uv run pytest` prints `4 passed`, and a second run too, because the fixture cleans up. `uv run ruff format .` then `uv run ruff check .` prints `All checks passed!`. Commit.

## Read and watch
1. [ORM Quick Start](https://docs.sqlalchemy.org/en/20/orm/quickstart.html) (SQLAlchemy docs): the same `Base`, `Mapped`, `select` and `Session` you type in Steps 3 to 6; read it before Step 3.
2. [Session Basics](https://docs.sqlalchemy.org/en/20/orm/session_basics.html) (SQLAlchemy docs): what `add`, `commit` and `refresh` really do; read only the part "Basics of Using a Session".
3. [Alembic Tutorial](https://alembic.sqlalchemy.org/en/latest/tutorial.html) (Alembic docs): the `init`, `revision` and `upgrade` commands of Step 4; stop before "Relative Migration Identifiers".
4. Video: [TUTORIAL: SQLAlchemy 2.0](https://www.youtube.com/watch?v=Uym2DHnUEno) (Mike Bayer, the author of SQLAlchemy, PyCon US 2023): watch the first 50 minutes, the ORM part, over two evenings; everything he types works with the 2.1 you installed.

## Turn in
- Push `tasks-api` with one commit per step. Open the repository on GitHub and confirm `.env` is not there, but `.env.example` and `migrations/` are.
- Add a section "Run the database" to your `README.md`: `docker compose up -d`, then `uv run alembic upgrade head`.
- Add `notes/week-03.md` with your answers to the self-check below, in your own words.
- Friday demo (10 minutes): fresh clone, `docker compose up -d`, `uv sync`, `uv run alembic upgrade head`, start the server, create a project and two tasks, restart the server and show the tasks are still there, show `?done=true` and `?q=`, show one `404` for an unknown project, run `uv run pytest`.

## Self-check
1. Why do your tasks survive a server restart this week, when last week they did not?
2. What is a migration, and why not create tables by hand in `psql` like in Step 2?
3. `project_id` is a nullable foreign key. What do "nullable" and "foreign key" each mean for a task?
4. Why does the test fixture delete tasks before projects?
5. What does `db.commit()` do, and what happens to your `INSERT` if you forget it?

### Answers
1. They live in PostgreSQL, which writes them to disk inside the `pgdata` volume; the dict lived in the memory of the server process.
2. A script in git that changes the database structure; `alembic upgrade head` gives everyone the same tables, while hand-made tables exist only on your laptop.
3. Nullable: a task may have no project. Foreign key: if it has one, the value must be the `id` of a real row in `projects`, or the database refuses it.
4. Tasks point to projects; deleting a project that still has tasks would break the foreign key, so PostgreSQL refuses.
5. It runs the waiting SQL and saves it permanently; without it the session closes at the end of the request and the change is thrown away.
