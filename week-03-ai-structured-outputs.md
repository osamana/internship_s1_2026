# Week 3: Getting data, not text

In week 2 Claude answered you with free text. Text is nice to read, but a program cannot use it: `priority: high` one time and `Priority - High` the next. This week you make the model return exact, typed data, and you measure how often that data is right. The project is an **inbox triage** tool. Twenty fake customer-support emails become one table (category, priority, customer name, one-line summary, needs-a-human flag). You score the table against your own answers, compare two models, then build a small chat that answers questions about the inbox. Written against `anthropic` 1.11.0; prices are the September 2026 numbers already in your `cost.py`. The reading is shorter than in week 2, the project is bigger, and every script still prints tokens and cost.

## What you will have by Friday

- [ ] `data/emails/` with 20 emails, and `data/labels.csv` with your own answer for each one.
- [ ] `src/ailab/prompts.py` (your system prompt) and `src/ailab/schemas.py` (your `Triage` class).
- [ ] `triage_one.py`: one email in, one typed `Triage` object out.
- [ ] `triage_all.py` and two CSV files: the whole inbox as a table, once per model.
- [ ] `score.py` and a 2-row accuracy table in your README, with the model you chose and why.
- [ ] `chat.py`: a chat about the inbox, with prompt caching and the saving visible on screen.

## Before you start

1. Open your week-2 `ai-lab` project. `uv run scripts/sdk_claude.py "hello"` must still work.
2. Run `uv add pydantic`. Pydantic is the library for typed data classes you will use in Step 3.
3. Budget for the week: under $2. Haiku runs cost cents, Opus runs cost tens of cents. If you see `enforced_spend_limit_reached`, stop and tell the supervisor.

## The flow

### Step 1: Write the emails and the answers

Goal: 20 short fake support emails, plus the correct answer for each one, written by you before the model sees them.

Before you run this, understand: a **label** is the correct answer for one input, written by a person. You write all labels first. If you label after the model runs, you will agree with the model without noticing, and then you measure nothing. Decide your rules once and write them in your README. Suggested: `high` = the customer cannot work or lost money, `medium` = work is slowed, `low` = everything else.

Create `data/emails/` and write 20 files, `01.txt` to `20.txt`, 2 to 4 lines each. Here are six. Write the other 14 yourself: cover all five categories, and make at least two hard (angry tone, two problems in one email, no name).

```text
01.txt  Subject: Charged twice
        My card shows two payments of $49 on March 3 for one account. Please refund one. Thanks, Dana Lee
02.txt  Subject: App closes on export
        Every time I click Export to PDF the app closes. Windows 11, version 4.2.1. I can send a video. Omar
03.txt  Subject: Dark mode?
        Love the product. Any plan for a dark mode? My eyes would thank you. Best, Priya
04.txt  Subject: Locked out
        I reset my password twice and still cannot log in. I have a client demo in one hour. Please help! Marco Ruiz
05.txt  Subject: Interview request
        Hi team, I write a blog about productivity tools and would like to interview your founder. Regards, Sam
06.txt  Subject: This is unacceptable
        Third month your invoice is wrong and nobody answers. I want my money back for this month, today. Lina Haddad
```

Now create `data/labels.csv` with the header line `file,category,priority` and one row per file with your answers, for example `01.txt,billing,high` and `02.txt,bug,medium`.

Check: `ls data/emails | wc -l` prints `20`, and `data/labels.csv` has 21 lines. You have not called the model yet.

### Step 2: A prompt with four parts

Goal: a system prompt that says exactly what to do, and a script that triages one email as free text.

Before you run this, understand: a **prompt** is everything you send to the model. A good prompt has four parts, in this order: the role (who the model is: "a support triage assistant"), the task (the one thing to do now), the rules (how to decide each field, and what to do when something is missing), and the output format (the exact shape of the answer). Role, task and rules are the same for every email, so they go in the **system prompt** (the `system` text sent with every call, from week 2). The email goes in the user message, wrapped in `<email>` tags, so the model knows exactly where the data starts and ends.

