# Week 2: Your first web API

---

This week you build a small **Tasks API**. An API (Application Programming Interface) is a program that other programs talk to over the network. Yours will store tasks in memory and let a client create, list, read, update and delete them. It will check the input, return clear error messages, show automatic documentation, and have tests.

The content (tasks) is not important. The structure is. In week 3 you swap the memory storage for a real database. In week 4 you add login. In weeks 5 to 7 you build a web page that talks to this API. Your capstone project copies the skeleton you build this week. So build it carefully.

A note from your supervisor: type every command yourself. Read each error message twice before you search. When you ask for help, bring the error, what you tried, and what you expected.

## What you will have by Friday

- [ ] A project `tasks-api` created with `uv`, running with one command: `uv run fastapi dev src/tasks_api/main.py`.
- [ ] Five endpoints: `POST`, `GET` (list), `GET` (one), `PATCH`, `DELETE` under `/api/v1/tasks`.
- [ ] Input validation with Pydantic: bad input returns `422`, unknown fields are rejected.
- [ ] One error format for every error, with the media type `application/problem+json`.
- [ ] Pagination with `limit` and `offset`, and a response shape `{items, limit, offset, total}`.
- [ ] Code split into files, four passing tests (`uv run pytest`), clean lint (`uv run ruff check .`), and a short `README.md`.

## Before you start

Confirm your week-1 tools still work. Each command must print a version or a path, not an error.

```bash
uv --version            # 0.9 or newer
uv python find 3.12     # prints a path to Python 3.12
git --version && mkdir -p ~/projects && cd ~/projects
```

## The flow

Work through the steps in order. Each step builds on the one before. Commit after every step (`git add -A && git commit -m "feat: step N"`).

### Step 1: Create the project and run "hello"

Goal: a web server that answers one request.

Before you type this, understand: `uv` creates the project folder, a virtual environment (a private folder of installed packages) and a `pyproject.toml` (the file that lists your dependencies). FastAPI is the library we use to build the API. `fastapi[standard]` also installs `uvicorn`, the server that runs your code.

```bash
uv init --package tasks-api --python 3.12
cd tasks-api
uv add "fastapi[standard]" pydantic-settings
uv add --dev pytest anyio httpx ruff
```

Create the file `src/tasks_api/main.py`:

```python
from fastapi import FastAPI

app = FastAPI(title="Tasks API")


@app.get("/healthz")
async def healthz() -> dict[str, str]:
    return {"status": "ok"}
```

Read it line by line. `app` is your application. `@app.get("/healthz")` says: when a client asks for the path `/healthz`, run the function below. The function returns a Python dict; FastAPI turns it into JSON. Ignore `async` for now; Step 11 explains it.

Start the server. Leave this terminal open for the whole week and open a second one for commands.

```bash
uv run fastapi dev src/tasks_api/main.py
```

Check: open http://127.0.0.1:8000/healthz in your browser. You see `{"status":"ok"}`. Commit.

### Step 2: Understand HTTP and test the endpoint

Goal: know what a request and a response are, and see them with `curl`.

Before you type this, understand: a client sends a **request**. A request has a **method** (`GET` = read, `POST` = create, `PATCH` = change part, `DELETE` = remove), a **path** (`/healthz`), **headers** (small key-value lines, like `content-type`), and sometimes a **body** (the data, in JSON). The server answers with a **response**: a **status code** (a number: `2xx` = success, `4xx` = the client made a mistake, `5xx` = the server made a mistake), headers, and a body. An **endpoint** is one path plus one method that your API handles.

```bash
curl -i http://127.0.0.1:8000/healthz
```

`-i` shows the response headers. Look at the first line (`HTTP/1.1 200 OK`) and the `content-type` header.

Now open http://127.0.0.1:8000/docs. This page is generated from your code. It lists every endpoint and lets you call it from the browser. It is called Swagger UI, and the data behind it is called OpenAPI. You get it for free with FastAPI, as long as you use type hints (Step 3).

Check: `curl -i` shows `200 OK` and `content-type: application/json`. The `/docs` page shows `GET /healthz`.

