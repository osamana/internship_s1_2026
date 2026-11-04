# Week 6: Pages for your app with Next.js

Week 5 built `tasks-web`: one React page served by Vite. This week the same code moves into Next.js, the framework you use for the rest of the internship, and gains what Vite did not have: several pages with their own URLs, a shared layout with a navigation bar, pages rendered on the server, a page per task with a loading and a not-found state, Tailwind CSS and shadcn/ui for the look, and a production build. Login stays exactly as in week 5: the token lives in `localStorage` and every API call goes from the browser. Same rules as before: work in order, commit after every step (`git add -A && git commit -m "feat: step N"`), type every command yourself, and read every error twice before you search. Written against Next.js 16.4.4, React 19.3.0, TypeScript 5.9.3, Tailwind CSS 4.3.3, shadcn CLI 4.23.0, Node 24.22.2 and pnpm 12.16.1. Left out on purpose, each comes later in the internship: loading data on the server with cookies and a real session, `proxy.ts` (code that runs before every request), Cache Components and `use cache`, Server Actions, TanStack Query, a forms library, tests across the whole stack, Docker and deployment.

## What you will have by Friday
- [ ] `tasks-api` also allows the origin `http://localhost:3000`. One commit.
- [ ] A new project `tasks-next` (Next.js, TypeScript, Tailwind, shadcn/ui) with the week 5 `types.ts`, `tasks.ts` and `api.ts` (plus `getTask`) in `src/lib/`.
- [ ] Pages `/`, `/login`, `/tasks` and `/tasks/[id]` (with `loading.tsx` and `not-found.tsx`) with one shared navigation bar, rendered on the server.
- [ ] A `useSession()` hook, a login page that sends you to `/tasks`, and a tasks page that sends you to `/login` when you are not logged in.
- [ ] `pnpm lint` and `pnpm build` clean, and `pnpm start` serving the build.

## Before you start
```bash
cd ~/projects/tasks-api && docker compose up -d   # the database shows Up ... (healthy)
uv run alembic upgrade head && uv run pytest      # week 4 still passes: 9 passed
uv run fastapi dev main.py                        # terminal one: the API, leave it running all week
fnm use 24 && node --version && pnpm --version    # in a second terminal: v24.x.y, then 10 or newer
```

## The flow

### Step 1: A new project with Next.js
Goal: a Next.js app runs on http://localhost:3000 and shows Tasks. Before you run this, understand: **Next.js** is a framework built on React. Vite gave you a development server and a build; Next.js adds a **server** that renders your pages to HTML before the browser gets them, a **router** that turns folders into pages (`src/app/login/page.tsx` is the page at `/login`), and a production build. The command below answers every question the installer would otherwise ask: TypeScript, ESLint, Tailwind, the App Router, a `src/` folder, `@/*` as a short name for `src/` in imports, pnpm, and no to four extras you do not need this week: the React Compiler, Cache Components, an `AGENTS.md` notes file and a feedback report. The installer also runs `git init` and makes a first commit. `.node-version` records the Node version for `fnm`, and the `.svg` files belong to the demo page you are about to replace.
```bash
cd ~/projects
pnpm create next-app@latest tasks-next --ts --eslint --tailwind --app --src-dir --import-alias "@/*" --use-pnpm --no-react-compiler --no-cache-components --no-agents-md --no-agent-feedback
cd tasks-next
echo 24 > .node-version
rm public/*.svg
```
Look around before you change anything. `src/app/layout.tsx` is the **root layout**: the `<html>` and `<body>` that every page shares; it imports `src/app/globals.css`, the one CSS file of the project, which starts with `@import "tailwindcss";`. `src/app/page.tsx` is the page at `/`: a component exported as the file's default export. Replace the whole of `src/app/page.tsx` with the single line `export default function Home() { return <h1>Tasks</h1>; }`, then run `pnpm dev` in terminal two and leave it running all week; terminal three is for commands.
Check: the terminal prints `▲ Next.js 16.4.4 (Turbopack)`, `- Local:         http://localhost:3000` and `✓ Ready in` with a time. Open the address: the page shows Tasks, as plain text for now. `git log --oneline` shows `Initial commit from Create Next App`. Commit your changes on top of it.

