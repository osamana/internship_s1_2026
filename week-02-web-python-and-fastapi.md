# Week 2: Your first web API

This week you build a small **Tasks API**. An API (Application Programming Interface) is a program that other programs talk to over the network. Yours keeps tasks in memory and lets a client create, list, read, update and delete them. The whole week is one file, `main.py`. In week 3 you swap the memory for a real database. In weeks 5 to 7 you build a web page that talks to this API. Bigger topics (pagination, one error format for all errors, settings, splitting the code into files) come later in the internship.

A note from your supervisor: work through the steps in order, and commit after every step (`git add -A && git commit -m "feat: step N"`). Type every command yourself. Read each error message twice before you search. When you ask for help, bring the error, what you tried, and what you expected.

## What you will have by Friday

- [ ] A project `tasks-api` created with `uv`, running with `uv run fastapi dev main.py`.
- [ ] Five endpoints: `POST /tasks`, `GET /tasks`, `GET /tasks/{task_id}`, `PATCH /tasks/{task_id}`, `DELETE /tasks/{task_id}`.
- [ ] Input checks with Pydantic: bad input returns `422`, a missing task returns `404`.
- [ ] Two passing tests (`uv run pytest`).
- [ ] Clean code (`uv run ruff check .`).

## Before you start

Confirm your week-1 tools still work. Each command must print a version or a path, not an error.

```bash
uv --version            # 0.9 or newer
uv python find 3.12     # prints a path to Python 3.12
git --version && mkdir -p ~/projects && cd ~/projects
```

### Step 1: Create the project and say hello

Goal: a web server that answers one request. `uv` creates the project folder, a **virtual environment** (a private folder of installed packages) and `pyproject.toml` (the file that lists what your project needs). **FastAPI** is the library we use to build the API. `fastapi[standard]` also installs the server that runs your code.

```bash
uv init tasks-api --python 3.12
cd tasks-api
uv add "fastapi[standard]"
```

`uv init` already made a `main.py`. Replace its content with this:

```python
from fastapi import FastAPI

app = FastAPI(title="Tasks API")


@app.get("/hello")
def hello() -> dict[str, str]:
    return {"message": "hello"}
```

Read it line by line. `app` is your application. `@app.get("/hello")` says: when a client asks for the path `/hello`, run the function below. The function returns a Python dict. FastAPI turns it into **JSON** (a text format for data, like `{"message": "hello"}`). Now start the server. Leave this terminal open all week, and open a second terminal for other commands.

```bash
uv run fastapi dev main.py
```

Check: open http://127.0.0.1:8000/hello in your browser. You see `{"message":"hello"}`. Commit.

### Step 2: Requests and responses

Goal: know what a request and a response are, and see them with `curl`. A client sends a **request**. A request has a **method** (`GET` = read, `POST` = create, `PATCH` = change part, `DELETE` = remove), a **path** (`/hello`), and sometimes a **body** (the data, in JSON). The server answers with a **response**. A response has a **status code** (a number: `2xx` = success, `4xx` = the client made a mistake, `5xx` = the server made a mistake) and often a body. An **endpoint** is one method plus one path that your API handles. `curl` is a command that sends a request from the terminal. `-i` also shows the response **headers** (small key-value lines at the top of the response).

```bash
curl -i http://127.0.0.1:8000/hello
```

Check: the first line is `HTTP/1.1 200 OK` and one header line is `content-type: application/json`. Then open http://127.0.0.1:8000/docs. FastAPI makes this page from your code: it lists every endpoint and lets you call it from the browser. You see `GET /hello`.

### Step 3: The Task model

Goal: describe what a task looks like and what input is allowed. A **type hint** tells Python what kind of value a variable holds, for example `title: str`. Python itself ignores hints, but FastAPI and Pydantic read them. **Pydantic** is a library that turns a class with type hints into a **model**. A model checks incoming data (this is called validation) and converts it to and from JSON.

You write two models. `TaskCreate` is the input: what the client may send. `Task` is the output: a `TaskCreate` plus an `id`. They are different because the client does not choose the `id`; your server does. Add this to `main.py`, below the `import` line and above `app = FastAPI(...)`:

```python
from pydantic import BaseModel, Field


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    done: bool = False


class Task(TaskCreate):
    id: int
```

