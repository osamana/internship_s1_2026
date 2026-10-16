# Week 4: Users and login

Today anyone who can reach your Tasks API can read and change every task. This week the API gets users, passwords that are stored safely, a login that returns a token, and tasks that belong to the person who created them. Same rules as before: work in order, commit after every step (`git add -A && git commit -m "feat: step N"`), type every command yourself, and read every error twice before you search. Written against pwdlib 0.3.1, PyJWT 2.15.1 and email-validator 2.3.0, on top of the FastAPI, SQLAlchemy 2.1 and PostgreSQL 17 setup from weeks 2 and 3. Five things are left out on purpose, and each comes later in the internship: refresh tokens (staying logged in longer than one hour), cookies (the browser sending the token by itself), "Login with Google" (OAuth), roles (an admin who sees everything), and rate limiting (slowing down someone who tries thousands of passwords).

## What you will have by Friday
- [ ] A `users` table and three new migrations.
- [ ] `auth.py`: Argon2 password hashing, token creation and checking, and a `CurrentUser` dependency.
- [ ] `POST /auth/register`, `POST /auth/login` and `GET /auth/me`.
- [ ] Every task and project endpoint needs a token and shows only your own rows.
- [ ] Nine passing tests, `ruff` clean.
- [ ] `docs/proposal.md` for your own project, opened as a pull request.

## Before you start
```bash
cd ~/projects/tasks-api && docker compose up -d   # the database shows Up ... (healthy)
uv run alembic upgrade head && uv run pytest      # week 3 still passes: 4 passed
git status                                        # clean: everything from week 3 is committed
```

### Step 1: The users table
Goal: a `users` table exists, built from a new model. Before you type this, understand: **authentication** is finding out who is calling ("this is Ann"); **authorization** is deciding what that person may do ("Ann may see her tasks, not Bob's"). Steps 1 to 5 build authentication, Steps 6 and 7 add authorization. A user has an email and a **password hash**, a scrambled form of the password that cannot be turned back into the password (Step 2 explains). `unique=True` makes the database refuse two users with the same email, and `index=True` makes finding a user by email fast; together they become one unique index. Add this at the end of `models.py`:
```python
class UserRow(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(320), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
```
```bash
uv run alembic revision --autogenerate -m "create users"
uv run alembic upgrade head
docker compose exec db psql -U app -d tasks -c '\d users'
```
Check: the new file in `migrations/versions/` has `op.create_table('users', ...)` and `op.create_index(..., unique=True)`. `\d users` shows three columns and `"ix_users_email" UNIQUE`. Commit.

### Step 2: Passwords done right
Goal: see with your own eyes why the database never holds a password. Before you type this, understand: a **hash** is a one-way scramble: the same input always gives the same output, and nobody can compute the input from the output. A **salt** is a random value mixed in before hashing, so two users with the same password get different hashes, and an attacker who steals the table cannot look the hashes up in a prepared list. **Argon2** is the hash made for passwords: it is slow on purpose (about 50 ms), so guessing millions of passwords takes years. `pwdlib` does all of this; `PasswordHash.recommended()` picks Argon2 with good settings and a fresh salt every time. Install it and create `auth.py` with just three lines for now:
```bash
uv add "pwdlib[argon2]"
```
```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()
```
Now try it. Run `uv run python` and type each line after the `>>>` prompt:
```python
>>> from auth import password_hash
>>> h = password_hash.hash("correct horse battery")
>>> h
'$argon2id$v=19$m=65536,t=3,p=4$...'
>>> password_hash.hash("correct horse battery") == h
False
>>> password_hash.verify("correct horse battery", h)
True
>>> password_hash.verify("wrong", h)
False
```
Check: hashing the same password twice gives two different texts (the salt is stored inside, after the settings), and `verify` still says `True` for the right password and `False` for a wrong one. Rule for the whole internship: never store, log or print a plain password. Type `exit()`, then commit.