Create `src/ailab/prompts.py`:

```python
TRIAGE_SYSTEM = """You are a support triage assistant for a small software company.
Read the customer email inside <email> tags and triage it.

Rules:
- category: billing (money, invoices, refunds), bug (something is broken), feature (a request for something new), account (login, password, access), other.
- priority: high if the customer cannot work or lost money; medium if work is slowed; low otherwise.
- customer_name: the name the customer signs with, or null if there is none.
- summary: one short sentence in plain English.
- needs_human: true if the customer asks for a refund, is angry, or mentions legal action.

Output format: one line per field, in this order: category, priority, customer_name, summary, needs_human.
"""
```

Create `scripts/triage_one.py`:

```python
import sys
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv

from ailab.cost import cost_usd
from ailab.prompts import TRIAGE_SYSTEM

load_dotenv()
client = Anthropic()
MODEL = "claude-opus-5"
email = Path(sys.argv[1]).read_text(encoding="utf-8")

msg = client.messages.create(
    model=MODEL, max_tokens=1000, output_config={"effort": "low"},
    system=TRIAGE_SYSTEM,
    messages=[{"role": "user", "content": f"<email>\n{email}\n</email>"}],
)
print("".join(b.text for b in msg.content if b.type == "text"))
u = msg.usage
print(f"--- in={u.input_tokens} out={u.output_tokens} cost=${cost_usd(MODEL, u.input_tokens, u.output_tokens):.5f}")
```

Check: `uv run scripts/triage_one.py data/emails/01.txt` prints five lines in the order you asked for, then the cost line. Try `04.txt`. The answer follows your rules, but it is still text. Run `01.txt` three times and compare: spelling, order, or extra words change. A program cannot trust that.

### Step 3: Force the shape with a schema

Goal: the same call, but the answer is a Python object with typed fields, every time.

Before you run this, understand: a **schema** is a precise description of a data shape: which fields, which types, which values are allowed. **Structured output** means the model is forced to answer in that shape. You write the schema as a Pydantic class. `Literal["low", "medium", "high"]` means "exactly one of these three values". `str | None` means "a text, or nothing". The SDK sends the schema with the request, and the server guarantees the answer matches it. No JSON parsing, no cleaning, no retries on your side.

Create `src/ailab/schemas.py`:

```python
from typing import Literal

from pydantic import BaseModel


class Triage(BaseModel):
    category: Literal["billing", "bug", "feature", "account", "other"]
    priority: Literal["low", "medium", "high"]
    customer_name: str | None
    summary: str
    needs_human: bool
```

In `triage_one.py`, add `from ailab.schemas import Triage` under the other `ailab` imports, then replace the call and the first `print` with:

```python
resp = client.messages.parse(
    model=MODEL, max_tokens=1000, output_config={"effort": "low"},
    system=TRIAGE_SYSTEM,
    messages=[{"role": "user", "content": f"<email>\n{email}\n</email>"}],
    output_format=Triage,
)
t = resp.parsed_output
print(repr(t))
u = resp.usage
```

`client.messages.parse` is `create` plus a schema. `resp.parsed_output` is a real `Triage` object: `t.category` is a string you can compare with `==`, `t.needs_human` is `True` or `False`. The field names in the class now define the output, so delete the "Output format" line from `TRIAGE_SYSTEM`. Keep the rules: the schema controls the shape, the rules control the judgment.

Check: the output starts with `Triage(category='billing', priority='high'`. Run all six example emails. Never free text, never a missing field, never a value outside your lists.

### Step 4: Teach by example

Goal: fix one email the model gets wrong by showing two worked examples.