### Step 2: Tailwind and shadcn/ui
Goal: the project has its design system before it has pages. Before you run this, understand: **Tailwind CSS** replaces CSS files with small classes written on the element: `className="flex gap-2 p-4"` is a flex row with a gap and padding, and you never write a `.login { ... }` rule again. The template installed it; Tailwind v4 has no config file, the `@import` line in `globals.css` is the whole setup. Tailwind also resets every browser default, so an `<h1>` looks like normal text until you style it. **shadcn/ui** is not a library you import from `node_modules`: it is a command that copies ready-made, accessible components into `src/components/ui/` as source files that you own and may edit. `init -d` takes the default configuration, so it asks nothing: it writes `components.json`, adds a few packages, rewrites `globals.css` with the theme's colors as CSS variables (`--primary`, `--muted-foreground` and so on, each with a class like `text-muted-foreground`), and copies `button.tsx` and `src/lib/utils.ts`. `add` copies more components. The last line gives headings their size back; `printf` with `\n` puts the rule on its own line, because the file shadcn wrote has no line break at the end.
```bash
pnpm dlx shadcn@latest init -d
pnpm dlx shadcn@latest add input checkbox
printf '\nh1 { font-size: 1.5rem; font-weight: 700; }\n' >> src/app/globals.css
```
Check: `src/components/ui/` holds `button.tsx`, `checkbox.tsx` and `input.tsx`; `git status` shows `globals.css`, `package.json` and `pnpm-lock.yaml` as modified and `components.json`, `src/components/` and `src/lib/` as new. Open `button.tsx`: about 60 lines of ordinary React and Tailwind. The browser shows Tasks as a heading again. `pnpm lint` prints the command it runs, `$ eslint`, and nothing else: no output means no problems. Commit.

### Step 3: Pages and links
Goal: three pages with their own URLs share one navigation bar, and the server renders them. Before you run this, understand: a **route is a folder with a `page.tsx`**: `src/app/login/page.tsx` is `/login`, `src/app/tasks/page.tsx` is `/tasks`. A **layout** is the frame around the pages below it; the root layout stays on screen while you move between pages, so it is not rendered again and keeps its state. `<Link>` from `next/link` renders an `<a>`, but a click does not reload the page: Next.js fetches only the new page's content and swaps it in. A **Server Component** is the default for every component in `src/app`: it runs on the server, its HTML is sent to the browser, and it cannot use `useState` or click handlers. A **Client Component** starts with `"use client"` and runs in the browser too (the server still renders it once); you need one for anything interactive. The layout and these three pages are Server Components. `next/font` downloads the Geist font when the project builds and serves it from your own server; `variable: "--font-sans"` is the CSS variable that shadcn's `font-sans` class expects. Replace `src/app/layout.tsx`:
```tsx
import { Geist } from "next/font/google";
import Link from "next/link";
import type { ReactNode } from "react";
import "./globals.css";

const geist = Geist({ subsets: ["latin"], variable: "--font-sans" });

export const metadata = { title: "Tasks" };

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en" className={`${geist.variable} font-sans`}>
      <body className="mx-auto max-w-lg p-4">
        <nav className="mb-6 flex gap-4 border-b pb-2">
          <Link href="/">Home</Link>
          <Link href="/login">Log in</Link>
          <Link href="/tasks">Tasks</Link>
        </nav>
        <main>{children}</main>
      </body>
    </html>
  );
}
```
Create `src/app/login/page.tsx` with the single line `export default function LoginPage() { return <h1>Log in</h1>; }` and `src/app/tasks/page.tsx` with `export default function TasksPage() { return <h1>Your tasks</h1>; }`.
Check: the three links switch pages and the heading changes. Open the developer tools (F12), Network tab, and click a link: there is no new document request, only a small fetch for the page content. In terminal three, `curl -s localhost:3000/login | grep -o '<h1>[^<]*</h1>'` prints `<h1>Log in</h1>`: the server rendered the heading, no JavaScript ran. `curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/nope` prints `404`, Next.js's own not-found page. Commit.