### Step 3: Register
Goal: `POST /auth/register` creates a user and never shows the hash. `EmailStr` is a Pydantic type that rejects anything that is not an email address; it needs one extra package. The input model `UserCreate` wants a password of at least 8 characters. The output model `UserRead` has only `id` and `email`, so the hash never leaves the server. `db.scalar(...)` runs a `select` and returns the first row or `None`; you use it to find an existing email and answer `409 Conflict`, the code for "this already exists". Run `uv add "pydantic[email]"`. Then in `main.py` add `EmailStr` to the `pydantic` import, add `UserRow` to the `models` import, add `from auth import password_hash` above `from db import get_db`, and add this at the end of the file:
```python
class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: str


@app.post("/auth/register", status_code=201)
def register(body: UserCreate, db: DbSession) -> UserRead:
    if db.scalar(select(UserRow).where(UserRow.email == body.email)) is not None:
        raise HTTPException(status_code=409, detail="email already registered")
    row = UserRow(email=body.email, password_hash=password_hash.hash(body.password))
    db.add(row)
    db.commit()
    db.refresh(row)
    return UserRead.model_validate(row)
```
Start the server (`uv run fastapi dev main.py`), register Ann twice, then send bad input:
```bash
curl -i -X POST localhost:8000/auth/register -H 'content-type: application/json' -d '{"email":"ann@example.com","password":"correct horse battery"}'
curl -i -X POST localhost:8000/auth/register -H 'content-type: application/json' -d '{"email":"ann@example.com","password":"correct horse battery"}'
curl -i -X POST localhost:8000/auth/register -H 'content-type: application/json' -d '{"email":"not-an-email","password":"short"}'
```
Check: `201` with `{"id":1,"email":"ann@example.com"}` and no password in the answer, then `409` with `email already registered`, then `422` with two errors. Run `docker compose exec db psql -U app -d tasks -c 'SELECT * FROM users'`: the `password_hash` column starts with `$argon2id$`. Commit.

### Step 4: Tokens and login
Goal: `POST /auth/login` checks the password and hands back a token. Before you type this, understand: after login the client must prove who it is on every request, without sending the password again. It sends a **token**, a short text the server gave it. Ours is a **JWT** (JSON Web Token): a signed note like "user 1, valid until 15:30". The server signs it with a **secret**, a long random text only the server knows; on every request it checks the signature, and a changed or invented token fails. So the server can trust the note without a database lookup. Anyone can read the note (it is encoded, not encrypted), so put nothing private in it. If the secret leaks, anyone can write notes that the server trusts, so it lives in `.env` like the database password. The two fields inside, called claims, are `sub` (subject: the user id, as a string, because the standard says so) and `exp` (expiry, one hour from now). Install PyJWT, generate a secret and add it to both env files:
```bash
uv add pyjwt
echo "SECRET_KEY=$(uv run python -c 'import secrets; print(secrets.token_hex(32))')" >> .env
echo 'SECRET_KEY=change-me' >> .env.example
```
`load_dotenv()` is called here too, because `auth.py` can be imported before `db.py`. `jwt.encode` builds and signs the token. `jwt.decode` checks the signature and the expiry and gives the claims back, or raises an error. `algorithms=["HS256"]` names the one signing method you accept; never let the token choose. Add this to `auth.py`: the four import lines next to the existing import, the rest below `password_hash`:
```python
import os
from datetime import UTC, datetime, timedelta

import jwt
from dotenv import load_dotenv

load_dotenv()
SECRET_KEY = os.environ["SECRET_KEY"]


def create_token(user_id: int) -> str:
    expires = datetime.now(UTC) + timedelta(minutes=60)
    claims = {"sub": str(user_id), "exp": expires}
    return jwt.encode(claims, SECRET_KEY, algorithm="HS256")


def decode_token(token: str) -> int:
    claims = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    return int(claims["sub"])
```
Now the endpoint. `LoginIn` has no minimum length: a short password is simply wrong, not invalid. `401 Unauthorized` means "I do not know who you are". A wrong email and a wrong password return the same `401` message, because a different message would tell an attacker which emails have an account. In `main.py`, change the import to `from auth import create_token, password_hash` and add at the end:
```python
class LoginIn(BaseModel):
    email: EmailStr
    password: str


class TokenOut(BaseModel):
    access_token: str
    token_type: str = "bearer"


@app.post("/auth/login")
def login(body: LoginIn, db: DbSession) -> TokenOut:
    user = db.scalar(select(UserRow).where(UserRow.email == body.email))
    if user is None or not password_hash.verify(body.password, user.password_hash):
        raise HTTPException(status_code=401, detail="wrong email or password")
    return TokenOut(access_token=create_token(user.id))
```
```bash
curl -s -X POST localhost:8000/auth/login -H 'content-type: application/json' -d '{"email":"ann@example.com","password":"correct horse battery"}'
curl -i -X POST localhost:8000/auth/login -H 'content-type: application/json' -d '{"email":"ann@example.com","password":"wrong"}'
curl -i -X POST localhost:8000/auth/login -H 'content-type: application/json' -d '{"email":"nobody@example.com","password":"wrong"}'
```
Check: the first answer is `{"access_token":"eyJ...","token_type":"bearer"}`; the token has three parts separated by dots (header, claims, signature). The other two both return `401` with the same `wrong email or password`. Commit.