`BaseModel` is the class every Pydantic model starts from. `Field(min_length=1, max_length=200)` adds rules to `title`: an empty title is rejected. `done: bool = False` has a default value, so the client may leave it out. `Task(TaskCreate)` means `Task` has every field of `TaskCreate`, and adds `id`.

Check: the first terminal shows that the server restarted with no error.

### Step 4: Storage, create and list

Goal: `POST /tasks` creates a task and returns it with status `201`. `GET /tasks` returns all tasks. "In memory" means the tasks live in a Python dict inside the server. When the server stops, the tasks are gone. That is fine this week. The dict maps an id to a task, and a counter gives every new task the next id. Add this below `hello`:

```python
tasks: dict[int, Task] = {}
next_id = 1


@app.post("/tasks", status_code=201)
def create_task(body: TaskCreate) -> Task:
    global next_id
    task = Task(id=next_id, **body.model_dump())
    tasks[task.id] = task
    next_id += 1
    return task


@app.get("/tasks")
def list_tasks() -> list[Task]:
    return list(tasks.values())
```

How it works: `body: TaskCreate` tells FastAPI "read the JSON body and check it as `TaskCreate`". `global next_id` says we change the variable that lives outside the function. `body.model_dump()` gives a dict, and `**` spreads it into the `Task(...)` call. `-> Task` tells FastAPI to send the result back as JSON. `status_code=201` means "Created", the correct code for a successful `POST`.

```bash
curl -i -X POST localhost:8000/tasks -H 'content-type: application/json' -d '{"title":"Learn FastAPI"}'
curl -s localhost:8000/tasks
```

Check: the first command returns `HTTP/1.1 201 Created` and a task with `"id":1`. The second returns a list with that task.

### Step 5: Get one task, with 404

Goal: `GET /tasks/{task_id}` returns one task, or `404` if it does not exist. `{task_id}` in a path is a **path parameter**. FastAPI reads it from the URL and converts it to the type you declare (`int`). `HTTPException` is FastAPI's way to stop a request with a status code. Change the first line of `main.py` to `from fastapi import FastAPI, HTTPException`, then add this below `list_tasks`:

```python
@app.get("/tasks/{task_id}")
def get_task(task_id: int) -> Task:
    task = tasks.get(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="task not found")
    return task
```

404 or 422? `404 Not Found` means the id has the right form but no task has it. `422 Unprocessable Entity` means the input breaks a rule: an empty title, or `abc` where an `int` is needed. You write the `404` yourself; FastAPI returns the `422` automatically, before your function runs.

```bash
curl -i localhost:8000/tasks/1
curl -i localhost:8000/tasks/999
curl -i -X POST localhost:8000/tasks -H 'content-type: application/json' -d '{"title":""}'
```

Check: the first command returns `200` and task 1. The second returns `404` with `{"detail":"task not found"}`. The third returns `422` with a message about `title`.

### Step 6: Update and delete

Goal: `PATCH` changes only the fields the client sends. `DELETE` removes a task and returns `204`. In a `PATCH` body every field is optional, so you need a third model, `TaskUpdate`. `str | None = None` means: a string, or nothing, and nothing is the default. `model_dump(exclude_unset=True)` gives only the fields the client really sent. `model_copy(update=...)` makes a copy of a model with some fields changed. `204 No Content` is the status for a successful delete: the response has no body. Add this below `get_task`. Both endpoints call `get_task` first, so a missing id gives the same `404`.

```python
class TaskUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=200)
    done: bool | None = None


@app.patch("/tasks/{task_id}")
def update_task(task_id: int, body: TaskUpdate) -> Task:
    task = get_task(task_id)
    changes = body.model_dump(exclude_unset=True)
    tasks[task_id] = task.model_copy(update=changes)
    return tasks[task_id]


@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int) -> None:
    get_task(task_id)
    del tasks[task_id]
```

```bash
curl -s -X PATCH localhost:8000/tasks/1 -H 'content-type: application/json' -d '{"done":true}'
curl -i -X DELETE localhost:8000/tasks/1
curl -i -X DELETE localhost:8000/tasks/1
```

Check: after `PATCH`, `done` is `true` and `title` is unchanged. The first `DELETE` returns `204`, the second returns `404`.

Your complete `main.py` should now look like this. Compare it with yours line by line.

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    done: bool = False


class Task(TaskCreate):
    id: int


app = FastAPI(title="Tasks API")


@app.get("/hello")
def hello() -> dict[str, str]:
    return {"message": "hello"}