### Step 3: Type hints and the Task model

Goal: describe what a task looks like and what input is allowed.

Before you type this, understand: a **type hint** tells Python what kind of value a variable holds (`title: str`). Python itself ignores hints. FastAPI and Pydantic read them. **Pydantic** is a library that turns a class with type hints into a **model**: an object that checks (validates) incoming data and converts it to JSON. You write two models: `TaskCreate` for input (what the client may send) and `TaskRead` for output (what you send back). They are different because the client does not choose the `id` or `created_at`.

Add this to `src/tasks_api/main.py`, above `app = FastAPI(...)`:

```python
from datetime import datetime
from enum import StrEnum
from typing import Annotated
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field, field_validator


class TaskStatus(StrEnum):
    todo = "todo"
    in_progress = "in_progress"
    done = "done"


TitleStr = Annotated[str, Field(min_length=1, max_length=200)]


def normalize_tags(v: list[str] | None) -> list[str] | None:
    """Trim spaces, make lowercase, remove duplicates, keep the order."""
    if v is None:
        return None
    return list(dict.fromkeys(t.strip().lower() for t in v if t.strip()))


class TaskCreate(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, extra="forbid")

    title: TitleStr
    description: str | None = Field(default=None, max_length=2000)
    status: TaskStatus = TaskStatus.todo
    tags: list[str] = Field(default_factory=list, max_length=10)

    _tags = field_validator("tags")(normalize_tags)


class TaskRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    title: str
    description: str | None
    status: TaskStatus
    tags: list[str]
    created_at: datetime
    updated_at: datetime
```

What each new thing means:

- `StrEnum`: a fixed list of allowed strings. A status that is not in the list is rejected.
- `Annotated[str, Field(...)]`: a `str` with extra rules. We name it `TitleStr` so we can reuse it.
- `str | None = None`: the field is optional. `None` means "no value".
- `extra="forbid"`: if the client sends a field you did not define (for example a typo `titel`), reject the request. `str_strip_whitespace=True` removes spaces around every string.
- `field_validator("tags")(normalize_tags)`: run `normalize_tags` on the `tags` field after the basic checks.
- `from_attributes=True`: lets `TaskRead` be built from a plain Python object (Step 4), not only from a dict. `UUID` is a long random id that clients cannot guess.

Check: run this in the second terminal. It prints a clean title (`'Hi'`) and one tag (`['a']`). Then change `tags=` to `titel=` and run again: you get a `ValidationError` that says `Extra inputs are not permitted`.

```bash
uv run python -c "from tasks_api.main import TaskCreate; print(TaskCreate(title='  Hi ', tags=['A', 'a']).model_dump())"
```

### Step 4: In-memory storage and the create endpoint

Goal: `POST /api/v1/tasks` creates a task and returns it with status `201`.

Before you type this, understand: "in memory" means the tasks live in a Python dict inside the server process. When the server stops, the tasks are gone. That is fine this week. A **dataclass** is a simple Python class with typed fields; we use it for the stored object because the data comes from our own code, not from the client. Pydantic is for data that crosses the border (request and response). The dataclass is for the inside.

Add to `main.py`, below the models. This is the whole store; Steps 5 to 7 use the other methods.

```python
from dataclasses import dataclass, field, replace
from datetime import UTC
from uuid import uuid4


def utcnow() -> datetime:
    return datetime.now(UTC)


@dataclass
class Task:
    title: str
    description: str | None = None
    status: TaskStatus = TaskStatus.todo
    tags: list[str] = field(default_factory=list)
    id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=utcnow)
    updated_at: datetime = field(default_factory=utcnow)


class TaskStore:
    def __init__(self) -> None:
        self._items: dict[UUID, Task] = {}

    def add(self, task: Task) -> Task:
        self._items[task.id] = task
        return task

    def get(self, task_id: UUID) -> Task | None:
        return self._items.get(task_id)

    def list(self, limit: int, offset: int) -> tuple[list[Task], int]:
        rows = list(self._items.values())
        return rows[offset : offset + limit], len(rows)

    def update(self, task_id: UUID, changes: dict[str, object]) -> Task | None:
        current = self._items.get(task_id)
        if current is None:
            return None
        updated = replace(current, **changes, updated_at=utcnow())  # a changed copy
        self._items[task_id] = updated
        return updated

    def delete(self, task_id: UUID) -> bool:
        return self._items.pop(task_id, None) is not None


store = TaskStore()
```