### Step 5: Who is calling
Goal: a dependency turns the token in a request into a `UserRow`, and `GET /auth/me` tells you who you are. Before you type this, understand: the client sends the token in a request header, `Authorization: Bearer <token>`. The **bearer header** means "whoever carries this token is allowed in", so a token must be kept as secret as a password. `HTTPBearer` is FastAPI's reader for that header; `auto_error=False` means "give me `None` when the header is missing", so that you write the error yourself: the same `401` for every case, a missing header, a broken or expired token (both are a `jwt.InvalidTokenError`), or a user deleted after the token was made. A **dependency** (week 3, `Depends`) is a function FastAPI runs before your endpoint; `get_current_user` is one, and `CurrentUser` packs it into a type name, exactly like `DbSession`. Any endpoint with a `user: CurrentUser` parameter now needs a valid token. Your complete `auth.py`:
```python
import os
from datetime import UTC, datetime, timedelta
from typing import Annotated

import jwt
from dotenv import load_dotenv
from fastapi import Depends, HTTPException
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from pwdlib import PasswordHash
from sqlalchemy.orm import Session

from db import get_db
from models import UserRow

load_dotenv()
SECRET_KEY = os.environ["SECRET_KEY"]
password_hash = PasswordHash.recommended()
bearer = HTTPBearer(auto_error=False)


def create_token(user_id: int) -> str:
    expires = datetime.now(UTC) + timedelta(minutes=60)
    claims = {"sub": str(user_id), "exp": expires}
    return jwt.encode(claims, SECRET_KEY, algorithm="HS256")


def decode_token(token: str) -> int:
    claims = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    return int(claims["sub"])


def get_current_user(
    creds: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer)],
    db: Annotated[Session, Depends(get_db)],
) -> UserRow:
    not_authenticated = HTTPException(status_code=401, detail="not authenticated")
    if creds is None:
        raise not_authenticated
    try:
        user_id = decode_token(creds.credentials)
    except jwt.InvalidTokenError:
        raise not_authenticated
    user = db.get(UserRow, user_id)
    if user is None:
        raise not_authenticated
    return user


CurrentUser = Annotated[UserRow, Depends(get_current_user)]
```
In `main.py`, change the import to `from auth import CurrentUser, create_token, password_hash` and add at the end:
```python
@app.get("/auth/me")
def me(user: CurrentUser) -> UserRead:
    return UserRead.model_validate(user)
```
Log in again, copy the `access_token` value from the answer, and keep it in a shell variable:
```bash
TOKEN='paste the access_token value here'
curl -i localhost:8000/auth/me -H "Authorization: Bearer $TOKEN"
curl -i localhost:8000/auth/me
curl -i localhost:8000/auth/me -H "Authorization: Bearer garbage"
```
Check: `200` with Ann's id and email, then `401` with `not authenticated` twice. Open http://127.0.0.1:8000/docs: there is a new **Authorize** button at the top right. Paste the token (without the word `Bearer`), click Authorize, and `GET /auth/me` works from the browser too. Commit.

