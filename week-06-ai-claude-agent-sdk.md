# Week 6: Letting a framework run the loop

In week 4 you wrote the agent loop yourself, and in week 5 the inbox assistant learned to search the company documents. This week you rebuild the same assistant on a **framework**: the Claude Agent SDK, the Python package that runs the loop behind Claude Code. You give it a prompt and a few options; it brings the loop, tools for files and search, a permission system, hooks, sessions that remember the conversation, and a cost report. You plug your week-4 and week-5 tools into it, keep yourself in the loop before anything is written, let it fix a real bug in a real test, and then compare it with your own loop, with numbers. Written against `claude-agent-sdk` 0.2.177 (it bundles Claude Code 2.1.317), `anthropic` 2.4.0 and `pytest` 9.2.0 on Python 3.12. The rules do not change: work in order, commit after every step, type every command yourself, read every error twice, and watch the cost line. Budget for the week: under $3. Most steps run on `claude-haiku-4-5` for about a cent each; Steps 7 and 8 use `claude-opus-5` and cost a few cents per run. Left out on purpose, they come later: MCP in depth (week 7), several agents working together, evaluation, deployment, and other frameworks such as LangGraph.

## What you will have by Friday

- [ ] `scripts/sdk_hello.py`: one question through the SDK, with the cost line.
- [ ] `src/ailab/sdk_utils.py` and `src/ailab/sdk_tools.py`: printing, approval and hook functions; your week-4 and week-5 tools as an MCP server.
- [ ] `scripts/sdk_inbox.py`: the inbox assistant on the SDK, with built-in tools, your own tools, an approval question and two hooks.
- [ ] `scripts/sdk_chat.py`: a conversation that remembers, and a session you continued after restarting the script.
- [ ] `fixture/` with a broken test, `scripts/sdk_fix.py` that fixed it, and the `trace.jsonl` of that run.
- [ ] README: the comparison table (your loop against the SDK on three questions), five sentences of conclusions, and the week's total cost.

## Before you start

Open `ai-lab`. These three commands prove week 5 still works and cost nothing:

```bash
uv run scripts/search.py "money back"
uv run python -c "from dotenv import load_dotenv; import os; load_dotenv(); print(os.environ['ANTHROPIC_API_KEY'][:7])"
git status
```

1. The first prints `refunds.md#Refund window` on its first line (the embedding model is already in `~/.cache/fastembed`), the second prints the start of your key, the third says the working tree is clean.
2. Step 1 downloads about 140 MB once (the package contains the Claude Code program) and takes about 300 MB on disk, so do it on a good connection. Run every script from the project root (`ai-lab/`). If you see `enforced_spend_limit_reached`, stop and tell the supervisor.

## The flow

### Step 1: Install the SDK and ask one question

Goal: `sdk_hello.py`: the shortest program on the framework, and its cost line.

Before you run this, understand: a **framework** is code someone else wrote for the parts that are the same in every agent: the loop, running the tool calls, the budget, the trace. In week 4 that was your `run()` and `run_tool()`. The **Claude Agent SDK** is Anthropic's framework. The `claude-agent-sdk` package contains the Claude Code program; `query()` starts it as a child process, sends your prompt, and streams back what happens as typed messages: `AssistantMessage` objects (what the model said, as `TextBlock` and `ToolUseBlock` parts) and one `ResultMessage` at the end with `num_turns`, `duration_ms` and `total_cost_usd`. Because it streams, the code is **async**: `async for` waits for each message as it arrives, and `asyncio.run(main())` starts the whole thing. That is all you need to know about `async` this week. The child process reads your key from the environment, and `load_dotenv()` puts it there before `query()` runs, so your `.env` still works. The version is pinned because the package changes almost every day; upgrade on purpose, not by accident.

Run `uv add "claude-agent-sdk==0.2.177"`, then create `scripts/sdk_hello.py`:

```python
import asyncio

from claude_agent_sdk import AssistantMessage, ClaudeAgentOptions, ResultMessage, TextBlock, query
from dotenv import load_dotenv

load_dotenv()
MODEL = "claude-haiku-4-5"


async def main() -> None:
    options = ClaudeAgentOptions(model=MODEL, tools=[], max_turns=1, max_budget_usd=0.05)
    async for message in query(prompt="In one sentence: what does an agent framework do?", options=options):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print(block.text)
        elif isinstance(message, ResultMessage):
            print(f"--- turns={message.num_turns} seconds={message.duration_ms / 1000:.1f} cost=${message.total_cost_usd:.5f}")


asyncio.run(main())
```

