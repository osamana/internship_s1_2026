# Week 4: Giving the model tools

In week 3 the model could only talk: you sent an email in, it sent a `Triage` back. This week you give it **tools**: small Python functions it can ask you to run. Over the same 20 emails you build an **inbox assistant**. It can list, read and search emails, triage one (with your week-3 code), and save a draft reply, and you approve before anything is written to disk. Written against `anthropic` 1.15.0; prices are the September 2026 numbers already in your `cost.py`. Every script still prints tokens and cost, and an agent run now adds them up over several calls. Two models only: `claude-opus-5` is the assistant, and `claude-haiku-4-5` works inside one tool.

## What you will have by Friday

- [ ] `src/ailab/tools.py`: six tool functions and their six definitions.
- [ ] `tool_once.py`: one tool call done by hand, in both directions.
- [ ] `agent.py`: the loop, with a step budget, error results, a trace file and an approval question.
- [ ] `trace.jsonl` from at least three runs, and `drafts/` with one saved reply.
- [ ] README: the week's total cost, and two sentences on one thing the assistant got wrong.
- [ ] `docs/proposal.md` in your new capstone repository, opened as a pull request.

## Before you start

1. Open `ai-lab`. `uv run scripts/triage_one.py data/emails/01.txt` must still work, and `triage.csv` from week 3 must exist. Nothing new to install.
2. Budget for the week: under $3. One agent run is 2 to 6 calls, so expect a few cents per run on Opus. If you see `enforced_spend_limit_reached`, stop and tell the supervisor.
3. Run every script from the project root (`ai-lab/`). The tools use paths like `data/emails/` and `triage.csv`.

## The flow

### Step 1: What a tool is, and the first tool

Before you run this, understand: a **tool** is a Python function you allow the model to ask for. The model never runs code. It reads your description of the tool and, when it needs it, writes a request: "call `list_emails` with these arguments". That request is a **tool call**. Your Python runs the function and sends the output back as a **tool result**. Then the model continues, now with real data. So a tool has two parts: the function, and a **tool definition**, a dict with `name`, `description` and `input_schema`. The `input_schema` is a JSON Schema (the same idea as week 3: which fields, which types, which are required). The description is the important part. It is how the model decides *when* to use the tool, so say what the tool does and when to call it.

Create `src/ailab/tools.py`:

```python
"""Tools the inbox assistant can ask to run. Each function has a definition the model reads."""
from pathlib import Path

EMAILS = Path("data/emails")


def list_emails() -> list[str]:
    return sorted(p.name for p in EMAILS.glob("*.txt"))


LIST_EMAILS = {
    "name": "list_emails",
    "description": "List the file names of all emails in the support inbox. Call this when the user asks how many emails there are or wants an overview.",
    "input_schema": {"type": "object", "properties": {}, "required": []},
}
```

Check: `uv run python -c "from ailab.tools import list_emails; print(list_emails())"` prints 20 file names. No model was involved yet.

### Step 2: Watch the model ask

Before you run this, understand: you pass `tools=[LIST_EMAILS]` to `client.messages.create`. If the model wants a tool, the reply has `stop_reason == "tool_use"`, and `content` contains one or more blocks with `type == "tool_use"`. Each such block has a `name` (which tool), an `input` (the arguments, already a Python dict) and an `id` (you need it in Step 3). The model may ask for several tools in one reply.

Create `scripts/tool_once.py`:

```python
from anthropic import Anthropic
from dotenv import load_dotenv

from ailab.cost import cost_usd
from ailab.tools import LIST_EMAILS

load_dotenv()
client = Anthropic()
MODEL = "claude-opus-5"
messages = [{"role": "user", "content": "How many emails are in the inbox?"}]

msg = client.messages.create(model=MODEL, max_tokens=1000, output_config={"effort": "low"},
                             tools=[LIST_EMAILS], messages=messages)
print("stop_reason =", msg.stop_reason)
for b in msg.content:
    if b.type == "tool_use":
        print(f"tool_use: name={b.name} input={b.input} id={b.id}")
u = msg.usage
print(f"--- in={u.input_tokens} out={u.output_tokens} cost=${cost_usd(MODEL, u.input_tokens, u.output_tokens):.5f}")
```