### Step 6: Tasks belong to users
Goal: every task has an owner, and you only see your own. Before you type this, understand: **ownership** is the simplest authorization rule: a row belongs to the user who created it, and nobody else may see it. `owner_id` is a foreign key to `users` (week 3), and `Mapped[int]` without `| None` makes it NOT NULL: every task must have an owner. Your development database already has tasks and projects without one, so the migration would fail. A real site would give them an owner inside the migration; this is test data, so delete it (projects too, they get an owner in Step 7). Add this inside `TaskRow`, after `project`:
```python
    owner_id: Mapped[int] = mapped_column(
        ForeignKey("users.id", name="fk_tasks_owner"), index=True
    )
```
```bash
docker compose exec db psql -U app -d tasks -c 'DELETE FROM tasks' -c 'DELETE FROM projects'
uv run alembic revision --autogenerate -m "add task owner"
uv run alembic upgrade head
```
Now `main.py`. Every task endpoint gets a `user: CurrentUser` parameter. `create_task` stores `owner_id=user.id`. `list_tasks` adds one `.where(...)` to the first `select`, so the filter can never be forgotten. `find_task` treats "not yours" exactly like "does not exist": `404`, not `403 Forbidden` ("I know who you are, and you may not do this"). A `403` would confirm to Bob that task 8 exists and belongs to someone; `404` tells him nothing. Change these three functions (lines not shown stay as they are):
```python
def create_task(body: TaskCreate, user: CurrentUser, db: DbSession) -> Task:
    if body.project_id is not None:
        find_project(db, body.project_id)
    row = TaskRow(**body.model_dump(), owner_id=user.id)


def list_tasks(
    user: CurrentUser, db: DbSession, done: bool | None = None, q: str | None = None
) -> list[Task]:
    stmt = select(TaskRow).where(TaskRow.owner_id == user.id).order_by(TaskRow.id)


def find_task(db: Session, task_id: int, user: UserRow) -> TaskRow:
    row = db.get(TaskRow, task_id)
    if row is None or row.owner_id != user.id:
        raise HTTPException(status_code=404, detail="task not found")
    return row
```
In `get_task`, `update_task` and `delete_task`, add `user: CurrentUser` to the parameters and change `find_task(db, task_id)` to `find_task(db, task_id, user)`. Then create a task as Ann, register and log in a second user, `bob@example.com`, the same way as in Steps 3 to 5, and keep his token in `TOKEN2`:
```bash
curl -i localhost:8000/tasks
curl -s -X POST localhost:8000/tasks -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' -d '{"title":"Buy milk"}'
curl -i localhost:8000/tasks/8 -H "Authorization: Bearer $TOKEN2"
curl -s localhost:8000/tasks -H "Authorization: Bearer $TOKEN2"
```
Check: without a token, `401`. Ann's task is created; note its `id` and use it in place of `8` (ids keep counting after a `DELETE`). Bob gets `404` with `task not found` for it, and his list is `[]`. The same `PATCH` and `DELETE` by Bob also give `404`. Commit.