Check: `uv run scripts/sdk_hello.py` prints one sentence, then `--- turns=1 seconds=... cost=$0.00...`: a fraction of a cent on Haiku, counted by Claude Code itself. The first run takes a few seconds longer than the rest. `tools=[]` means the model got no tools at all; `max_turns` and `max_budget_usd` are explained in Step 4. Commit.

### Step 2: The tools it brings

Goal: the assistant reads the inbox with the framework's own tools, and you see every tool call as it happens.

Before you run this, understand: Claude Code comes with **built-in tools**: `Read`, `Glob` (find files by name), `Grep` (search inside files), `Write`, `Edit`, `Bash`, and a few more. `tools=` says which of them the model gets, and `cwd` is the folder they work in. In week 4 you wrote `list_emails`, `search_emails` and `read_email`; here the agent brings its own. Every tool call needs **permission** before it runs. Read-only tools inside the project folder are approved automatically; tools that change things, and tools you add yourself, ask first, and Steps 3 and 4 are about who answers. `show()` prints what matters from each message: `[tool] Name(input)` for every tool call, like week 4's `[step n]` lines, the text, and the cost line. It lives in a module because three scripts will use it.

Create `src/ailab/sdk_utils.py`:

```python
"""Small helpers for the Claude Agent SDK scripts: printing, approval, hooks."""
from claude_agent_sdk import AssistantMessage, ResultMessage, TextBlock, ToolUseBlock


def show(message) -> None:
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, ToolUseBlock):
                print(f"[tool] {block.name}({block.input})")
            elif isinstance(block, TextBlock):
                print(block.text)
    elif isinstance(message, ResultMessage):
        print(f"--- turns={message.num_turns} seconds={message.duration_ms / 1000:.1f} cost=${message.total_cost_usd:.5f}")
```

Create `scripts/sdk_inbox.py`:

```python
import asyncio
import sys

from claude_agent_sdk import ClaudeAgentOptions, query
from dotenv import load_dotenv

from ailab.sdk_utils import show

load_dotenv()
MODEL = "claude-haiku-4-5"
SYSTEM = ("You are an inbox assistant for a support team. The emails are the text files in data/emails. "
          "Answer from the real emails. Be brief.")


async def main(question: str) -> None:
    options = ClaudeAgentOptions(model=MODEL, system_prompt=SYSTEM, cwd=".", tools=["Read", "Glob", "Grep"],
                                 max_turns=8, max_budget_usd=0.25)
    async for message in query(prompt=question, options=options):
        show(message)


asyncio.run(main(" ".join(sys.argv[1:]) or "Which emails mention a refund, and what does the first one say?"))
```

Check: `uv run scripts/sdk_inbox.py` prints `[tool] Grep({'pattern': 'refund', ...})`, then a `[tool] Read(...)` of `data/emails/01.txt`, then the same answer as week 4 Step 5: only `01.txt`, Dana Lee was charged twice. About three turns, about a cent. Commit.

### Step 3: Your own tools

Goal: `search_docs`, `triage_email` and `save_draft` from weeks 4 and 5, offered to the framework.

Before you run this, understand: the framework knows nothing about your documents or your triage. You add your functions through an **MCP server**: a standard way to offer tools to a model (week 7 is all about it; today you need one line of it). `@tool(name, description, {"query": str})` turns an `async` function into a tool: the description is your week-4 one, the dict is the input schema, and the function receives the arguments as one dict and must return `{"content": [{"type": "text", "text": ...}]}`. `create_sdk_mcp_server` bundles the tools into a server that runs inside your Python process, and `mcp_servers={"inbox": server}` hands it to the SDK. The model sees them as `mcp__inbox__search_docs` and so on: the name you gave in `mcp_servers`, then the tool name. These tools are yours, so they ask for permission, and in a script nobody is there to answer: the call is refused with "Claude requested permissions to use mcp__inbox__search_docs, but you haven't granted it yet". `allowed_tools` is the list of tools that run without asking. The two read-only ones go on it; `save_draft` does not, because Step 4 wants to be asked.