tasks: dict[int, Task] = {}
next_id = 1


@app.post("/tasks", status_code=201)
def create_task(body: TaskCreate) -> Task:
    global next_id
    task = Task(id=next_id, **body.model_dump())
    tasks[task.id] = task
    next_id += 1
    return task


@app.get("/tasks")
def list_tasks() -> list[Task]:
    return list(tasks.values())


@app.get("/tasks/{task_id}")
def get_task(task_id: int) -> Task:
    task = tasks.get(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="task not found")
    return task


class TaskUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=200)
    done: bool | None = None


@app.patch("/tasks/{task_id}")
def update_task(task_id: int, body: TaskUpdate) -> Task:
    task = get_task(task_id)
    changes = body.model_dump(exclude_unset=True)
    tasks[task_id] = task.model_copy(update=changes)
    return tasks[task_id]


@app.delete("/tasks/{task_id}", status_code=204)
def delete_task(task_id: int) -> None:
    get_task(task_id)
    del tasks[task_id]
```

### Step 7: Tests and lint

Goal: `uv run pytest` proves the API works, and `ruff` keeps the code tidy. A **test** is a small function that calls your API and checks the answer with `assert` ("this must be true, or the test fails"). `TestClient` calls your app directly in memory, so no server needs to run. **pytest** finds and runs every function that starts with `test_`. **ruff** is a linter (it finds mistakes and bad style) and a formatter (it fixes spacing). Install the three tools, then create `test_main.py` next to `main.py`:

```bash
uv add --dev pytest httpx2 ruff
```

```python
from fastapi.testclient import TestClient

from main import app

client = TestClient(app)


def test_create_then_get() -> None:
    r = client.post("/tasks", json={"title": "Learn FastAPI"})
    assert r.status_code == 201
    task_id = r.json()["id"]
    r = client.get(f"/tasks/{task_id}")
    assert r.status_code == 200
    assert r.json()["title"] == "Learn FastAPI"


def test_get_missing_is_404() -> None:
    assert client.get("/tasks/999").status_code == 404
```

Run the tests, then the formatter, then the linter:

```bash
uv run pytest
uv run ruff format .
uv run ruff check .
```

Check: pytest prints `2 passed`. ruff prints `All checks passed!`. Commit.

## Read and watch

1. [Models](https://docs.pydantic.dev/latest/concepts/models/) (Pydantic docs): what a model is and what `model_dump` does; read before Step 3 if you can, and skip the advanced parts.
2. [Handling Errors](https://fastapi.tiangolo.com/tutorial/handling-errors/) (FastAPI docs): the `HTTPException` you use in Step 5; stop before "Install custom exception handlers".
3. [Best Practices for Designing a Pragmatic RESTful API](https://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api) (Vinay Sahni): practical rules for paths, methods and status codes; read only the headings and the part about status codes.
4. Video: [Anatomy of a Scalable Python Project (FastAPI)](https://www.youtube.com/watch?v=Af6Zr0tNNdE) (ArjanCodes, 21 min): where your one file goes when a project grows. Just watch; you do not need to copy his layout this week.

## Turn in

- Push your `tasks-api` project to your own GitHub repository, with one commit per step.
- Add a `README.md` with two short sections: how to run, how to test.
- Add `notes/week-02.md` with your answers to the self-check below, in your own words.
- Friday demo (10 minutes): clone your repository into a new folder, run `uv sync` and `uv run fastapi dev main.py`, show `/docs`, create, list, update and delete a task with `curl`, show one `404` and one `422`, and run `uv run pytest`.

## Self-check

1. Which status code does a successful `POST /tasks` return, and why not `200`?
   Answer: `201 Created`; `200` means "here is what you asked for", `201` tells the client something new now exists.
2. What is the difference between `404` and `422` in your API?
   Answer: `404`: no task has that id. `422`: the input breaks a rule (empty title, `abc` instead of a number); FastAPI sends it before your code runs.
3. Why does `PATCH` use `model_dump(exclude_unset=True)`?
   Answer: it gives only the fields the client sent, so the other fields keep their old values.
4. What happens to your tasks when you restart the server, and why?
   Answer: they are lost; the store is a Python dict in memory. Week 3 replaces it with a real database.
5. Why are `TaskCreate` and `Task` two different models?
   Answer: the client sends a title (and maybe `done`), but the server chooses the `id`; the input must not contain it.