Before you run this, understand: a **few-shot example** is a small input/output pair inside the prompt that shows what a correct answer looks like. Rules explain; examples show. Examples are the strongest tool for borderline cases, where two rules seem to apply at once. They go in the system prompt, inside `<examples>` tags, so the model does not confuse them with the real email. Write them in the same shape as the real output (here, JSON with your five field names).

Run `uv run scripts/triage_one.py data/emails/06.txt` three times. Lina is angry, says the invoice is wrong, and wants a refund. You want `billing`, `high`, `needs_human=True`. Write down what you get. Often it is `other` or `medium`, and not the same every time.

Add this to `TRIAGE_SYSTEM`, after the rules and before the closing `"""`:

```text
<examples>
<example>
<email>Subject: Invoice typo. The company name on invoice 1042 is spelled wrong. No rush. - Nadia</email>
<answer>{"category": "billing", "priority": "low", "customer_name": "Nadia", "summary": "Company name misspelled on invoice 1042.", "needs_human": false}</answer>
</example>
<example>
<email>Subject: Where is my refund?! I cancelled two weeks ago and you charged me again. Fix this now or I call my bank.</email>
<answer>{"category": "billing", "priority": "high", "customer_name": null, "summary": "Charged after cancelling; demands a refund.", "needs_human": true}</answer>
</example>
</examples>
```

Check: `06.txt` now gives `billing`, `high`, `needs_human=True`, three times out of three. Run `01.txt` and `03.txt` again: the examples must not break the easy cases. Note that `in=` went up: examples cost input tokens on every call.

### Step 5: Triage the whole inbox

Goal: one CSV row per email, with tokens and cost, for the model you name on the command line.

Before you run this, understand: this is the Step 3 call inside a loop. One new idea: `sys.argv[1]` picks the model, so the same script serves both models. Haiku 4.5 does not support the `effort` setting, so the script only sends it to Opus. `model_dump()` turns the `Triage` object into a dict, so its five fields become five CSV columns.

Create `scripts/triage_all.py`:

```python
import csv
import sys
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv

from ailab.cost import cost_usd
from ailab.prompts import TRIAGE_SYSTEM
from ailab.schemas import Triage

load_dotenv()
client = Anthropic()
MODEL = sys.argv[1] if len(sys.argv) > 1 else "claude-haiku-4-5"
EXTRA = {"output_config": {"effort": "low"}} if MODEL == "claude-opus-5" else {}

rows = []
for path in sorted(Path("data/emails").glob("*.txt")):
    resp = client.messages.parse(
        model=MODEL, max_tokens=1000, system=TRIAGE_SYSTEM,
        messages=[{"role": "user", "content": f"<email>\n{path.read_text(encoding='utf-8')}\n</email>"}],
        output_format=Triage, **EXTRA,
    )
    u = resp.usage
    rows.append({"file": path.name, **resp.parsed_output.model_dump(),
                 "input_tokens": u.input_tokens, "output_tokens": u.output_tokens,
                 "cost_usd": round(cost_usd(MODEL, u.input_tokens, u.output_tokens), 6)})
    print(f"{path.name} {rows[-1]['category']:8s} {rows[-1]['priority']:6s} {rows[-1]['summary']}")

with open("triage.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=list(rows[0]))
    w.writeheader()
    w.writerows(rows)
print(f"\nwrote triage.csv ({len(rows)} rows) model={MODEL} total cost ${sum(r['cost_usd'] for r in rows):.4f}")
```

Check: `uv run scripts/triage_all.py` prints 20 lines and a total of a few cents. Open `triage.csv`: the columns are `file, category, priority, customer_name, summary, needs_human, input_tokens, output_tokens, cost_usd`.

### Step 6: Measure, then compare two models

Goal: a number for how often the model agrees with your labels, for Haiku and for Opus.

Before you run this, understand: **accuracy** is the share of rows where the model's answer equals your label. 17 right out of 20 is 0.85. "It looks right" is not a measurement: you looked at three rows and remembered the good ones. One number over all 20 rows, against labels written before the run, is a measurement. Category and priority get separate numbers because they fail for different reasons.