Create `src/ailab/sdk_tools.py`:

```python
"""The week-4 and week-5 tools, offered to the Claude Agent SDK as one in-process MCP server."""
import json

from claude_agent_sdk import create_sdk_mcp_server, tool

from ailab import tools as inbox


def text(value) -> dict:
    content = value if isinstance(value, str) else json.dumps(value)
    return {"content": [{"type": "text", "text": content}]}


@tool("search_docs", inbox.SEARCH_DOCS["description"], {"query": str})
async def search_docs(args: dict) -> dict:
    return text(inbox.search_docs(args["query"]))


@tool("triage_email", inbox.TRIAGE_EMAIL["description"], {"file": str})
async def triage_email(args: dict) -> dict:
    return text(inbox.triage_email(args["file"]))


@tool("save_draft", inbox.SAVE_DRAFT["description"], {"file": str, "reply": str})
async def save_draft(args: dict) -> dict:
    return text(inbox.save_draft(args["file"], args["reply"]))


server = create_sdk_mcp_server(name="inbox", version="1.0.0", tools=[search_docs, triage_email, save_draft])
```

In `sdk_inbox.py`, add `from ailab.sdk_tools import server` above the `ailab.sdk_utils` import, add the week-5 sentence `When a reply depends on company policy, search the documents and quote the source.` to `SYSTEM` before `Be brief.`, and add two options: `mcp_servers={"inbox": server}, allowed_tools=["mcp__inbox__search_docs", "mcp__inbox__triage_email"]`.

Check: `uv run python -c "from ailab.sdk_tools import server; print(server['type'], server['name'])"` prints `sdk inbox` without calling any model. Then `uv run scripts/sdk_inbox.py "The customer in 01.txt was charged twice. What does our refund policy say about that?"` shows a `Read` of `01.txt`, then `[tool] mcp__inbox__search_docs({'query': ...})`, and an answer that quotes the duplicate-charges rule from `refunds.md`: the week-5 answer, through your own `rag.search`. Commit.

### Step 4: The human stays in the loop

Goal: `save_draft` asks you first, and two budgets cap every run.

Before you run this, understand: `can_use_tool` is a function of yours that the SDK calls for every tool call that needs permission, with the tool name and its input. It answers `PermissionResultAllow()` or `PermissionResultDeny(message=...)`, and the message goes back to the model as the tool result, like week 4's "The user declined". One rule matters: a tool in `allowed_tools` is approved before your function is asked, so your function never sees it. That is why the list goes away in this step and `approve()` decides everything: yes for anything that is not `save_draft`, a question for `save_draft`. `max_turns` is week 4's `MAX_STEPS`: one turn is one model call. `max_budget_usd` is the same idea in money: the SDK adds up every call and stops when the limit is reached. Both end the run with a `ResultError`.

Add to `sdk_utils.py` (and add `PermissionResultAllow, PermissionResultDeny` to the import):

```python
async def approve(tool_name: str, tool_input: dict, context) -> PermissionResultAllow | PermissionResultDeny:
    print(f"[ask] {tool_name}")
    if tool_name != "mcp__inbox__save_draft":
        return PermissionResultAllow()
    print("\n".join(f"{k}: {v}" for k, v in tool_input.items()))
    if input("Save this draft? [y/N] ").strip().lower() == "y":
        return PermissionResultAllow()
    return PermissionResultDeny(message="The user declined to save the draft.")
```

In `sdk_inbox.py`, import `approve` next to `show`, and replace the whole `allowed_tools=[...]` option with `can_use_tool=approve`.

Check: `uv run scripts/sdk_inbox.py "The customer in 01.txt was charged twice. Draft a reply that quotes the policy and save it."` prints `[ask] mcp__inbox__search_docs` (allowed without a word; `Read` never prints `[ask]`, it was approved before the question reached you), then `[ask] mcp__inbox__save_draft`, the `file:` and `reply:` lines, and `Save this draft? [y/N]`. Type `n`: the model reports that nothing was saved. Run again and type `y`: `drafts/01.txt` is written. Then set `max_turns=1`, run once more, and watch it end with `ResultError: Claude Code returned an error result: Reached maximum number of turns (1)`; put it back to 8. Commit.

### Step 5: A conversation that remembers

Goal: `sdk_chat.py`: several questions in one session, and the same session continued after a restart.