Check: `uv run scripts/tool_once.py` prints `stop_reason = tool_use`, then one line with `name=list_emails input={} id=toolu_...`. There is no answer yet: the model stopped and is waiting for you.

### Step 3: Send the result back

Before you run this, understand: the conversation is still the `messages` list from week 3, with two new shapes. First you append the model's turn exactly as you received it: `{"role": "assistant", "content": msg.content}`. Then you append a user turn whose content is a *list* of `tool_result` blocks: `{"type": "tool_result", "tool_use_id": <the block's id>, "content": <a string>}`. One result per `tool_use` block, matched by `id`, all in that one user message. Then you call the model again with the same `tools`.

Change the import to `from ailab.tools import LIST_EMAILS, list_emails` and add this to the end of `tool_once.py`:

```python
tool_block = next(b for b in msg.content if b.type == "tool_use")
result = "\n".join(list_emails())                       # you run the tool, not the model
messages.append({"role": "assistant", "content": msg.content})
messages.append({"role": "user", "content": [
    {"type": "tool_result", "tool_use_id": tool_block.id, "content": result}]})
final = client.messages.create(model=MODEL, max_tokens=1000, output_config={"effort": "low"},
                               tools=[LIST_EMAILS], messages=messages)
print("".join(b.text for b in final.content if b.type == "text"))
cost = sum(cost_usd(MODEL, m.usage.input_tokens, m.usage.output_tokens) for m in (msg, final))
print(f"--- calls=2 stop={final.stop_reason} cost=${cost:.5f}")
```

Check: the answer says there are 20 emails, and the last line shows the cost of both calls. The second call's input is bigger: it carries the question, the tool call and the result.

### Step 4: The loop

Before you run this, understand: an **agent** is a model, a set of tools, and a **loop**: call the model; if it asks for tools, run them all, send the results back in one user message, and call again; if it answers, print the answer and stop. The model decides what to do next. Your code decides when it has gone on long enough. That limit is a **budget**. Without one, a confused model can call tools forever, and every call costs money. `TOOLS` maps a tool name to its function. `run_tool` turns one `tool_use` block into one `tool_result` block. A tool result must be a string, so lists and dicts are converted with `json.dumps`.

Create `scripts/agent.py`:

```python
import json
import sys

from anthropic import Anthropic
from dotenv import load_dotenv

from ailab import tools as inbox
from ailab.cost import cost_usd

load_dotenv()
client = Anthropic()
MODEL = "claude-opus-5"
MAX_STEPS = 8
TOOLS = {"list_emails": inbox.list_emails}
DEFS = [inbox.LIST_EMAILS]
SYSTEM = "You are an inbox assistant for a support team. Use the tools; answer from the real emails. Be brief."


def run_tool(block) -> dict:
    try:
        result = TOOLS[block.name](**block.input)
        content = result if isinstance(result, str) else json.dumps(result)
        return {"type": "tool_result", "tool_use_id": block.id, "content": content}
    except Exception as e:  # noqa: BLE001 - any tool error must become a result, not a crash
        return {"type": "tool_result", "tool_use_id": block.id, "content": f"{type(e).__name__}: {e}", "is_error": True}


def run(question: str) -> None:
    messages = [{"role": "user", "content": question}]
    step, cost = 0, 0.0
    while True:
        step += 1
        if step > MAX_STEPS:
            print(f"Stopped: reached MAX_STEPS={MAX_STEPS} without a final answer.")
            break
        msg = client.messages.create(model=MODEL, max_tokens=2000, output_config={"effort": "low"},
                                     system=SYSTEM, tools=DEFS, messages=messages)
        u = msg.usage
        usage = {"input_tokens": u.input_tokens, "output_tokens": u.output_tokens,
                 "cost_usd": round(cost_usd(MODEL, u.input_tokens, u.output_tokens), 6)}
        cost += usage["cost_usd"]
        if msg.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": msg.content})
            results = []
            for block in msg.content:
                if block.type == "tool_use":
                    print(f"[step {step}] {block.name}({block.input})")
                    r = run_tool(block)
                    results.append(r)
            messages.append({"role": "user", "content": results})
        else:
            text = "".join(b.text for b in msg.content if b.type == "text")
            print(text)
            break
    print(f"--- steps={step} cost=${cost:.5f}")