### Step 4: Bring the code over
Goal: the week 5 types and API functions live in `src/lib/`, and `tsc` is happy. Before you run this, understand: `types.ts` and `tasks.ts` do not change at all. `api.ts` changes in two places: `import.meta.env.VITE_API_URL` becomes `process.env.NEXT_PUBLIC_API_URL`, and it gains `getTask` for Step 7. **`NEXT_PUBLIC_`** is Next.js's `VITE_`: when `pnpm dev` starts it reads `.env.local` and puts every variable with that prefix into the code the browser receives; anything else in the file stays on the server. The template's `.gitignore` skips `.env*`, so every teammate creates their own `.env.local`; this one holds an address, not a secret. The `throw` line makes a forgotten `.env.local` fail with a clear message instead of sending requests to `undefined/tasks`. `@/lib/types` uses the import alias from Step 1: `@/` means `src/`, so an import stays the same when a file moves. Your API still trusts only `http://localhost:5173`, so a page on port 3000 would be blocked by CORS (week 5, Step 1).
```bash
cp ~/projects/tasks-web/src/types.ts ~/projects/tasks-web/src/tasks.ts src/lib/
echo 'NEXT_PUBLIC_API_URL=http://127.0.0.1:8000' > .env.local
```
Create `src/lib/api.ts`:
```ts
import type { Task, TaskChanges, TokenResponse, User } from "@/lib/types";

const BASE_URL = process.env.NEXT_PUBLIC_API_URL;
if (BASE_URL === undefined) throw new Error("NEXT_PUBLIC_API_URL is missing: create .env.local");

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
export function getTask(token: string, id: number): Promise<Task> {
  return request<Task>(`/tasks/${id}`, {}, token);
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
Stop `pnpm dev` with Ctrl+C and start it again so that it reads `.env.local`. Then, in `~/projects/tasks-api/main.py`, change the `allow_origins` line to `allow_origins=["http://localhost:5173", "http://localhost:3000"],`; the API reloads by itself.
Check: `pnpm exec tsc --noEmit` prints nothing. `curl -i localhost:8000/hello -H 'Origin: http://localhost:3000'` shows the header `access-control-allow-origin: http://localhost:3000`. In `tasks-api`, `uv run pytest` still prints `9 passed`. Commit in both projects.

### Step 5: The session and the login page
Goal: `/login` shows the week 5 form, logs you in through your API, and sends you to `/tasks`. Before you run this, understand: **hydration** is what happens after the server's HTML arrives: the browser runs the same components and attaches them to that HTML, and the first render in the browser must produce exactly what the server produced, or React reports a hydration error. Week 5's `App` read `localStorage` during the first render; here the first render also runs on the server, where `localStorage` does not exist, so the read moves into `useEffect`, which runs only in the browser and only after that first render. `ready` tells a page whether that read has happened yet. The lint rule `react-hooks/set-state-in-effect` dislikes setting state inside an effect, because it always causes a second render; here the second render is the whole point (the first one matches the server, the second adds what only the browser knows), so the comment above that line turns the rule off for exactly one line. A **custom hook** is a function named `use...` that calls other hooks; every component that calls `useSession()` gets its own copy of the state. `useRouter` from `next/navigation` changes the page from code: `router.push("/tasks")` is a click on a link. The login page is the week 5 `Login` component with four changes: `"use client"` at the top, shadcn's `Button` and `Input` instead of plain tags, Tailwind classes instead of the `.login` rules, and no `onLogin` prop: it saves the session and goes to `/tasks` itself. A page may be a Client Component; it is then rendered once on the server and again in the browser. First make sure Ann exists, because `pytest` empties the tables: `curl -i -X POST localhost:8000/auth/register -H 'content-type: application/json' -d '{"email":"ann@example.com","password":"correct horse battery"}'` answers `201`, or `409` if she is still there. Create `src/lib/session.ts`:
```ts
"use client";

import { useEffect, useState } from "react";
import type { Session } from "@/lib/types";

const STORAGE_KEY = "tasks-next-session";

export function useSession() {
  const [session, setSession] = useState<Session | null>(null);
  const [ready, setReady] = useState(false);

  useEffect(() => {
    // Runs in the browser only, after the first render. The server has no localStorage.
    const saved = localStorage.getItem(STORAGE_KEY);
    // eslint-disable-next-line react-hooks/set-state-in-effect
    if (saved !== null) setSession(JSON.parse(saved) as Session);
    setReady(true);
  }, []);

  function saveSession(next: Session) {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(next));
    setSession(next);
  }
  function clearSession() {
    localStorage.removeItem(STORAGE_KEY);
    setSession(null);
  }
  return { session, ready, saveSession, clearSession };
}
```
Replace `src/app/login/page.tsx`:
```tsx
"use client";

import { useRouter } from "next/navigation";
import { useState, type FormEvent } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { ApiError, login, me } from "@/lib/api";
import { useSession } from "@/lib/session";

export default function LoginPage() {
  const router = useRouter();
  const { saveSession } = useSession();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  async function handleSubmit(event: FormEvent) {
    event.preventDefault();
    setError("");
    try {
      const { access_token } = await login(email, password);
      const user = await me(access_token);
      saveSession({ token: access_token, user });
      router.push("/tasks"); // like clicking a link to /tasks
    } catch (err) {
      setError(err instanceof ApiError ? err.message : "could not reach the API");
    }
  }

  return (
    <form className="mx-auto flex max-w-sm flex-col gap-3" onSubmit={handleSubmit}>
      <h1>Log in</h1>
      <label>Email<Input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required /></label>
      <label>Password<Input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required /></label>
      <Button type="submit">Log in</Button>
      {error !== "" && <p className="text-destructive">{error}</p>}
    </form>
  );
}
```
Check: `/login` shows the form. A wrong password shows `wrong email or password` in red, straight from your API's `401`. The right one changes the address to `/tasks`, which still says Your tasks (Step 6 fills it). In the developer tools, Local Storage (Chrome: Application tab; Firefox: Storage tab) has the key `tasks-next-session`. `curl -s localhost:3000/login | grep -o 'type="email"'` prints `type="email"`: the server rendered the form before any JavaScript ran. Commit.