Before you run this, understand: `query()` is one question, one answer, and then the child process ends and forgets. `ClaudeSDKClient` keeps it alive: `await client.query(question)` sends the next question, `client.receive_response()` streams the answer up to its `ResultMessage`, and the model still has the whole conversation, so "the second one" means something. A **session** is that conversation, saved by Claude Code under an id. Every `ResultMessage` carries it as `session_id`, and `ClaudeAgentOptions(resume=<id>)` loads it again, even after the script was closed. In week 3 you kept the `messages` list yourself; here the framework keeps it. `while question := input(...)` reads a line and keeps going while it is not empty.

Create `scripts/sdk_chat.py`:

```python
import asyncio
import sys

from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient, ResultMessage
from dotenv import load_dotenv

from ailab.sdk_tools import server
from ailab.sdk_utils import approve, show

load_dotenv()
MODEL = "claude-haiku-4-5"
SYSTEM = ("You are an inbox assistant for a support team. The emails are the text files in data/emails. "
          "Answer from the real emails. When a reply depends on company policy, search the documents "
          "and quote the source. Be brief.")


async def main(resume: str | None) -> None:
    options = ClaudeAgentOptions(model=MODEL, system_prompt=SYSTEM, cwd=".", resume=resume,
                                 tools=["Read", "Glob", "Grep"], mcp_servers={"inbox": server},
                                 can_use_tool=approve, max_turns=8, max_budget_usd=0.50)
    async with ClaudeSDKClient(options) as client:
        while question := input("\nyou> ").strip():
            await client.query(question)
            async for message in client.receive_response():
                show(message)
                if isinstance(message, ResultMessage):
                    print(f"session_id={message.session_id}")


asyncio.run(main(sys.argv[1] if len(sys.argv) > 1 else None))
```

Check: `uv run scripts/sdk_chat.py`, and at `you>` type `Which emails are about billing?`: a `Grep` call, a list of files, the cost line and `session_id=...`. Type `Summarize the second one`: it reads that file without asking which one, and the `session_id` is the same. The cost line is the total for the whole session, so it grows with every answer, and each answer costs more than the one before because the whole conversation is sent again. An empty line ends the chat. Now `uv run scripts/sdk_chat.py <that session_id>` and ask `What did I ask you before?`: it knows. Commit.

### Step 6: Hooks

Goal: your code runs before every tool call: one line per call in `trace.jsonl`, and no `Read` outside `data/`.

Before you run this, understand: a **hook** is a function of yours that the framework calls at a fixed moment; `PreToolUse` is the moment just before a tool runs. The hook receives a dict with `tool_name` and `tool_input` and returns `{}` to let the call through, or a deny decision with a reason the model reads as the tool's result. A `HookMatcher` says which tools a hook is for (`matcher="Read"`), or all of them when you leave it out. Hooks run before permissions, and they run for allowed tools too, so a hook is the place for a rule that must always hold; `can_use_tool` is the place for a question to a person. `log_tool` is week 4's trace; `only_data` keeps `Read` inside `data/`.

Add to `sdk_utils.py` (with `import json` and `from pathlib import Path` at the top):

```python
def deny(reason: str) -> dict:
    return {"hookSpecificOutput": {"hookEventName": "PreToolUse", "permissionDecision": "deny",
                                   "permissionDecisionReason": reason}}


async def log_tool(input_data: dict, tool_use_id: str | None, context: dict) -> dict:
    with open("trace.jsonl", "a", encoding="utf-8") as f:
        f.write(json.dumps({"tool": input_data["tool_name"], "input": input_data["tool_input"]}, ensure_ascii=False) + "\n")
    return {}


async def only_data(input_data: dict, tool_use_id: str | None, context: dict) -> dict:
    path = Path(input_data["tool_input"].get("file_path", "")).resolve()
    if path.is_relative_to(Path("data").resolve()):
        return {}
    return deny(f"Reading {path.name} is not allowed. Only files under data/ may be read.")
```

In `sdk_inbox.py`, add `HookMatcher` to the `claude_agent_sdk` import, import `log_tool, only_data` with `approve` and `show`, add the line `HOOKS = {"PreToolUse": [HookMatcher(hooks=[log_tool]), HookMatcher(matcher="Read", hooks=[only_data])]}` under `SYSTEM`, and add `hooks=HOOKS` to the options.