run(" ".join(sys.argv[1:]) or "How many emails are in the inbox?")
```

Check: `uv run scripts/agent.py "How many emails are in the inbox?"` prints `[step 1] list_emails({})`, the answer, then `--- steps=2 cost=$0.0...`. Set `MAX_STEPS = 1` and run again: it stops with the budget message. Put it back to 8.

### Step 5: More tools over the inbox

Before you run this, understand: `run_tool` catches *any* exception and returns a `tool_result` with `"is_error": True` and the error message. The model reads the error like any other result and usually recovers: it tries another file name, or tells the user. A crash would throw away the whole conversation, and the money already spent on it.

Add to `tools.py` (after `LIST_EMAILS`; put `import csv` at the top):

```python
def read_email(file: str) -> str:
    return (EMAILS / file).read_text(encoding="utf-8")   # raises FileNotFoundError if the file is missing


def search_emails(query: str) -> list[str]:
    q = query.lower()
    return [f for f in list_emails() if q in read_email(f).lower()]


def high_priority() -> list[str]:
    with open("triage.csv", encoding="utf-8") as f:
        return [r["file"] for r in csv.DictReader(f) if r["priority"] == "high"]


READ_EMAIL = {
    "name": "read_email",
    "description": "Return the full text of one email. Call this when you need what an email actually says. Use a file name from list_emails or search_emails.",
    "input_schema": {"type": "object", "properties": {"file": {"type": "string", "description": "File name, for example 04.txt"}}, "required": ["file"]},
}
SEARCH_EMAILS = {
    "name": "search_emails",
    "description": "Find emails whose text contains a word or phrase, ignoring case. Call this when the user asks about a topic, such as refunds or login problems.",
    "input_schema": {"type": "object", "properties": {"query": {"type": "string", "description": "Word or phrase to look for"}}, "required": ["query"]},
}
HIGH_PRIORITY = {
    "name": "high_priority",
    "description": "List the emails already triaged as high priority in triage.csv. Call this when the user asks what is urgent.",
    "input_schema": {"type": "object", "properties": {}, "required": []},
}
```

In `agent.py`, add `"read_email": inbox.read_email, "search_emails": inbox.search_emails, "high_priority": inbox.high_priority` to `TOOLS`, and `inbox.READ_EMAIL, inbox.SEARCH_EMAILS, inbox.HIGH_PRIORITY` to `DEFS`.

Check: `uv run scripts/agent.py "Which emails mention a refund, and what does the first one say?"` shows `search_emails`, then `read_email`, then the answer. Then `uv run scripts/agent.py "Read 99.txt"`: the error goes back as a result, and the model recovers (it lists the inbox, or says the file does not exist). Then ask `"What is urgent?"` and watch `high_priority` run.

### Step 6: A tool that calls the model

Before you run this, understand: a tool can do anything your Python can do, including calling another model. Its tokens are real money too, so the tool records its cost in the list `TOOL_COSTS`, and `agent.py` adds that list to the total.

Add to `tools.py` (imports at the top: `from anthropic import Anthropic`, `from ailab.cost import cost_usd`, `from ailab.prompts import TRIAGE_SYSTEM`, `from ailab.schemas import Triage`; and `TOOL_COSTS: list[float] = []` under `EMAILS`):

```python
def triage_email(file: str) -> dict:
    resp = Anthropic().messages.parse(
        model="claude-haiku-4-5", max_tokens=1000, system=TRIAGE_SYSTEM,
        messages=[{"role": "user", "content": f"<email>\n{read_email(file)}\n</email>"}],
        output_format=Triage,
    )
    u = resp.usage
    TOOL_COSTS.append(cost_usd("claude-haiku-4-5", u.input_tokens, u.output_tokens))
    return resp.parsed_output.model_dump()