### Step 6: The tasks page
Goal: `/tasks` is the week 5 task list with a header that shows who is logged in, and it sends a visitor who is not logged in to `/login`. Before you run this, understand: the redirect lives in the effect, not in the render: a component must only compute what to show, and changing the page is a side effect. The effect waits for `ready`, because before that the session is unknown, not missing. `router.replace` instead of `push` means the Back button does not return to a page you were not allowed to see. Everything else is week 5 code: the `LoadState`, the effect that loads the list, and the three handlers. Two small differences: the token comes from the hook instead of a prop, and shadcn's `Checkbox` reports a click through `onCheckedChange`. After an hour the token expires (week 4), the API answers `401`, and the page shows `not authenticated` in red: log in again. `TaskList` is a Client Component; the page that renders it stays a Server Component. Create `src/components/TaskList.tsx`:
```tsx
"use client";

import { useRouter } from "next/navigation";
import { useEffect, useState, type FormEvent } from "react";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Input } from "@/components/ui/input";
import { createTask, deleteTask, listTasks, updateTask } from "@/lib/api";
import { useSession } from "@/lib/session";
import { countOpen } from "@/lib/tasks";
import type { LoadState, Task } from "@/lib/types";

export function TaskList() {
  const router = useRouter();
  const { session, ready, clearSession } = useSession();
  const [state, setState] = useState<LoadState<Task[]>>({ status: "loading" });
  const [title, setTitle] = useState("");

  useEffect(() => {
    if (!ready) return; // the browser has not read localStorage yet
    if (session === null) {
      router.replace("/login"); // not logged in: go to the login page instead
      return;
    }
    listTasks(session.token)
      .then((tasks) => setState({ status: "ready", data: tasks }))
      .catch((err: unknown) => setState({ status: "error", message: err instanceof Error ? err.message : String(err) }));
  }, [ready, session, router]);

  if (session === null || state.status === "loading") return <p>Loading...</p>;
  if (state.status === "error") return <p className="text-destructive">{state.message}</p>;
  const { token } = session;
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
      <header className="mb-4 flex items-center gap-2">
        <h1 className="flex-1">Tasks</h1>
        <span className="text-sm text-muted-foreground">{session.user.email}</span>
        <Button variant="outline" size="sm" onClick={clearSession}>Log out</Button>
      </header>
      <form className="flex gap-2" onSubmit={handleAdd}>
        <Input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="New task" required />
        <Button type="submit">Add</Button>
      </form>
      <p className="my-3 text-sm text-muted-foreground">{countOpen(tasks)} open</p>
      <ul className="flex flex-col gap-2">
        {tasks.map((task) => (
          <li key={task.id} className="flex items-center gap-3 rounded-md border p-3">
            <Checkbox checked={task.done} onCheckedChange={() => handleToggle(task)} />
            <span className={task.done ? "flex-1 text-muted-foreground line-through" : "flex-1"}>{task.title}</span>
            <Button variant="outline" size="sm" onClick={() => handleDelete(task.id)}>Delete</Button>
          </li>
        ))}
      </ul>
    </>
  );
}
```
Replace `src/app/tasks/page.tsx` with two lines: `import { TaskList } from "@/components/TaskList";` and `export default function TasksPage() { return <TaskList />; }`.
Check: after login you see `Loading...` for a moment, then the header with `ann@example.com` and Log out, the form, the count and the list. Add `Buy milk` and `Learn Next`, tick one, delete one: the same behaviour as week 5, now on port 3000, and the API terminal shows the `GET`, `POST`, `PATCH` and `DELETE` lines with `OPTIONS` preflights between them. Refresh: still logged in. Click Log out: you are on `/login`; type `localhost:3000/tasks` in the address bar: you land on `/login` again. Commit.