### Step 7: Projects too
Goal: projects have owners as well, and nobody can put a task into someone else's project. Repeat the pattern: add the same three lines to `ProjectRow`, after `tasks`, with the name `fk_projects_owner`; run `uv run alembic revision --autogenerate -m "add project owner"`, then `uv run alembic upgrade head`. In `main.py`, add `user: CurrentUser` to `create_project`, `list_projects` and `list_project_tasks`, and give `find_project` a third parameter `user: UserRow` like `find_task`. Only these lines change, in `create_project`, `list_projects` and `find_project`:
```python
    row = ProjectRow(**body.model_dump(), owner_id=user.id)

    stmt = select(ProjectRow).where(ProjectRow.owner_id == user.id)
    rows = db.scalars(stmt.order_by(ProjectRow.id))

    if row is None or row.owner_id != user.id:
```
Every call of `find_project` now passes the user: `find_project(db, project_id, user)` in `list_project_tasks` and `find_project(db, body.project_id, user)` in `create_task`. That last one is the point: when Bob sends the `project_id` of Ann's project, `find_project` answers `404` and the task is never created.
```bash
curl -s -X POST localhost:8000/projects -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' -d '{"name":"Home"}'
curl -i -X POST localhost:8000/tasks -H "Authorization: Bearer $TOKEN2" -H 'content-type: application/json' -d '{"title":"Sneak in","project_id":3}'
curl -s localhost:8000/projects -H "Authorization: Bearer $TOKEN2"
```
Check: Ann's project is created (use its `id` in place of `3`); Bob's task gets `404` with `project not found`; Bob's project list is `[]`. Commit.

### Step 8: Tests
Goal: `uv run pytest` proves every rule above. Before you type this, understand: every test now needs a logged-in user, so a helper `login` registers an email (if it already exists, the `409` is simply ignored), logs in, and returns the header to pass as `headers=`. The cleanup fixture must delete users too, and users last: tasks and projects point to users, so PostgreSQL refuses to delete a user who still owns rows. In `test_main.py`: add `UserRow` to the `models` import, add `PASSWORD = "correct horse battery"` below `client = TestClient(app)`, add `db.execute(delete(UserRow))` after the `ProjectRow` line of the fixture, and add the helper below the fixture:
```python
def login(client: TestClient, email: str) -> dict[str, str]:
    client.post("/auth/register", json={"email": email, "password": PASSWORD})
    r = client.post("/auth/login", json={"email": email, "password": PASSWORD})
    return {"Authorization": f"Bearer {r.json()['access_token']}"}
```
In each of the four week-3 tests, add `h = login(client, "ann@example.com")` as the first line and `headers=h` to every `client.post` and `client.get` call. Then add five tests at the end:
```python
def test_register_twice_is_409() -> None:
    body = {"email": "ann@example.com", "password": PASSWORD}
    assert client.post("/auth/register", json=body).status_code == 201
    assert client.post("/auth/register", json=body).status_code == 409


def test_wrong_password_is_401() -> None:
    login(client, "ann@example.com")
    body = {"email": "ann@example.com", "password": "wrong"}
    assert client.post("/auth/login", json=body).status_code == 401


def test_no_token_is_401() -> None:
    assert client.get("/tasks").status_code == 401


def test_me_returns_email() -> None:
    r = client.get("/auth/me", headers=login(client, "ann@example.com"))
    assert r.status_code == 200
    assert r.json()["email"] == "ann@example.com"


def test_other_users_task_is_404() -> None:
    ann, bob = login(client, "ann@example.com"), login(client, "bob@example.com")
    task_id = client.post("/tasks", json={"title": "Ann's"}, headers=ann).json()["id"]
    assert client.get(f"/tasks/{task_id}", headers=bob).status_code == 404
    assert client.get("/tasks", headers=bob).json() == []
```
Check: `uv run pytest` prints `9 passed` (a little slower than last week: Argon2 is slow on purpose). `uv run ruff format .` wraps two long lines for you, then `uv run ruff check .` prints `All checks passed!`. Commit.