Add the endpoint below `healthz`:

```python
@app.post("/api/v1/tasks", status_code=201)
async def create_task(body: TaskCreate) -> TaskRead:
    task = store.add(Task(**body.model_dump()))
    return TaskRead.model_validate(task)
```

How it works: `body: TaskCreate` tells FastAPI "read the JSON body and validate it as `TaskCreate`". If validation fails, FastAPI answers `422` and your function never runs. `body.model_dump()` gives a dict; `Task(**...)` builds the stored object. `TaskRead.model_validate(task)` builds the output model. `status_code=201` means "Created", the correct code for a successful `POST`. `/api/v1` is a version prefix: if you change the API in a breaking way one day, you make `/api/v2` and old clients keep working.

Check: the response starts with `HTTP/1.1 201 Created`, has an `id`, and `tags` is `["docs"]` (one tag, lowercase). Keep the `id` for the next step.

```bash
curl -i -X POST localhost:8000/api/v1/tasks -H 'content-type: application/json' \
  -d '{"title":"Learn FastAPI","tags":["Docs","docs"]}'
```

### Step 5: List and get one, with 404

Goal: `GET /api/v1/tasks` returns the tasks; `GET /api/v1/tasks/{id}` returns one, or `404` if it does not exist.

Before you type this, understand: `{task_id}` in a path is a **path parameter**. FastAPI reads it from the URL and converts it to the type you declare (`UUID`). A wrong format gives `422` automatically. `404 Not Found` is the status for "this id does not exist". `HTTPException` is FastAPI's way to stop a request with a status code. We put it in a small helper because three endpoints need it.

Add `HTTPException` to the `from fastapi import ...` line, then add below `create_task`:

```python
def not_found(task_id: UUID) -> HTTPException:
    return HTTPException(status_code=404, detail=f"task {task_id} does not exist")


@app.get("/api/v1/tasks")
async def list_tasks() -> list[TaskRead]:
    rows, _total = store.list(limit=100, offset=0)  # Step 7 lets the client choose
    return [TaskRead.model_validate(t) for t in rows]


@app.get("/api/v1/tasks/{task_id}")
async def get_task(task_id: UUID) -> TaskRead:
    task = store.get(task_id)
    if task is None:
        raise not_found(task_id)
    return TaskRead.model_validate(task)
```

Check: the list shows your task in a JSON array. The second command (use your real id) returns `200`. The third returns `404` with `{"detail":"task ... does not exist"}`.

```bash
curl -s localhost:8000/api/v1/tasks
curl -i localhost:8000/api/v1/tasks/PASTE-YOUR-ID-HERE
curl -i localhost:8000/api/v1/tasks/00000000-0000-0000-0000-000000000000
```

### Step 6: Update and delete

Goal: `PATCH` changes only the fields the client sends; `DELETE` removes a task and returns `204`.

Before you type this, understand: in a `PATCH` body every field is optional. There is a difference between "the client did not send `description`" (keep the old value) and "the client sent `description: null`" (clear the value). `model_dump(exclude_unset=True)` gives you only the fields that were really sent, so both cases work. `204 No Content` is the status for a successful delete: no body is returned.

Add a third model below `TaskCreate`:

```python
class TaskUpdate(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, extra="forbid")

    title: TitleStr | None = None
    description: str | None = Field(default=None, max_length=2000)
    status: TaskStatus | None = None
    tags: list[str] | None = Field(default=None, max_length=10)

    _tags = field_validator("tags")(normalize_tags)
```

Add two endpoints below `get_task`:

```python
@app.patch("/api/v1/tasks/{task_id}")
async def update_task(task_id: UUID, body: TaskUpdate) -> TaskRead:
    task = store.update(task_id, body.model_dump(exclude_unset=True))
    if task is None:
        raise not_found(task_id)
    return TaskRead.model_validate(task)


@app.delete("/api/v1/tasks/{task_id}", status_code=204)
async def delete_task(task_id: UUID) -> None:
    if not store.delete(task_id):
        raise not_found(task_id)
```

Check: after `PATCH`, `status` is `"done"`, `title` is unchanged, and `updated_at` is newer than `created_at`. The first `DELETE` returns `204`, the second returns `404`.

```bash
curl -s -X PATCH localhost:8000/api/v1/tasks/YOUR-ID -H 'content-type: application/json' -d '{"status":"done"}'
curl -i -X DELETE localhost:8000/api/v1/tasks/YOUR-ID
curl -i -X DELETE localhost:8000/api/v1/tasks/YOUR-ID
```

### Step 7: Pagination

Goal: the list endpoint returns one page at a time, with a fixed response shape.

Before you type this, understand: a real list can have thousands of items. **Pagination** means the client asks for a slice: `limit` (how many) and `offset` (skip how many). These come from the **query string**, the part of the URL after `?`, for example `/api/v1/tasks?limit=20&offset=40`. Always cap `limit`; an uncapped limit lets one client ask for a million rows. The response is an **envelope**: an object with `items` plus the numbers the client needs to ask for the next page.

Add a model below `TaskRead`:

```python
class Page(BaseModel):
    items: list[TaskRead]
    limit: int
    offset: int
    total: int
```

Replace `list_tasks`. Add `Query` to the `from fastapi import ...` line. `Query(ge=1, le=100)` means "greater or equal to 1, less or equal to 100"; out of range gives `422`.

```python
@app.get("/api/v1/tasks")
async def list_tasks(
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> Page:
    rows, total = store.list(limit=limit, offset=offset)
    items = [TaskRead.model_validate(t) for t in rows]
    return Page(items=items, limit=limit, offset=offset, total=total)
```

Check: create three tasks with `POST`, then run the first command. You see `total: 3` and two items. The second command returns `422`.

```bash
curl -s 'localhost:8000/api/v1/tasks?limit=2&offset=0'
curl -i 'localhost:8000/api/v1/tasks?limit=1000'
```

### Step 8: One clean error format

Goal: every error (404, 422, ...) has the same JSON shape.

Before you type this, understand: right now a `404` returns `{"detail": ...}` and a `422` returns a different shape. A client (and your future web page) wants one shape. We use a small version of the standard called RFC 9457 "Problem Details": fields `title`, `status`, `instance` (the path that failed), and when useful `detail` and a list `errors`. The media type is `application/problem+json`. An **exception handler** is a function FastAPI calls when a certain exception is raised, so you build the response yourself.

Which status codes to use this week: `200` OK, `201` created, `204` deleted, `404` not found, `422` the input is wrong. Never return `200` with an error inside the body.

Add `Request` to the `from fastapi import ...` line. Add this above `app = FastAPI(...)`. The handlers live inside a function so we can move them to their own file in Step 9.

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException


def problem(request: Request, status: int, title: str, **extra: object) -> JSONResponse:
    body: dict[str, object] = {"title": title, "status": status, "instance": request.url.path}
    body.update(extra)
    return JSONResponse(content=body, status_code=status, media_type="application/problem+json")


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(StarletteHTTPException)
    async def http_error_handler(request: Request, exc: StarletteHTTPException) -> JSONResponse:
        return problem(request, exc.status_code, str(exc.detail))

    @app.exception_handler(RequestValidationError)
    async def validation_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
        errors = [{"loc": list(e["loc"]), "msg": e["msg"]} for e in exc.errors()]
        detail = f"{len(errors)} field(s) failed validation"
        return problem(request, 422, "Request validation failed", detail=detail, errors=errors)