### Step 7: A page for each task
Goal: `/tasks/12` shows task 12, and `/tasks/999999` shows a Task not found page. Before you run this, understand: a folder named `[id]` is a **dynamic segment**: `src/app/tasks/[id]/page.tsx` answers `/tasks/12`, `/tasks/13` and any other value, and `useParams()` from `next/navigation` gives you `{ id: "12" }`, always as text. **`loading.tsx`** next to a page is what Next.js shows while that page's code and content are still on the way; you see it for a moment on a slow connection. **`not-found.tsx`** is what Next.js shows when the page calls `notFound()`. Our page is a Client Component that asks the API after it appears, so the HTTP answer for `/tasks/999999` is still `200`, and the Task not found page replaces the content a moment later; a page that asks the API on the server and answers a real `404` comes later in the internship. First make every title in the list a link: in `src/components/TaskList.tsx`, add `import Link from "next/link";` above the `useRouter` import, and on the line with `{task.title}` change `<span className=` to ``<Link href={`/tasks/${task.id}`} className=`` and its `</span>` to `</Link>`; the `<span>` in the header stays. Then create `src/app/tasks/[id]/page.tsx`:
```tsx
"use client";

import Link from "next/link";
import { notFound, useParams } from "next/navigation";
import { useEffect, useState } from "react";
import { ApiError, getTask } from "@/lib/api";
import { useSession } from "@/lib/session";
import type { Task } from "@/lib/types";

export default function TaskPage() {
  const { id } = useParams<{ id: string }>();
  const { session, ready } = useSession();
  const [task, setTask] = useState<Task | null>(null);
  const [error, setError] = useState<unknown>(null);

  useEffect(() => {
    if (session !== null) getTask(session.token, Number(id)).then(setTask).catch(setError);
  }, [session, id]);

  if (error instanceof ApiError && error.status === 404) notFound(); // shows not-found.tsx
  if (error !== null) return <p className="text-destructive">{error instanceof Error ? error.message : String(error)}</p>;
  if (ready && session === null) return <p>Please <Link href="/login" className="underline">log in</Link> first.</p>;
  if (task === null) return <p>Loading task...</p>;
  return (
    <>
      <h1>{task.title}</h1>
      <p className="my-2 text-muted-foreground">{task.done ? "Done" : "Open"}, task number {task.id}</p>
      <Link href="/tasks" className="underline">Back to the list</Link>
    </>
  );
}
```
Create `src/app/tasks/[id]/loading.tsx` with the single line `export default function Loading() { return <p>Loading page...</p>; }`, and `src/app/tasks/[id]/not-found.tsx` with two lines: `import Link from "next/link";` and `export default function NotFound() { return <><h1>Task not found</h1><Link href="/tasks" className="underline">Back to the list</Link></>; }`.
Check: on `/tasks`, click `Buy milk`: the address becomes `/tasks/12` (your id), the page shows the title, `Open, task number 12` and a link back; the browser's Back button returns to the list. Type `localhost:3000/tasks/999999`: Task not found, with the link. Log out and open a task address: `Please log in first.` Commit.

