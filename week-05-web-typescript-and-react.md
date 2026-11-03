# Week 5: A web page for your API

Weeks 2 to 4 built the back end: a Tasks API that only `curl` and the tests have talked to. This week you build the front end: a small web page, written in TypeScript with React and served by Vite, that logs in through your own `POST /auth/login`, lists the logged-in user's tasks, adds one, ticks one as done, deletes one, and logs out. Same rules as before: work in order, commit after every step (`git add -A && git commit -m "feat: step N"`), type every command yourself, and read every error twice before you search. Written against React 19.3.1, TypeScript 7.0.2, Vite 8.3.5, Vitest 5.1.0, Node 24.22.2 and pnpm 12.16.1. Left out on purpose, each comes later in the internship: routing (several pages with their own URLs), Next.js, TanStack Query (a library that loads data for you), Tailwind and shadcn (styling), and refresh tokens and cookies (staying logged in safely for longer than one hour).

## What you will have by Friday
- [ ] `tasks-api` answers the browser: CORS allows `http://localhost:5173`. One commit.
- [ ] A new project `tasks-web` (Vite, React, TypeScript) with `types.ts` for the API's JSON and `api.ts` with one typed `request` function behind `login`, `me`, `listTasks`, `createTask`, `updateTask` and `deleteTask`.
- [ ] A login screen with a session that survives a page refresh, and a Log out button.
- [ ] A task list that loads from your API, adds, ticks and deletes tasks, and counts the open ones.
- [ ] Two passing Vitest tests; `pnpm lint` and `pnpm build` clean.

## Before you start
```bash
cd ~/projects/tasks-api && docker compose up -d   # the database shows Up ... (healthy)
uv run alembic upgrade head && uv run pytest      # week 4 still passes: 9 passed
git status                                        # clean: everything from week 4 is committed
fnm install 24 && fnm use 24 && node --version    # v24.x.y (you installed fnm in week 1)
pnpm --version                                    # 10 or newer
```

## The flow

### Step 1: Let the browser in
Goal: your API accepts requests that come from a web page served on `http://localhost:5173`. Before you run this, understand: **CORS** (Cross-Origin Resource Sharing) is a browser rule. An **origin** is scheme, host and port together, like `http://localhost:5173`. A page from one origin may not read answers from another origin (`http://127.0.0.1:8000`) unless that server allows it in a response header. `curl` never cared, because the rule lives in the browser, not in the server. For a request that is not simple (a `PATCH`, or anything with an `Authorization` header) the browser first sends a **preflight**, an `OPTIONS` request that asks "may I?", and only then the real request. FastAPI's `CORSMiddleware` answers both. `allow_origins` lists the pages you trust; `*` for methods and headers is fine for a server that only runs on your laptop. Vite, which you install in Step 2, serves your page on port 5173, so that is the origin you allow. In `main.py`, add `from fastapi.middleware.cors import CORSMiddleware` below the `from fastapi import ...` line, and add this directly below `app = FastAPI(title="Tasks API")`:
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```
Start the server (`uv run fastapi dev main.py`) and leave it running all week. In a second terminal:
```bash
curl -i localhost:8000/hello -H 'Origin: http://localhost:5173'
curl -i -X OPTIONS localhost:8000/tasks -H 'Origin: http://localhost:5173' -H 'Access-Control-Request-Method: PATCH' -H 'Access-Control-Request-Headers: authorization,content-type'
curl -i localhost:8000/hello -H 'Origin: http://other.example'
```
Check: the first answer has a header `access-control-allow-origin: http://localhost:5173`. The second is `200 OK` with `access-control-allow-methods: DELETE, GET, HEAD, OPTIONS, PATCH, POST, PUT, QUERY` and `access-control-allow-headers: authorization,content-type`. The third has no `access-control-allow-origin` header at all: a page from that origin would be blocked. `uv run pytest` still prints `9 passed` and `uv run ruff check .` prints `All checks passed!`. Commit.