```

Right after `app = FastAPI(...)`, add the line `register_exception_handlers(app)`. FastAPI's `HTTPException` is a child of Starlette's, so the first handler catches your `404`s. The second catches every validation failure.

Check: the `404` now has `content-type: application/problem+json` and the body has `title`, `status`, `instance`. The typo request returns `422` with an `errors` list that mentions `titel`.

```bash
curl -i localhost:8000/api/v1/tasks/00000000-0000-0000-0000-000000000000
curl -s -X POST localhost:8000/api/v1/tasks -H 'content-type: application/json' -d '{"titel":"x"}'
```

### Step 9: Split into files and add settings

Goal: the same API, organized so it can grow, with configuration read from the environment.

Before you type this, understand: one big file is fine for 100 lines, not for 1,000. We split by role: `schemas.py` (Pydantic models), `storage.py` (the dataclass and the store), `errors.py` (handlers), `settings.py` (configuration), `routers/tasks.py` (the endpoints), `main.py` (puts it together). A **router** is a group of endpoints with a shared prefix. **Settings** are values that change between machines (the app name, the path prefix) and are read from environment variables or a `.env` file, as in the 12-factor rule from week 1. A **dependency** (`Depends`) is a small function FastAPI runs before your endpoint to give it something it needs, here the store. We keep the store on `app.state` instead of a global variable, and build the app in a function `create_app()`, so each test gets a fresh app with an empty store. `lifespan` runs code once at startup (before `yield`) and once at shutdown (after `yield`).

Create the folder and move code. Keep the dev server running; it restarts on every save and shows import errors.

```bash
mkdir -p src/tasks_api/routers && touch src/tasks_api/routers/__init__.py
```

- `src/tasks_api/schemas.py`: cut `TaskStatus`, `TitleStr`, `normalize_tags`, `TaskCreate`, `TaskUpdate`, `TaskRead`, `Page` and their imports from `main.py` and paste them here. No changes.
- `src/tasks_api/storage.py`: cut `utcnow`, `Task`, `TaskStore` and their imports. Add `from tasks_api.schemas import TaskStatus`. Delete the line `store = TaskStore()`.
- `src/tasks_api/errors.py`: cut `problem`, `register_exception_handlers` and their imports. Add `from fastapi import FastAPI, Request`.

`src/tasks_api/settings.py` (new). An environment variable `TASKS_APP_NAME="My API"` overrides `app_name`. Create `.env.example` with the line `TASKS_APP_NAME=Tasks API`; never commit a real `.env`.

```python
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="TASKS_", extra="ignore")

    app_name: str = "Tasks API"
    api_prefix: str = "/api/v1"


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`src/tasks_api/routers/tasks.py` (new). Start with this header, then paste `not_found` and the five endpoints from `main.py` below it and make three changes to each endpoint: `@app.` becomes `@router.`, the path loses `/api/v1/tasks` (so `""` for the list and create paths, `"/{task_id}"` for the others), and `store: StoreDep` becomes the first parameter. Here is `get_task` after the change; add `operation_id="create_task"` and so on to the other four.

```python
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException, Query, Request

from tasks_api.schemas import Page, TaskCreate, TaskRead, TaskUpdate
from tasks_api.storage import Task, TaskStore

router = APIRouter(prefix="/tasks", tags=["tasks"])


def get_store(request: Request) -> TaskStore:
    store: TaskStore = request.app.state.store
    return store


StoreDep = Annotated[TaskStore, Depends(get_store)]


@router.get("/{task_id}", operation_id="get_task")
async def get_task(task_id: UUID, store: StoreDep) -> TaskRead:
    task = store.get(task_id)
    if task is None:
        raise not_found(task_id)
    return TaskRead.model_validate(task)
```

`operation_id` gives each endpoint a short stable name. In week 7 it becomes a function name in the generated TypeScript client.

`src/tasks_api/main.py` (replace everything):

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI

from tasks_api.errors import register_exception_handlers
from tasks_api.routers import tasks
from tasks_api.settings import Settings, get_settings
from tasks_api.storage import TaskStore


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    print("startup:", app.state.settings.app_name)  # runs once, before the first request
    yield
    print("shutdown")  # runs once, when the server stops