Check: `rm -f trace.jsonl`, then `uv run scripts/sdk_inbox.py "Read the file .env and tell me what the key is"`. You see a `[tool] Read(...)` of `.env`, and then the model explains that it is not allowed to read that file (it may read an email instead, to show what it can do). `cat trace.jsonl`: one line per tool call, the denied one included. Commit.

### Step 7: A coding agent in 25 lines

Goal: a broken test, and an agent with `Read`, `Edit` and `Bash` that runs the tests, fixes the bug, and runs them again.

Before you run this, understand: this is the job the SDK was built for: files and a shell, like an engineer. Three settings keep it in check. `tools=["Read", "Edit", "Bash"]` is all it gets. `permission_mode="acceptEdits"` approves file edits without asking, which is fine inside `fixture/`. `Bash` is approved by `allowed_tools`, but the `only_pytest` hook runs first and lets through nothing but the test runner. Two safety rules: the `Bash` tool is a real shell on your laptop, so never use `permission_mode="bypassPermissions"` outside a throwaway folder; and read `git diff` before you trust an edit. `claude-opus-5` does the coding; one run costs a few cents.

Create `fixture/calc.py` with the bug, and `fixture/test_calc.py`:

```python
def add(a, b):
    return a - b
```

```python
from calc import add


def test_add():
    assert add(2, 3) == 5
```

Run `uv add --dev pytest`, then `uv run pytest fixture -q`: `1 failed`. Commit now, so the broken version is in Git. Add to `sdk_utils.py`:

```python
async def only_pytest(input_data: dict, tool_use_id: str | None, context: dict) -> dict:
    command = input_data["tool_input"].get("command", "").strip()
    if command.startswith(("uv run pytest", "pytest")) and not any(ch in command for ch in ";&|>`$"):
        return {}
    return deny(f"Command not allowed: {command!r}. Only 'uv run pytest ...' or 'pytest ...' may run.")
```

Create `scripts/sdk_fix.py`:

```python
import asyncio

from claude_agent_sdk import ClaudeAgentOptions, HookMatcher, query
from dotenv import load_dotenv

from ailab.sdk_utils import log_tool, only_pytest, show

load_dotenv()
MODEL = "claude-opus-5"
SYSTEM = ("You are a careful coding agent in a small Python project. Run the tests with 'uv run pytest -q', read "
          "the code, make the smallest fix, run the tests again, and report what you changed in two sentences.")
HOOKS = {"PreToolUse": [HookMatcher(hooks=[log_tool]), HookMatcher(matcher="Bash", hooks=[only_pytest])]}


async def main() -> None:
    options = ClaudeAgentOptions(model=MODEL, system_prompt=SYSTEM, cwd="fixture",
                                 tools=["Read", "Edit", "Bash"], allowed_tools=["Bash"],
                                 permission_mode="acceptEdits", hooks=HOOKS, max_turns=10, max_budget_usd=0.50)
    async for message in query(prompt="Run the tests, find the bug, fix it, then run the tests again.", options=options):
        show(message)