TRIAGE_EMAIL = {
    "name": "triage_email",
    "description": "Triage one email: category, priority, customer name, one-line summary, needs_human. Call this when the user asks to triage, classify or prioritise an email.",
    "input_schema": READ_EMAIL["input_schema"],   # the same single input: file
}
```

In `agent.py`, add `"triage_email": inbox.triage_email` to `TOOLS`, `inbox.TRIAGE_EMAIL` to `DEFS`, and the line `cost += sum(inbox.TOOL_COSTS)` just before the final `print` in `run`.

Check: `uv run scripts/agent.py "Triage the email about the login problem"` shows `search_emails`, then `triage_email`, and the answer has the five `Triage` fields. The Haiku call is a dict in the tool result, so `run_tool` sent it as JSON.

### Step 7: The trace

Before you run this, understand: a **trace** is a record of every step of a run: which tool, with what input, what came back, what it cost. You cannot step through an agent with a debugger, because the decisions happen inside the model. The trace is how you debug it. Keep it short: the first 80 characters of a result are enough to see what happened.

Add this helper to `agent.py`, above `run_tool`:

```python
def log(**fields) -> None:
    with open("trace.jsonl", "a", encoding="utf-8") as f:
        f.write(json.dumps(fields, ensure_ascii=False) + "\n")
```

Then add `log(step=step, tool=block.name, input=block.input, result=r["content"][:80], **usage)` after `results.append(r)`, and `log(step=step, tool=None, input=None, result=text[:80], **usage)` after `print(text)`.

Check: `rm -f trace.jsonl`, run the refund question again, then `cat trace.jsonl`: one line per tool call plus one for the final answer, and the `cost_usd` values add up to the cost line.

### Step 8: A tool with a side effect, and a human in the loop

Before you run this, understand: reading and searching change nothing; you can run them a hundred times. Writing a file is a **side effect**: the world is different afterwards. Anything that writes, sends or deletes needs a person to approve it, because the model can be wrong, and a wrong read costs nothing while a wrong send cannot be undone. Putting that approval inside the loop is called **human in the loop**. A "no" is not an error: it is a normal result the model should hear about, so it can tell the user.

Add to `tools.py`:

```python
def save_draft(file: str, reply: str) -> str:
    path = Path("drafts") / file
    path.parent.mkdir(exist_ok=True)
    path.write_text(reply, encoding="utf-8")
    return f"saved {path} ({len(reply)} chars)"


SAVE_DRAFT = {
    "name": "save_draft",
    "description": "Save a draft reply for one email into the drafts folder. Call this only when the user asks to draft or save a reply. The user is asked to approve first.",
    "input_schema": {"type": "object", "properties": {"file": {"type": "string", "description": "Email file name, for example 06.txt"},
                                                      "reply": {"type": "string", "description": "The full reply text"}}, "required": ["file", "reply"]},
}
```

In `agent.py`, add `"save_draft": inbox.save_draft` to `TOOLS`, `inbox.SAVE_DRAFT` to `DEFS`, and `NEEDS_APPROVAL = {"save_draft"}` under `DEFS`. Then add this at the top of `run_tool`, before `try`:

```python
    if block.name in NEEDS_APPROVAL:
        print("\n".join(f"{k}: {v}" for k, v in block.input.items()))
        if input("Save this draft? [y/N] ").strip().lower() != "y":
            return {"type": "tool_result", "tool_use_id": block.id, "content": "The user declined to save the draft."}