Create `scripts/score.py`:

```python
import csv
import sys

with open("data/labels.csv", encoding="utf-8") as f:
    labels = {r["file"]: r for r in csv.DictReader(f)}

print(f"{'csv':18s} {'category':>8s} {'priority':>8s} {'cost_usd':>8s}  wrong files")
for path in sys.argv[1:] or ["triage.csv"]:
    with open(path, encoding="utf-8") as f:
        rows = list(csv.DictReader(f))
    cat_ok = [r["category"] == labels[r["file"]]["category"] for r in rows]
    pri_ok = [r["priority"] == labels[r["file"]]["priority"] for r in rows]
    wrong = [r["file"] for r, c, p in zip(rows, cat_ok, pri_ok) if not (c and p)]
    cost = sum(float(r["cost_usd"]) for r in rows)
    print(f"{path:18s} {sum(cat_ok) / len(rows):8.2f} {sum(pri_ok) / len(rows):8.2f} {cost:8.4f}  {', '.join(wrong) or 'none'}")
```

Run both models and keep both tables:

```bash
uv run scripts/triage_all.py claude-haiku-4-5 && cp triage.csv triage_haiku.csv
uv run scripts/triage_all.py claude-opus-5 && cp triage.csv triage_opus.csv
uv run scripts/score.py triage_haiku.csv triage_opus.csv
```

Check: a 2-row table with two accuracies and two costs. Open each wrong file and decide: was the model wrong, or was your label wrong? Fix a label only when you are sure, and say so in the commit message. Then write two sentences in your README: which model you would use for this inbox and why. Use both numbers, accuracy and cost.

### Step 7: Ask the inbox questions

Goal: a chat in the terminal that answers questions about `triage.csv` and remembers the conversation.

Before you run this, understand: the model has no memory between calls. The **conversation history** is the `messages` list. Each turn you append your question as a `user` message, get the answer, and append it as an `assistant` message. The next turn sends the whole list again. So `input_tokens` grows every turn: you pay for the full history each time. The inbox table goes in the system prompt, so the model answers from your data instead of guessing.

Create `scripts/chat.py`:

```python
import sys
from pathlib import Path

from anthropic import Anthropic
from dotenv import load_dotenv

from ailab.cost import cost_usd

load_dotenv()
client = Anthropic()
MODEL = sys.argv[1] if len(sys.argv) > 1 else "claude-haiku-4-5"
EXTRA = {"output_config": {"effort": "low"}} if MODEL == "claude-opus-5" else {}
SYSTEM = ("You answer questions about a support inbox. Use only the table below. "
          "Name the files your answer is based on.\n\n" + Path("triage.csv").read_text(encoding="utf-8"))

messages = []
while True:
    q = input("\nyou> ").strip()
    if q == "quit":
        break
    messages.append({"role": "user", "content": q})
    msg = client.messages.create(model=MODEL, max_tokens=600, system=SYSTEM, messages=messages, **EXTRA)
    text = "".join(b.text for b in msg.content if b.type == "text")
    messages.append({"role": "assistant", "content": text})
    u = msg.usage
    print(f"bot> {text}\n--- turn={len(messages) // 2} in={u.input_tokens} out={u.output_tokens} "
          f"cost=${cost_usd(MODEL, u.input_tokens, u.output_tokens):.5f}")
```

Check: ask `Which emails are high priority?`, then `And which of those are about billing?`. The second answer filters the first one: "those" only works because the first question and answer are still in `messages`. `in=` is larger on turn 2 than on turn 1, and larger again on turn 3. Type `quit` to exit.

### Step 8: Stop paying for the same table every turn

Goal: the same chat on Opus, with the system prompt cached, and the saving visible in the usage numbers.