### Step 2: A new project with Vite
Goal: a React page runs on http://localhost:5173 and shows the heading Tasks. Before you run this, understand: **Node.js** runs JavaScript outside the browser; your tools run on it, your page runs in the browser. **pnpm** is to Node what `uv` is to Python: it installs packages and records them in `package.json` (like `pyproject.toml`) and `pnpm-lock.yaml`. **Vite** is the development server and build tool: it serves your files to the browser, updates the page the moment you save, and at the end bundles everything into plain files for a real web server. **React** is a library for building a page out of **components**; a component is a function that returns what it should look like. **JSX** is the HTML-like syntax inside those functions, `<h1>Tasks</h1>`; it is turned into JavaScript before the browser sees it. **TypeScript** is JavaScript plus types: the browser never runs it, Vite strips the types, and the compiler `tsc` checks them. Files end in `.ts`, or `.tsx` when they contain JSX. In the command below, `--template react-ts` picks React with TypeScript, `--no-eslint` keeps the template's default code checker (Oxlint), and `--no-immediate` says "do not install yet"; with these three flags it asks no questions. `.node-version` records the Node version so `fnm use` picks it in this folder.
```bash
cd ~/projects
pnpm create vite tasks-web --template react-ts --no-eslint --no-immediate
cd tasks-web
echo 24 > .node-version
pnpm install
```
Look around before you change anything: `index.html` is the one page; it loads `src/main.tsx`, which puts the `App` component from `src/App.tsx` into `<div id="root">`. The template ships a demo page. Delete its files with `rm -r src/App.css src/assets public/icons.svg`, replace the whole of `src/App.tsx` with the single line `export default function App() { return <h1>Tasks</h1>; }`, and replace `src/index.css` with:
```css
body { font-family: system-ui, sans-serif; max-width: 32rem; margin: 2rem auto; padding: 0 1rem; }
header, form, li { display: flex; align-items: center; gap: 0.5rem; }
header h1 { flex: 1; }
.login { flex-direction: column; align-items: stretch; }
ul { list-style: none; padding: 0; }
.done span { text-decoration: line-through; color: gray; }
.error { color: darkred; }
```
Run `pnpm dev` in a third terminal and leave it running all week: terminal one is the API, two is Vite, three is for commands.
Check: the terminal prints `VITE v8.3.5  ready in` (or a newer 8.x) and a time, then `Local:   http://localhost:5173/`. Open that address: the page shows Tasks. Change the text in `App.tsx`, save, and the browser shows the new text without a reload. This is a new repository, so run `git init` first (the template's `.gitignore` already skips `node_modules`), then commit.

### Step 3: Types first
Goal: TypeScript knows the exact shape of every JSON your API sends, and catches a wrong field before the browser ever runs the code. Before you run this, understand: a **type** describes the shape of a value. `type Task = { id: number; title: string; ... }` lists the fields and their types, like a Pydantic model, with one difference: TypeScript checks your code when you compile it and does nothing at run time. `project_id: number | null` is a **union**, one of several types, exactly like `int | None`. `title?: string` with a `?` is an **optional field**, it may be left out, like `str | None = None` in `TaskUpdate`. `LoadState<T>` is a **generic** type: `T` is a placeholder filled in later (`LoadState<Task[]>`), and the type is a union of three object shapes that share a `status` field with a different fixed text in each. When your code checks `state.status === "ready"`, TypeScript knows that `state.data` exists in that branch and that `state.message` does not. This is how a screen can be loading, failed, or ready, and never two of these at once. Last thing: JSON from `fetch` comes over the network, TypeScript cannot see your server, so to it the value is `unknown` ("could be anything") until you say what it is. Step 4 does the saying. Create `src/types.ts`:
```ts
// What the API sends, field by field. Compare with the JSON you saw in week 4.
export type Task = {
  id: number;
  title: string;
  done: boolean;
  project_id: number | null;
};

// What PATCH /tasks/{id} accepts: every field may be left out.
export type TaskChanges = { title?: string; done?: boolean };
export type TokenResponse = { access_token: string; token_type: string };
export type User = { id: number; email: string };
export type Session = { token: string; user: User }; // what the browser remembers after a login

// Exactly one of three shapes, never a mix of them.
export type LoadState<T> =
  | { status: "loading" }
  | { status: "error"; message: string }
  | { status: "ready"; data: T };
```
Now break it on purpose. Add the line `export const broken: Task = { id: "1", title: "Buy milk", done: false, project_id: null };` at the very end of the file and run the type checker, `pnpm exec tsc -b`. Read the error, delete the line, and run it again.
Check: with the extra line, `tsc` prints `src/types.ts(20,31): error TS2322: Type 'string' is not assignable to type 'number'.` (file, line, column, problem). Without it, `tsc` prints nothing: no output means no errors. Commit.

### Step 4: Talking to the API
Goal: one file knows how to call your API, and the rest of the app never touches `fetch`. Before you run this, understand: `fetch(url, options)` is the browser's function for sending a request. It returns a **Promise**, a value that arrives later, and `await` waits for it, like `async` and `await` in Python. `request<T>` is a generic function: the caller says what type the answer has, `request<Task[]>`, and gets exactly that back. Inside it, `(await response.json()) as T` is the "saying" from Step 3: "trust me, this JSON is a `T`". Nobody checks that at run time; it is fine here because the server is yours and Pydantic shaped the JSON. `response.ok` is true for every `2xx` status. When it is false, the function reads the API's `{"detail": "..."}` body and throws an **ApiError** that carries the status code and that message; Step 5 shows the message on screen. A `204` answer has no body, so there is nothing to parse. The base URL comes from `.env`: when Vite starts it reads this file and puts every variable whose name starts with `VITE_` into `import.meta.env`; nothing else from the file reaches the browser, and whatever does reach it is visible to every visitor, so never put a secret in a `VITE_` variable. This `.env` holds an address, not a secret, so unlike the API's `.env` it may be committed. Run `echo 'VITE_API_URL=http://127.0.0.1:8000' > .env`, then create `src/api.ts`:
```ts
import type { Task, TaskChanges, TokenResponse, User } from "./types";

const BASE_URL: string = import.meta.env.VITE_API_URL;

export class ApiError extends Error {
  status: number;
  constructor(status: number, message: string) {
    super(message);
    this.status = status;
  }
}

async function request<T>(path: string, init: RequestInit = {}, token?: string): Promise<T> {
  const headers = new Headers({ "Content-Type": "application/json" });
  if (token !== undefined) headers.set("Authorization", `Bearer ${token}`);
  const response = await fetch(BASE_URL + path, { ...init, headers });
  if (!response.ok) {
    const body = (await response.json().catch(() => null)) as { detail?: unknown } | null;
    const detail = typeof body?.detail === "string" ? body.detail : `HTTP ${response.status}`;
    throw new ApiError(response.status, detail);
  }
  if (response.status === 204) return undefined as T;
  return (await response.json()) as T;
}

export function login(email: string, password: string): Promise<TokenResponse> {
  return request<TokenResponse>("/auth/login", { method: "POST", body: JSON.stringify({ email, password }) });
}
export function me(token: string): Promise<User> {
  return request<User>("/auth/me", {}, token);
}
export function listTasks(token: string): Promise<Task[]> {
  return request<Task[]>("/tasks", {}, token);
}
export function createTask(token: string, title: string): Promise<Task> {
  return request<Task>("/tasks", { method: "POST", body: JSON.stringify({ title }) }, token);
}
export function updateTask(token: string, id: number, changes: TaskChanges): Promise<Task> {
  return request<Task>(`/tasks/${id}`, { method: "PATCH", body: JSON.stringify(changes) }, token);
}
export function deleteTask(token: string, id: number): Promise<void> {
  return request<void>(`/tasks/${id}`, { method: "DELETE" }, token);
}
```
Stop `pnpm dev` with Ctrl+C and start it again, so that Vite reads the new `.env`.
Check: `pnpm exec tsc -b` prints nothing, and `pnpm lint` prints `Found 0 warnings and 0 errors.` Commit.

### Step 5: The login screen
Goal: the page shows a login form, logs you in through your API, remembers you after a refresh, and lets you log out. Before you run this, understand: **state** is a value a component remembers between renders. `const [email, setEmail] = useState("")` gives the current value and a function to change it; every call to `setEmail` makes React run the component function again (a **render**) and update the page to match. A **controlled input** is an `<input>` whose `value` is state and whose `onChange` handler writes every keystroke back into that state, so React always knows what is in the box. An **event handler** is a function React calls when something happens (`onSubmit`, `onChange`, `onClick`); `event.preventDefault()` stops the browser from reloading the page, which is what a submitted form did before JavaScript. **Props** are the arguments of a component: `App` passes `onLogin`, a function, into `Login`, and `Login` calls it when the login worked; that is how a child talks to its parent. `App` owns the **session** (token and user) and mirrors it into `localStorage`, a small key-value store the browser keeps per origin, so a refresh finds it again; `useState(loadSession)` runs `loadSession` once, on the first render. `localStorage` is the simple choice for this week; the safer cookie version comes later. First make sure Ann exists, because `pytest` empties the tables: run `curl -i -X POST localhost:8000/auth/register -H 'content-type: application/json' -d '{"email":"ann@example.com","password":"correct horse battery"}'`. Then create `src/Login.tsx`:
```tsx
import { useState, type FormEvent } from "react";
import { ApiError, login, me } from "./api";
import type { Session } from "./types";

type Props = { onLogin: (session: Session) => void };

export function Login({ onLogin }: Props) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  async function handleSubmit(event: FormEvent) {
    event.preventDefault();
    setError("");
    try {
      const { access_token } = await login(email, password);
      const user = await me(access_token);
      onLogin({ token: access_token, user });
    } catch (err) {
      setError(err instanceof ApiError ? err.message : "could not reach the API");
    }
  }

  return (
    <form className="login" onSubmit={handleSubmit}>
      <h1>Log in</h1>
      <label>
        Email
        <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
      </label>
      <label>
        Password
        <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
      </label>
      <button type="submit">Log in</button>
      {error !== "" && <p className="error">{error}</p>}
    </form>
  );
}
```
Replace `src/App.tsx`:
```tsx
import { useState } from "react";
import { Login } from "./Login";
import type { Session } from "./types";

const STORAGE_KEY = "tasks-web-session";

function loadSession(): Session | null {
  const saved = localStorage.getItem(STORAGE_KEY);
  return saved === null ? null : (JSON.parse(saved) as Session);
}

export default function App() {
  const [session, setSession] = useState<Session | null>(loadSession);

  function handleLogin(next: Session) {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(next));
    setSession(next);
  }
  function handleLogout() {
    localStorage.removeItem(STORAGE_KEY);
    setSession(null);
  }

  if (session === null) return <Login onLogin={handleLogin} />;
  return (
    <>
      <header>
        <h1>Tasks</h1>
        <span>{session.user.email}</span>
        <button onClick={handleLogout}>Log out</button>
      </header>
      <p>Your tasks will appear here.</p>
    </>
  );
}
```
Check: the `curl` answers `201`, or `409` with `email already registered` if the last test run left Ann behind; both are fine. The page shows the form. Log in with a wrong password: `wrong email or password` appears in red, straight from your API's `401`. Log in with the right one: the heading Tasks, `ann@example.com`, a Log out button and `Your tasks will appear here.` Refresh the page: still logged in. Open the browser's developer tools (F12) and find Local Storage (Chrome: Application tab; Firefox: Storage tab): the key `tasks-web-session` holds your token and email. Click Log out: the form is back and the key is gone. Commit.

### Step 6: The task list
Goal: after login the page loads your tasks from the API, lets you add, tick and delete them, and shows how many are open. Before you run this, understand: a component function must only compute what to show; loading data over the network is a side effect, and React has `useEffect(fn, deps)` for that: it runs `fn` after the component is on screen, and again whenever a value in `deps` changes. Inside, `listTasks` returns a Promise; `.then` receives the tasks, `.catch` the error, and each path sets the `LoadState` from Step 3. The two `if` lines below the effect turn the other statuses into something to show. A token that expired (one hour, week 4) gives a `401`, and the effect logs you out. In development React runs every effect twice on purpose, to catch effects that cannot be repeated safely, so two `GET /tasks` lines in the API terminal are normal. An honest note: an effect is good enough for one list this week, but it has no cache and no protection against two answers arriving in the wrong order; a library that does this properly comes later in the internship. The handlers change the list without touching the old array: `[...tasks, task]` builds a new array with one more element (`...` is the spread), `map` builds one with one item replaced, `filter` one with one item removed. React redraws only when it gets a new array, so `tasks.push(task)` would change nothing on screen. `key={task.id}` tells React which `<li>` is which when the list changes. `countOpen` is a plain function in its own file, so Step 8 can test it without a browser. Create `src/tasks.ts`:
```ts
import type { Task } from "./types";

export function countOpen(tasks: Task[]): number {
  return tasks.filter((task) => !task.done).length;
}
```
Create `src/TaskList.tsx`:
```tsx
import { useEffect, useState, type FormEvent } from "react";
import { ApiError, createTask, deleteTask, listTasks, updateTask } from "./api";
import { countOpen } from "./tasks";
import type { LoadState, Task } from "./types";

type Props = { token: string; onLogout: () => void };

export function TaskList({ token, onLogout }: Props) {
  const [state, setState] = useState<LoadState<Task[]>>({ status: "loading" });
  const [title, setTitle] = useState("");

  useEffect(() => {
    listTasks(token)
      .then((tasks) => setState({ status: "ready", data: tasks }))
      .catch((err: unknown) => {
        if (err instanceof ApiError && err.status === 401) {
          onLogout(); // the token expired: back to the login form
        } else {
          setState({ status: "error", message: err instanceof Error ? err.message : String(err) });
        }
      });
  }, [token, onLogout]);

  if (state.status === "loading") return <p>Loading...</p>;
  if (state.status === "error") return <p className="error">{state.message}</p>;
  const tasks = state.data;

  async function handleAdd(event: FormEvent) {
    event.preventDefault();
    const task = await createTask(token, title);
    setState({ status: "ready", data: [...tasks, task] });
    setTitle("");
  }
  async function handleToggle(task: Task) {
    const updated = await updateTask(token, task.id, { done: !task.done });
    setState({ status: "ready", data: tasks.map((t) => (t.id === updated.id ? updated : t)) });
  }
  async function handleDelete(id: number) {
    await deleteTask(token, id);
    setState({ status: "ready", data: tasks.filter((t) => t.id !== id) });
  }

  return (
    <>
      <form onSubmit={handleAdd}>
        <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="New task" required />
        <button type="submit">Add</button>
      </form>
      <p>{countOpen(tasks)} open</p>
      <ul>
        {tasks.map((task) => (
          <li key={task.id} className={task.done ? "done" : undefined}>
            <input type="checkbox" checked={task.done} onChange={() => handleToggle(task)} />
            <span>{task.title}</span>
            <button onClick={() => handleDelete(task.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </>
  );
}
```
In `src/App.tsx`, add `import { TaskList } from "./TaskList";` directly below the `Login` import, and replace the line `<p>Your tasks will appear here.</p>` with `<TaskList token={session.token} onLogout={handleLogout} />`.
Check: after login you see `Loading...` for a moment, then the count and Ann's current tasks (none, or one left behind by the last test run). Add `Buy milk`, then `Learn React`: each appears at once, the box empties, and the count goes up by one each time. Tick `Buy milk`: it is crossed out and the count goes down by one. Delete `Learn React`: it disappears. The API terminal shows `"POST /tasks HTTP/1.1" 201 Created`, `"PATCH /tasks/... HTTP/1.1" 200 OK` and `"DELETE /tasks/... HTTP/1.1" 204 No Content`, with `OPTIONS` lines in between: the preflights from Step 1 (the browser remembers an answer for ten minutes, so not every request gets one). Commit.

### Step 7: Prove it is real
Goal: see with your own eyes that a click became a request, and the request became a row in PostgreSQL. Before you run this, understand: the browser's **developer tools** (F12) have a Network tab that lists every request the page sends, with method, status, headers and body. It is `curl -i` for the browser, and the first place to look when the page "does nothing". Open the Network tab, tick a task, and click the `PATCH` line that appears: Headers shows status `200` and the request header `Authorization: Bearer eyJ...`, Payload (Request in Firefox) shows `{"done":true}`, Response shows the task as JSON. Often right above it is an `OPTIONS` line, the preflight from Step 1. Then look in the database: `cd ~/projects/tasks-api && docker compose exec db psql -U app -d tasks -c 'SELECT id, title, done, owner_id FROM tasks'`.
Check: the task you ticked has `t` in the `done` column, the one you deleted is not there, and every row has Ann's `owner_id`. Refresh the page: the list is the same, because it comes from the database on every load, not from the page's memory. Nothing to commit.

### Step 8: Tests and checks
Goal: `pnpm test` runs two tests, and `pnpm lint` and `pnpm build` pass. Before you run this, understand: **Vitest** is this project's pytest: it finds files named `*.test.ts` and runs every `test(...)` in them; `expect(x).toBe(y)` is `assert x == y`. Both tests check plain functions with no browser and no React in them: the counter from Step 6 and the `ApiError` class from Step 4. Testing a component needs a fake browser document; that is the first bullet of "If you finish early". `pnpm pkg set` adds the `test` script to `package.json`, so that `pnpm test` works like `pnpm dev`. `pnpm build` is what a server runs later: `tsc -b` for the types, then Vite bundles everything into `dist/`. Run `pnpm add -D vitest`, then `pnpm pkg set scripts.test="vitest run"`, then create `src/tasks.test.ts`:
```ts
import { expect, test } from "vitest";
import { ApiError } from "./api";
import { countOpen } from "./tasks";
import type { Task } from "./types";

const tasks: Task[] = [
  { id: 1, title: "Buy milk", done: false, project_id: null },
  { id: 2, title: "Learn React", done: true, project_id: null },
];

test("countOpen counts only the tasks that are not done", () => {
  expect(countOpen(tasks)).toBe(1);
  expect(countOpen([])).toBe(0);
});

test("ApiError keeps the status and the API's message", () => {
  const err = new ApiError(401, "wrong email or password");
  expect(err.status).toBe(401);
  expect(err.message).toBe("wrong email or password");
});
```
Run `pnpm test`, then `pnpm lint`, then `pnpm build`.
Check: `pnpm test` prints `Test Files  1 passed (1)` and `Tests  2 passed (2)`. `pnpm lint` prints `Found 0 warnings and 0 errors.` `pnpm build` ends with `✓ built in` and a time, and a `dist/` folder exists (git ignores it). Commit.

## Read and watch
1. [Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html) (TypeScript Handbook): how an `if` on `state.status` tells TypeScript which fields exist; read before Step 3, and stop after "Discriminated unions".
2. [Render and Commit](https://react.dev/learn/render-and-commit) (react.dev, 10 min): the three steps between a `setState` call and the screen changing; read before Step 5.
3. [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects) (react.dev): what `useEffect` is for and why it runs twice in development; read before Step 6, up to and including "How to handle the Effect firing twice in development?".
4. Video: [Watch This Before Fetching in useEffect](https://www.youtube.com/watch?v=VQGduNUpVG4) (Josef Bender, 10 min): the problems with loading data in an effect, which your Step 6 has too; watch after Step 6, so you know what the library you meet later in the internship fixes.

## Turn in
- Push `tasks-web` to a new GitHub repository with one commit per step, and push the Step 1 commit of `tasks-api`.
- `README.md` in `tasks-web`: how to run (start the database and the API in `tasks-api`, then `pnpm install` and `pnpm dev` here, with the `.env` line) and how to test (`pnpm test`).
- Add `notes/week-05.md` with your answers to the self-check below, in your own words, and one screenshot of the page with a few tasks.
- Friday demo (10 minutes): fresh clone of both projects, start the database and the API, `pnpm install`, `pnpm dev`, log in with a wrong and then the right password, add, tick and delete a task, show the `PATCH` in the Network tab and the row in `psql`, refresh and stay logged in, log out, then run `pnpm test`, `pnpm lint` and `pnpm build`.

## Self-check
1. `curl` could always call your API. Why did the browser need the CORS change?
2. What does `(await response.json()) as T` promise, and who checks that the promise is kept?
3. `tasks.push(task)` adds a task to the array, but the page does not change. Why, and what does the right line look like?
4. Why does React want a `key` on every `<li>`, and why `task.id` and not the position in the list?
5. After login, the API terminal shows `GET /tasks` twice. Is that a bug?
6. Why is one `LoadState` union better than two separate booleans, `loading` and `failed`?

### Answers
1. CORS is a browser rule: a page from one origin may read another origin's answers only if that server allows it in a header. `curl` is not a browser and ignores origins.
2. That the JSON really has the shape `T`. Nobody checks at run time; TypeScript simply trusts you. That is acceptable because the server is yours and Pydantic shaped the JSON.
3. `push` changes the same array; React compares the new state with the old, sees the same array, and does nothing. `setState({ status: "ready", data: [...tasks, task] })` hands it a new array.
4. When the list changes, React matches old and new items by key, so it updates only what changed and keeps each item's own state. A position changes when you delete an item above it; an id never changes.
5. No. In development React runs every effect twice to catch effects that cannot be repeated safely. A production build runs it once.
6. Two booleans allow four combinations, and two of them are impossible (loading and failed at once). The union allows exactly three shapes, and TypeScript lets you read `data` only in the `ready` branch.

## If you finish early
- Show an error under the form when adding, ticking or deleting fails: wrap the three handlers in `try` and `catch` like `Login` does.
- Test a component: `pnpm add -D jsdom @testing-library/react`, then create `src/Login.test.tsx` whose first line is `// @vitest-environment jsdom` (a fake browser document for this file), `render(<Login onLogin={() => {}} />)` with `render` and `screen` from `@testing-library/react`, and check that `screen.getByRole("button", { name: "Log in" })` exists.
- For your own project: sketch its 3 to 5 screens on paper and write next to each one the API calls it needs. You build the first of them soon.