### Step 8: The production build
Goal: the app runs from a production build. Before you run this, understand: `pnpm dev` compiles a page when you open it and checks nothing in advance. `next build` compiles every page once into `.next/` (git ignores it), runs the type checker, and prints a table. A page marked `○` (static) was rendered to HTML now, at build time, because its HTML needs nothing from the request: `/`, `/login`, `/tasks` and the not-found page; a server can hand them out as files. `/tasks` is on that list even though the list is personal: its server HTML is only `Loading...`, the browser adds the data. A page marked `ƒ` (dynamic) is rendered when a request arrives, because Next.js cannot know every `id` in advance. `next start` serves the build on port 3000, so stop `pnpm dev` first. `NEXT_PUBLIC_` values are written into the files at build time, which is why `.env.local` must exist before `pnpm build`.
```bash
pnpm lint
pnpm build
pnpm start
```
Check: `pnpm build` prints `✓ Compiled successfully`, `Running TypeScript ...` and then the table `Route (app)` with five lines, `○ /`, `○ /_not-found`, `○ /login`, `○ /tasks` and `ƒ /tasks/[id]`, followed by the legend `○  (Static)   prerendered as static content` and `ƒ  (Dynamic)  server-rendered on demand`. `pnpm start` prints `- Local:         http://localhost:3000`, and the app works exactly as in Step 7. `curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/` prints `200`. Nothing new to commit; stop `pnpm start` with Ctrl+C and go back to `pnpm dev` when you continue working.

## Read and watch
1. [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components) (Next.js docs): when a component runs on the server, when in the browser, and what `"use client"` really marks; read up to "Examples" before Step 3, and the rest after Step 6.
2. [Environment Variables](https://nextjs.org/docs/app/guides/environment-variables) (Next.js docs): what `NEXT_PUBLIC_` does and which files Next.js reads; read the section "Bundling Environment Variables for the Browser" before Step 4.
3. [Theming](https://ui.shadcn.com/docs/theming) (shadcn/ui docs): how the variables that `init` wrote into `globals.css` become classes like `bg-primary` and `text-muted-foreground`; skim after Step 2.
4. Video: [Next.js Explained: Partial Prerendering (PPR)](https://www.youtube.com/watch?v=MTcPrTIBkpA) (Delba, 11 min): why a page is split into a part rendered at build time and a part rendered per request; watch after Step 8 with your route table next to it. The feature itself belongs to a later week.

## Turn in
- Push `tasks-next` to a new GitHub repository with one commit per step, and push the Step 4 commit of `tasks-api`.
- Replace the template's `README.md` in `tasks-next`: how to run (start the database and the API in `tasks-api`, then `pnpm install`, create `.env.local` with its one line, `pnpm dev`) and how to build (`pnpm build`, `pnpm start`).
- Add `notes/week-06.md` with your answers to the self-check below, in your own words, and two screenshots: `/tasks` with a few tasks, and one task's page.
- Friday demo (10 minutes): fresh clone of both projects, start the database and the API, `pnpm install`, create `.env.local`, `pnpm dev`, log in with a wrong and then the right password, add, tick and delete a task, open a task's page and `/tasks/999999`, refresh and stay logged in, log out, show the `curl ... /login` heading, then `pnpm build` and explain the `○` and `ƒ` lines.

## Self-check
1. `curl localhost:3000/login` shows the heading. In week 5, `curl localhost:5173` showed only an empty `<div id="root">`. What changed?
2. What does `"use client"` change about a file, and why can the root layout do without it?
3. Why does `useSession` read `localStorage` inside `useEffect` and not in the `useState` initializer, as week 5's `App` did?
4. `/tasks/999999` shows Task not found, but the HTTP status is `200`. Why?
5. In the build table, why is `/tasks` static (`○`) while `/tasks/[id]` is dynamic (`ƒ`)?

### Answers
1. Next.js renders the page on the server and sends finished HTML. Vite sent an almost empty page plus JavaScript, and the browser drew everything.
2. The file and everything it imports also run in the browser, so it may use state, effects and event handlers. The layout only renders links and `children`, which needs none of that.
3. The first render also runs on the server, where `localStorage` does not exist, and the browser's first render must match the server's HTML (hydration). An effect runs only in the browser, after that first render.
4. The page is a Client Component: the server answered `200` with `Loading task...`, and only then did the browser ask the API. `notFound()` swapped in `not-found.tsx` without a new HTTP response.
5. The server HTML of `/tasks` is the same for everyone (`Loading...`), so it can be made once at build time. For `/tasks/[id]`, Next.js cannot know which ids exist, so it renders that route when a request arrives.

## If you finish early
- Pages for projects: `src/app/projects/page.tsx` and `src/app/projects/[id]/page.tsx` on the week 3 endpoints `GET /projects` and `GET /projects/{id}/tasks`; add a `Project` type and `listProjects` and `listProjectTasks` to `api.ts` first.
- For your own project: list your screens; which are pages, which share a layout; which need the token (Client Components, for now) and which can stay Server Components.