```

Check: `uv run scripts/agent.py "Draft a short, polite reply to the angry billing customer and save it"`. The assistant finds the email, writes a reply, and the script shows the file and the reply and asks. Type `n`: the model reports that nothing was saved. Run again and type `y`: `drafts/06.txt` exists.

### Step 9: Your capstone proposal

From next week you build your own agent, for the rest of the internship. This week you write the plan. Create a new repository, add `docs/proposal.md` with the headings below, one short paragraph or list each, and open it as a pull request. The supervisor reviews it there. It is due at the end of this week. The example lines use a "support ticket assistant".

- **Problem and user**: who it is for, and where judgment is needed, not just rules. *Support agents at a small company; deciding whether a ticket needs a human is a judgment call.*
- **The tools**: a table with name, inputs, output, side effect yes/no, approval yes/no. *`search_kb(query) -> list[str]`, no side effect, no approval; `send_reply(ticket, text) -> str`, side effect, approval.*
- **Data the agent reads**: *150 past tickets with the reply that was sent, exported as CSV.*
- **How a run starts**: a person, a schedule, or an event. *A new ticket arrives.*
- **What must never happen**: *A reply is sent without a human reading it.*
- **How you will measure it**: dataset, size, metrics. *50 held-out tickets; category accuracy; share of drafts the agents accept without edits.*
- **Cost per run estimate**: *4 Opus calls of about 3,000 input and 300 output tokens: about $0.09 per ticket.*
- **Weekly plan for weeks 5 to 12**: one line per week. *Week 5: data and labels. Week 6: tools and loop. Week 7: ...*
- **Three risks**: *The model invents a policy that does not exist; the labels are inconsistent; the ticket export arrives late.*

Check: the pull request is open, every heading has content, and the supervisor is the reviewer.

## Read and watch

- [Tool use overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview): the official page for Steps 1 to 4; read the tool definition, the `tool_use` and `tool_result` shapes, and parallel tool use.
- [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents): when an agent is worth it, and when a fixed sequence of calls is better; you will need this for your proposal.
- [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents): how to write the descriptions in `tools.py` so the model picks the right tool.
- Video: [How We Build Effective Agents](https://www.youtube.com/watch?v=D7_ipDqhtwk) (Barry Zhang, Anthropic, 15 min): the talk version of the article, with three rules for people who build agents.

## Turn in

- The `ai-lab` project with `src/ailab/tools.py`, `scripts/tool_once.py` and `scripts/agent.py`. No key anywhere in Git.
- `trace.jsonl` (three or more runs) and `drafts/` with one saved reply. README: the total cost of the week, and two sentences on one wrong tool call you found in the trace.
- The capstone repository with `docs/proposal.md` as an open pull request.
- Friday demo (10 minutes): `tool_once.py`; `agent.py` on the refund question, on `Read 99.txt`, and on the draft with `n` then `y`; the cost line; then two minutes on your proposal.

## Self-check

1. Does the model run `list_emails`? What exactly does it send you, and what do you send back?
2. The model asks for three tools in one reply. How many user messages do you send back, and how many `tool_result` blocks are in them?
3. `read_email` is called with a file that does not exist. What should the loop do, and why not let the script crash?
4. Which of the six tools has a side effect? Why does `search_emails` not need approval, and why does `triage_email` still cost money?
5. A run makes 4 Opus calls of 3,000 input and 200 output tokens each, plus one Haiku triage of 800 input and 100 output tokens. What does it cost?

### Answers

1. No. It sends a `tool_use` block with `name`, `input` and `id`; you run the function and send back a `tool_result` with that `id` and the output as a string.
2. One user message, with three `tool_result` blocks, one per `tool_use` id.
3. Catch the exception and return a `tool_result` with `is_error: True` and the message; the model can then try another file or tell the user, instead of losing the conversation.
4. `save_draft` writes a file. Searching changes nothing, so a wrong search is harmless. `triage_email` changes nothing on disk, but it calls Haiku, and tokens cost money.
5. Opus: 4 × (3,000 × $5 + 200 × $25) / 1,000,000 = $0.08. Haiku: (800 × $1 + 100 × $5) / 1,000,000 = $0.0013. Total about $0.081.

## If you finish early

- Ask `"How many emails are there, and which are urgent?"` and look for two `tool_use` blocks in one reply, answered by two results in one user message.
- Add a `mark_done(file)` tool that appends a line to `done.csv`. It writes, so put it in `NEEDS_APPROVAL`.
- Try the SDK's beta tool runner (`client.beta.messages.tool_runner` with `@beta_tool`): it writes the loop for you. Compare it with your `agent.py`.