### Step 9: Your project proposal
Goal: from next week each of you builds your own project, and the plan is written down first. Create a new repository for it with `docs/proposal.md` inside, with these eight headings and one short paragraph or list under each. The example lines are from a clinic appointment scheduler; yours are about your own idea.
1. **Problem and user**: who has what problem. "A small clinic books by phone and paper; the receptionist double-books and patients forget their time."
2. **The 3 to 5 main screens or flows**: "Login; a doctor's day as a calendar; book or cancel an appointment; a patient's list of their own appointments."
3. **Data**: the tables and how they link, as a bullet list or a small Mermaid ER diagram. "`users`, `doctors`, `patients`, `appointments` (`doctor_id`, `patient_id`, `starts_at`, `ends_at`, `status`)."
4. **Endpoints**: a table with method, path, and who may call it. "`POST /appointments` | patient or receptionist" and "`GET /doctors/{id}/day` | any logged-in user".
5. **Roles**: at least two kinds of user. "Patient (books for themselves only), receptionist (books for anyone and sees every doctor)."
6. **One rule the server must enforce**: "A doctor never has two appointments that overlap in time; the server rejects the second one, and a test proves it."
7. **Weekly plan for weeks 5 to 12**: one line per week. "Week 5: repo, users, tables, migrations. Week 6: booking endpoints and the overlap rule. Week 7: first web pages. Week 8: a working demo of login and booking. Week 9: deploy..."
8. **Three risks**: what could go wrong, and what you do then. "SMS reminders need a paid service; fallback: print the reminder to the server log."
Open the file as a pull request and send the link; your supervisor reviews it there. It is due at the end of this week, and one page is enough.

## Read and watch
1. [OAuth2 with Password (and hashing), Bearer with JWT tokens](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/) (FastAPI docs): the same `pwdlib` and PyJWT code as yours, with a form login instead of JSON; read it before Step 4 and skip the OAuth2 parts.
2. [Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) (OWASP): why Argon2id, and what a salt protects against; read the introduction and the Argon2id section before Step 2.
3. [PyJWT usage](https://pyjwt.readthedocs.io/en/stable/) (PyJWT docs): `encode`, `decode` and how `exp` is checked; read the first page before Step 4.
4. Video: [Session vs Token Authentication in 100 Seconds](https://www.youtube.com/watch?v=UBUNrFtufWo) (Fireship, 2 min): the two ways to remember a logged-in user; watch before Step 4.

## Turn in
- Push `tasks-api` with one commit per step. On GitHub, confirm `.env` is still not there and `.env.example` shows `SECRET_KEY=change-me`.
- Add a section "Log in" to your `README.md`: how to register, log in, and send the token with `curl`.
- Add `notes/week-04.md` with your answers to the self-check below, in your own words.
- Friday demo (10 minutes): fresh clone, `docker compose up -d`, `uv sync`, `uv run alembic upgrade head`, start the server, register two users and log in as both, show that each sees only their own tasks and that one gets `404` for the other's task, show `/auth/me` without a token, run `uv run pytest`, then open your proposal pull request.

## Self-check
1. Hashing the same password twice gives two different texts. Why, and how can `verify` still say `True`?
2. What is inside your token, and why can the server trust it without asking the database?
3. Why do a wrong email and a wrong password return the same message?
4. Bob asks for Ann's task. Why `404` and not `403`?
5. Why must the test fixture delete users last?

### Answers
1. Each hash gets a fresh random salt, stored inside the hash text; `verify` reads that salt, hashes the typed password with it, and compares.
2. The user id (`sub`) and an expiry (`exp`), signed with `SECRET_KEY`; a changed token fails the signature check, so only the server could have made it.
3. A different message would tell an attacker which emails have an account; one message gives nothing away.
4. `403` would confirm that the task exists and belongs to someone; `404` looks exactly like a task that never existed.
5. Tasks and projects point to users through foreign keys; PostgreSQL refuses to delete a user who still owns rows.