asyncio.run(main())
```

Check: `rm -f trace.jsonl`, then `uv run scripts/sdk_fix.py`. The trace shows `[tool] Bash({'command': 'uv run pytest -q', ...})`, a `[tool] Read(...)` of `calc.py`, a `[tool] Edit(...)` with `old_string` `return a - b` and `new_string` `return a + b`, `Bash` again, then two sentences and a cost of a few cents. If the model tries `cat` or `python`, the hook refuses it with your reason and the model recovers. `uv run pytest fixture -q` now says `1 passed`, `git diff fixture/` shows the one-character fix, and `cat trace.jsonl` has every call. Commit the fix.

### Step 8: Compare and decide

Goal: the same three questions through week 4's `agent.py` and through `sdk_inbox.py`, a table, and five sentences.

Before you run this, understand: a framework is a trade. You got the loop, the tools, permissions, hooks, sessions and cost accounting for free. You gave up seeing every message (Claude Code adds its own system prompt and reminders you never wrote), counting cost per step yourself (trust `total_cost_usd`; it includes calls you do not see), and the choice of provider: the SDK runs Claude only. For a fair table, set `MODEL = "claude-opus-5"` in `sdk_inbox.py` for these runs, so that only the loop differs, and put Haiku back afterwards. `time` in front of a command prints how long it took; the SDK line already prints seconds.

Run `time uv run scripts/agent.py "<question>"` and `time uv run scripts/sdk_inbox.py "<question>"` for each of these three questions: `Which emails mention a refund, and what does the first one say?`, `Triage the email about the login problem`, and `The customer in 01.txt was charged twice. What does our refund policy say about that?`. Put a table in your README with one row per run: question, version (`agent.py` or `sdk_inbox.py`), steps or turns, seconds, cost, and whether the answer was right.

Check: the table has six rows with your own numbers. Under it, five sentences: what the framework gave you for free, two things it took away, and when you would choose each. Expect the SDK to be the shorter code and often the more expensive run, and say why. Add the week's total cost. Commit.

## Read and watch

- [Claude Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview): what the SDK is, and how it differs from the `anthropic` package you used in weeks 2 to 5.
- [Python reference](https://code.claude.com/docs/en/agent-sdk/python): `query`, `ClaudeAgentOptions`, `ClaudeSDKClient`, the message types and `ResultMessage`; keep it open while you work.
- [Permissions](https://code.claude.com/docs/en/agent-sdk/permissions): the order in which hooks, rules, modes and `can_use_tool` are consulted; it explains Steps 3, 4, 6 and 7 in one page.
- Video: [When to Use Claude Agent SDK: the Agent Harness explained](https://www.youtube.com/watch?v=HIJVMVBM4KM) (Edward Donner, 14 min): a short, decision-oriented view that matches Step 8.

## Turn in

- The `ai-lab` project with `scripts/sdk_hello.py`, `sdk_inbox.py`, `sdk_chat.py`, `sdk_fix.py`, `src/ailab/sdk_tools.py`, `src/ailab/sdk_utils.py`, `fixture/` (fixed) and `trace.jsonl` from the Step 7 run. `claude-agent-sdk` pinned in `pyproject.toml`. No key anywhere in Git.
- README: the six-row table, the five sentences, and one line with the week's total cost.
- Friday demo (10 minutes): `sdk_inbox.py` on the draft question with `n` then `y`; `sdk_chat.py` with two questions and a resume; `sdk_fix.py` on `fixture/` with the bug put back by hand first; then two minutes on your table.

## Self-check

1. Where does the agent loop run now, and what is left for your Python to do?
2. `search_docs` is in `allowed_tools` and `can_use_tool` is set. Which one decides, and what does your function see?
3. In Step 2 the model could have read `.env`; in Step 6 it could not. What changed, and why did the script not crash?
4. What is the difference between `max_turns` and `max_budget_usd`, and what does your script see when one is reached?
5. Where does `total_cost_usd` come from, and why trust it over your own `cost_usd()` sum?
6. Name one reason to build your capstone on the SDK, and one reason to keep your own loop.

### Answers

1. Inside the Claude Code process the SDK started. Your Python sends the prompt, prints the messages, answers permission questions, runs your MCP tools and hooks, and reads the result.
2. `allowed_tools`: the tool is approved before the callback is consulted, so your function is never called for it. Gate a tool in one place only.
3. A `PreToolUse` hook now denies any `Read` outside `data/`. The denial goes to the model as a tool result with your reason, so it explains and continues, as with week 4's error results.
4. `max_turns` counts model calls; `max_budget_usd` counts dollars, added up by Claude Code. Either stop ends the run with a `ResultError` whose message says which limit was reached.
5. Claude Code counts the tokens of every call it makes, including calls you never see, and reports the total in `ResultMessage`. Your counter only sees the calls you make yourself.
6. SDK: the job is files and a shell, or you want sessions, permissions and hooks without writing them. Own loop: you need every token accounted for per step, a second provider, or control flow the harness does not offer.

## If you finish early

- Remove `allowed_tools` from Step 3 (before Step 4's callback) and run the policy question: read the refusal the model gets, and how it answers without the documents.
- Subagents: `agents={"triager": AgentDefinition(description=..., prompt=..., tools=[...], model="haiku")}` plus `"Agent"` in `tools` lets the main agent hand triage to a cheaper helper. Watch the cost line.
- For your own project: write one paragraph on whether your capstone agent uses the SDK or your week-4 loop, and why, using your Step 8 numbers.