Before you run this, understand: a **cache** is a saved copy of work, kept so the work is not done again. With **prompt caching** the server keeps your processed system prompt for 5 minutes. Reading it again costs 10% of the normal input price; the first write costs 1.25x. Two rules: the cached part must be the unchanging start of the request (our system prompt is), and it must be long enough: at least 512 tokens on Opus 5, 4,096 on Haiku 4.5. That is why this step uses Opus. `cost.py` ignores cache prices on purpose this week, so its cost line now misses the cached tokens; just read the token numbers.

In `chat.py`, make `system` a list with one cached block: replace `system=SYSTEM` with `system=[{"type": "text", "text": SYSTEM, "cache_control": {"type": "ephemeral"}}]`. Then add this line inside the last `print`, between its two existing lines: `f"cache_write={u.cache_creation_input_tokens} cache_read={u.cache_read_input_tokens} "`.

Check: `uv run scripts/chat.py claude-opus-5`, same two questions. Turn 1: `cache_write` > 0 and `cache_read` = 0. Turn 2, within 5 minutes: `cache_read` > 0 and `cache_write` = 0. On both turns `in=` is small compared with Step 7, because the table is now counted under the cache fields, not under `in=`. If `cache_write` is 0 on turn 1, your system prompt is under 512 tokens: add the raw emails to `SYSTEM` after the table, with `"\n\n".join(p.read_text(encoding="utf-8") for p in sorted(Path("data/emails").glob("*.txt")))`, and run again.

## Read and watch

- [Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs): the official page for Step 3; read the Pydantic part and the list of what a schema may contain.
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices): the long reference behind Steps 2 and 4; read the sections on clarity, examples, and XML tags.
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching): the rules behind Step 8, including the minimum size per model.
- Video: [Prompting 101 | Code w/ Claude](https://www.youtube.com/watch?v=ysPbXH0LpIE) (Anthropic, 25 min): a prompt built layer by layer; match each layer to a part of `TRIAGE_SYSTEM`.

## Turn in

- The `ai-lab` project with `data/`, `src/ailab/prompts.py`, `src/ailab/schemas.py`, and the four scripts. No key anywhere in Git.
- `data/labels.csv`, `triage_haiku.csv` and `triage_opus.csv`.
- README: your priority rules, the 2-row accuracy table, two sentences on the model you chose, and one line with the week's total API cost.
- Friday demo (10 minutes): `triage_one.py` on `06.txt`, `score.py` on both CSVs, then three questions in `chat.py` on Opus showing `cache_read` > 0. End with the cost line.

## Self-check

1. Why do you write the labels before you run the model?
2. What does `Literal["low", "medium", "high"]` change about the model's answer?
3. Which parts of the prompt go in `system` and which in the user message, and why does that split help caching?
4. On turn 5 of `chat.py`, how many items are in `messages` when you send the call, and why does `input_tokens` grow?
5. A 2,000-token system prompt on `claude-opus-5`, 10 turns within 5 minutes: what does that prompt cost in input tokens without caching, and with it?

### Answers

1. Labels are your independent answer; written after, they drift toward the model's output and the accuracy number means nothing.
2. The answer can only be one of those three exact strings; the model cannot invent `urgent` or `High`.
3. Stable things (role, rules, examples, the inbox table) in `system`; the changing email or question in the user message; a cache only works on an unchanging start.
4. Nine (4 user/assistant pairs plus the new question); the whole list is re-sent every turn, so the input gets longer.
5. Without: 10 × 2,000 × $5 / 1,000,000 = $0.10. With: one write 2,000 × $6.25 / 1,000,000 = $0.0125, plus 9 reads × 2,000 × $0.50 / 1,000,000 = $0.009; total $0.0215.

## If you finish early

- Add `reply_draft: str` to `Triage` (a two-sentence draft answer) and watch `out=` and the cost per email change.
- Run `triage_all.py claude-opus-5` with `"effort": "medium"` and score again: did accuracy move, and what did it cost?
- Stream the chat answer word by word with `client.messages.stream` from week 2, keeping the `messages` list the same.