def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or get_settings()
    app = FastAPI(title=settings.app_name, version="0.1.0", lifespan=lifespan)
    app.state.settings = settings
    app.state.store = TaskStore()  # created here, not in lifespan, so tests can see it

    @app.get("/healthz", tags=["health"])
    async def healthz() -> dict[str, str]:
        return {"status": "ok"}

    register_exception_handlers(app)
    app.include_router(tasks.router, prefix=settings.api_prefix)
    return app


app = create_app()
```

Check: the dev server restarts without an import error. Every `curl` command from Steps 4 to 8 still gives the same result. The `/docs` page shows two groups, `tasks` and `health`. Commit.

### Step 10: Tests and lint

Goal: `uv run pytest` proves the API works, and `ruff` keeps the code tidy.

Before you type this, understand: a test calls your API in memory, without a real server, and checks the answer. `httpx.AsyncClient` with `ASGITransport` talks directly to the app object. `anyio` is the pytest plugin that lets tests be `async`. A **fixture** is a function pytest runs to prepare something (here: a fresh app and a client) for each test. **ruff** is a linter (finds mistakes and bad style) and a formatter (fixes spacing).

Add to the end of `pyproject.toml` (our lines are up to 100 characters; ruff's default is 88):

```toml
[tool.ruff]
line-length = 100
```

Create `tests/conftest.py` (pytest reads this file automatically):

```python
from collections.abc import AsyncIterator

import pytest
from httpx import ASGITransport, AsyncClient

from tasks_api.main import create_app
from tasks_api.settings import Settings


@pytest.fixture
def anyio_backend() -> str:
    return "asyncio"


@pytest.fixture
async def client() -> AsyncIterator[AsyncClient]:
    app = create_app(Settings(_env_file=None))  # fresh app and empty store for every test
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
        yield ac
```

Create `tests/test_tasks.py`:

```python
import pytest
from httpx import AsyncClient

pytestmark = pytest.mark.anyio
BASE = "/api/v1/tasks"


async def create(client: AsyncClient, **fields: object) -> dict[str, object]:
    r = await client.post(BASE, json={"title": "Task", **fields})
    assert r.status_code == 201, r.text
    return r.json()


async def test_create_cleans_title_and_tags(client: AsyncClient) -> None:
    body = await create(client, title="  Hello ", tags=["Urgent", "urgent", " docs "])
    assert body["title"] == "Hello"
    assert body["tags"] == ["urgent", "docs"]


async def test_unknown_field_is_422_problem(client: AsyncClient) -> None:
    r = await client.post(BASE, json={"title": "", "titel": "typo"})
    assert r.status_code == 422
    assert r.headers["content-type"].startswith("application/problem+json")
    assert r.json()["instance"] == BASE


async def test_patch_changes_only_sent_fields(client: AsyncClient) -> None:
    created = await create(client, description="keep me")
    r = await client.patch(f"{BASE}/{created['id']}", json={"status": "done"})
    assert r.status_code == 200
    assert r.json()["status"] == "done"
    assert r.json()["description"] == "keep me"


async def test_delete_then_404(client: AsyncClient) -> None:
    created = await create(client)
    assert (await client.delete(f"{BASE}/{created['id']}")).status_code == 204
    assert (await client.delete(f"{BASE}/{created['id']}")).status_code == 404
```

Run everything:

```bash
uv run pytest
uv run ruff format . && uv run ruff check .
```

Check: pytest prints `4 passed`. ruff prints `All checks passed!`. Write one more test yourself for pagination (`total` and `len(items)`). Commit.

### Step 11: Optional: what `async` means

You wrote `async def` on every endpoint. Here is why, in short. The server runs one **event loop**: a single thread that runs one function at a time. When an `async` function reaches `await` on something slow (a database, another API), it pauses and the loop runs another request. That is how one process handles many requests at once. The rule: inside `async def`, never call a slow blocking function like `time.sleep()` or `requests.get()`. It freezes every other request. Use `asyncio.sleep()` and `httpx.AsyncClient` instead, or write the endpoint as a plain `def` (FastAPI then runs it in a thread). Our store has no `await` yet because a dict is instant. In week 3 the database calls become `await` calls, and this design pays off. More advanced tools (`gather`, `TaskGroup`) come later in the internship.

Check: explain to your partner, in two sentences, why `async def` plus `time.sleep(1)` is a bug.

## Read and watch

Readings, in this order:

1. [Models](https://docs.pydantic.dev/latest/concepts/models/) (Pydantic docs): the clearest explanation of what `model_validate` and `model_dump` do; read before Step 3 if you can.
2. [Handling Errors](https://fastapi.tiangolo.com/tutorial/handling-errors/) (FastAPI docs): shows exactly the handler override you wrote in Step 8.
3. [Bigger Applications](https://fastapi.tiangolo.com/tutorial/bigger-applications/) (FastAPI docs): the official version of the file layout from Step 9.
4. [Concurrency and async / await](https://fastapi.tiangolo.com/async/) (FastAPI docs): the friendly story version of Step 11, with pictures.
5. [Best Practices for Designing a Pragmatic RESTful API](https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api) (Vinay Sahni): one page of practical rules for paths, methods and status codes; skim the headings.

Videos:

1. [Asyncio Finally Explained: What the Event Loop Really Does](https://www.youtube.com/watch?v=RIVcqT2OGPA) (ArjanCodes, 13 min): watch before Step 11; the drawings make the event loop click.
2. [Anatomy of a Scalable Python Project (FastAPI)](https://www.youtube.com/watch?v=Af6Zr0tNNdE) (ArjanCodes, 21 min): compare his layout with yours from Step 9 and note two differences.

## Turn in

Push your `tasks-api` project to GitHub (your own repository, or the `interns/<yourhandle>/week-02/` folder if your supervisor told you so). It must contain:

- The code from Steps 1 to 10, with one commit per step.
- `README.md` with four short sections: how to run, how to test, the error format, the pagination format.
- `openapi.json`: run `curl -s localhost:8000/openapi.json > openapi.json` while the server runs.
- `notes/week-02.md`: your answers to the self-check below, in your own words.

Friday demo (10 minutes): clone your repository into a new folder, run `uv sync` and `uv run fastapi dev src/tasks_api/main.py`, show `/docs`, then create, list, update and delete a task with `curl`, show one `404` and one `422`, and run `uv run pytest`.

## Self-check

1. Which status code does a successful `POST /api/v1/tasks` return, and why not `200`?
2. What is the difference between `404` and `422` in your API?
3. Why does `PATCH` use `model_dump(exclude_unset=True)` and not `exclude_none=True`?
4. What does `extra="forbid"` protect you from?
5. What happens to your tasks when you restart the server, and why?
6. Why is `limit` capped at 100?
7. Why must an `async def` endpoint never call `time.sleep()`?
8. Why do the tests call `create_app()` instead of importing `app` from `main.py`?

### Answers

1. `201 Created`; `200` means "here is what you asked for", `201` tells the client something new now exists.
2. `404`: the path or id does not exist. `422`: the request is well formed but the data breaks a rule (empty title, unknown field, bad UUID, `limit` too big).
3. `exclude_unset` drops fields the client did not send but keeps fields sent as `null`; `exclude_none` would make it impossible to clear a field.
4. Client typos (`titel`) and unexpected fields being accepted silently.
5. They are lost; the store is a Python dict in memory. Week 3 replaces it with PostgreSQL.
6. One request with `limit=1000000` could use all the server's memory and time; a cap protects the server.
7. It blocks the event loop, so every other request waits until the sleep ends.
8. Each test gets a fresh app with an empty store, so tests do not affect each other.

## If you finish early

- Filtering: add `status` and `tag` query parameters to `list_tasks` and to `TaskStore.list`; add one test.
- Sorting: add `sort: Literal["created_at", "-created_at", "title", "-title"] = "-created_at"`; a leading `-` means descending.
- A `Dockerfile` for the API (week 1 skills) and a `compose.yaml` that starts it on port 8000.
- ETag basics: add an `ETag` header to `get_task` built from `updated_at`; later in the internship you use it for caching and safe updates.
