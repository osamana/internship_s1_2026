# Week 2: Talking to language models

---

This week you write your first real Python project. It is small. It calls Claude (the language model from Anthropic) from Python. Each script prints the answer, how many tokens it used, and how much it cost. At the end of the week you build `bench.py`. It sends the same 5 prompts to two Claude models and saves tokens, cost, and speed in a CSV file.

Why this matters: every decision in your capstone project (which model, how long a prompt, whether to stream) depends on the ideas in this week. You also spend real money (less than a dollar). By Friday you can say exactly where it went. Model names and prices in this document are **as of September 2026**. Re-check the pricing page before you quote a number to anyone.

## What you will have by Friday

- [ ] A uv project named `ai-lab` with a `.env` file that is git-ignored and a `.env.example` with no values.
- [ ] `count_tokens.py`: counts the tokens of a text with the Claude token counter.
- [ ] `sdk_claude.py`: your first call to Claude; you saw the effect of the system prompt and of `max_tokens`.
- [ ] `stream_claude.py`: you saw what streaming looks like.
- [ ] `cost.py` and `bench.py` plus `bench.csv` (10 rows).
- [ ] Three written rules for yourself about hallucination.

## Before you start

1. Confirm week-1 tools work: `uv --version`, `python3 --version` (3.12), `git --version`.
2. Ask the supervisor for an Anthropic API key. An API key is a secret password that lets a program use a paid service. Never share it. Never commit it. It has a small spending cap: if you see `enforced_spend_limit_reached`, stop and tell the supervisor.
3. Every script you write this week must print the tokens it used and the estimated cost. This is a rule for the whole internship.

## The flow

### Step 1: Create the project and load the key

Goal: a uv project with the packages installed and the key loaded safely.

Before you run this, understand: your key must never be inside code. It goes in a file called `.env`. Git must ignore this file. Your code reads the file at start-up with the `python-dotenv` library, which copies each line of `.env` into the environment (the list of variables every program can see). The Anthropic SDK then finds the key by itself.

```bash
mkdir ai-lab && cd ai-lab
uv init --package --name ailab --python 3.12
uv add anthropic python-dotenv
mkdir scripts docs
echo ".env" >> .gitignore
```

Create `.env.example` with one line, `ANTHROPIC_API_KEY=` (this file is committed, it has no value). Copy it to `.env` and paste the real key after the `=`: `cp .env.example .env`. Every script this week starts with the same two lines: `from dotenv import load_dotenv` and `load_dotenv()`.

Check: `git status` does not list `.env`. `uv run python -c "from dotenv import load_dotenv; import os; load_dotenv(); print(os.environ['ANTHROPIC_API_KEY'][:7])"` prints the first letters of your key.

### Step 2: What an LLM is, and count some tokens

Goal: understand three words (token, context window, chat format) and count the tokens of a sentence.

Before you run this, understand:

- A **large language model (LLM)** is a program that reads text and predicts the next piece of text, one piece at a time. It repeats this until it decides to stop.
- The pieces are called **tokens**. A token is a short chunk of text, often a word or part of a word. For English, one token is about 4 letters. Code, JSON, and Arabic split into more tokens than English prose. You pay per token. Input tokens (what you send) and output tokens (what the model writes) have different prices. Output tokens cost about 5 times more.
- The **context window** is the maximum number of tokens the model can hold in one conversation: your input plus its output. Current Claude models hold 1M tokens. Treat this as a budget, not a target. Long prompts cost more.
- Conversations use a **chat format** with roles. A `system` message tells the model how to behave. A `user` message is what the person says. An `assistant` message is what the model says.

Create `scripts/count_tokens.py`:

```python
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()
client = Anthropic()
TEXT = "Large language models process text as tokens. Tokens cost money. Count them before you send them."

n = client.messages.count_tokens(
    model="claude-opus-5",
    messages=[{"role": "user", "content": TEXT}],
).input_tokens
print(f"chars={len(TEXT)} tokens={n}")
```

Check: the script prints two numbers, and `tokens` is about `chars` divided by 4. `count_tokens` asks the Claude server to count; it is exact and free. Now replace `TEXT` with a JSON string or an Arabic sentence and see the token count go up.

### Step 3: First call to Claude with the SDK

Goal: send one question to Claude and print the answer, the token usage, and why the model stopped.

Before you run this, understand: an **SDK** (software development kit) is a library that wraps the API for you. `max_tokens` is the maximum number of output tokens the model may write. It is a hard ceiling. If the answer hits it, the text is cut in the middle and `stop_reason` is `"max_tokens"`. The reply `content` is a list of blocks. Some blocks are the model's private thinking, some are text. You print only the text blocks. `output_config={"effort": "low"}` tells the model to think less before answering. Thinking is billed as output tokens, so low effort is cheaper. Use it in this lab.

Create `scripts/sdk_claude.py`:

```python
import sys
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()
client = Anthropic()
MODEL = "claude-opus-5"
prompt = " ".join(sys.argv[1:]) or "Explain what a token is in three sentences."

msg = client.messages.create(
    model=MODEL, max_tokens=300, output_config={"effort": "low"},
    system="You are a patient teacher. Use simple English.",
    messages=[{"role": "user", "content": prompt}],
)
print("".join(b.text for b in msg.content if b.type == "text"))
u = msg.usage
print(f"\n--- stop={msg.stop_reason} in={u.input_tokens} out={u.output_tokens}")
```

Check: `uv run scripts/sdk_claude.py` prints an answer, then a line like `--- stop=end_turn in=35 out=80`.

### Step 4: The system prompt

Goal: ask the same question with two different system prompts and watch the answer change.

Before you run this, understand: the **system prompt** is the `system` text you send with every call. It sets the role, the tone, and the rules for the whole conversation. The user message is only the question.

Open `sdk_claude.py`. Run it with the question `"Is coffee good for me?"`. Then change the `system` line to `"You are a strict doctor. Answer in two sentences."` and run the same question again. Then change it to `"You are a friendly coffee seller. Answer in two sentences."` and run it a third time.

Check: the three answers are clearly different in tone and content. Put the system line back to the patient teacher. Now set `max_tokens=20` and run it. The answer is cut and the last line shows `stop=max_tokens`. Put the value back to 300.

### Step 5: Streaming

Goal: print the answer word by word while the model is still writing.

Before you run this, understand: without streaming you wait for the whole answer, then get it all at once. With **streaming** the server sends small pieces as soon as they are ready. Total time is the same. But the user sees the first word much sooner. Chat apps always stream. The time until the first piece arrives is called **time to first token (TTFT)**.

Create `scripts/stream_claude.py`:

```python
import sys, time
from dotenv import load_dotenv
from anthropic import Anthropic

load_dotenv()
client = Anthropic()
prompt = " ".join(sys.argv[1:]) or "List 15 fruits, one per line."

t0 = time.perf_counter()
first = None
with client.messages.stream(
    model="claude-opus-5", max_tokens=300, output_config={"effort": "low"},
    messages=[{"role": "user", "content": prompt}],
) as stream:
    for text in stream.text_stream:
        if first is None:
            first = time.perf_counter()
        sys.stdout.write(text); sys.stdout.flush()
    final = stream.get_final_message()

u = final.usage
print(f"\n--- stop={final.stop_reason} in={u.input_tokens} out={u.output_tokens} "
      f"ttft={(first - t0) * 1000:.0f}ms total={(time.perf_counter() - t0) * 1000:.0f}ms")
```

Check: the fruits appear one by one, not all at once. The last line shows `ttft=` smaller than `total=`.

### Step 6: Turn tokens into dollars

Goal: one helper that every script imports to print cost.

Before you run this, understand: cost of one call = `input_tokens × input_price + output_tokens × output_price`. Prices are per million tokens (MTok). Cheaper models exist because most tasks are easy. A big part of your job is choosing the cheapest model that is good enough, and proving it with a benchmark (Step 7).

Create `src/ailab/cost.py`:

```python
# Prices in USD per million tokens, as of Sept 2026. Re-check the pricing page before quoting.
PRICES_SEPT_2026: dict[str, tuple[float, float]] = {   # model: (input, output)
    "claude-opus-5": (5.0, 25.0),
    "claude-haiku-4-5": (1.0, 5.0),
}

def cost_usd(model: str, input_tokens: int, output_tokens: int) -> float:
    """USD for one call."""
    p = PRICES_SEPT_2026[model]
    return (input_tokens * p[0] + output_tokens * p[1]) / 1_000_000
```

Now go back to `sdk_claude.py`. Add `from ailab.cost import cost_usd` at the top, and add `f" cost=${cost_usd(MODEL, u.input_tokens, u.output_tokens):.5f}"` to the last print line. Check: `uv run scripts/sdk_claude.py` now ends with `cost=$0.00...`. Do the same in `stream_claude.py`.

### Step 7: Build bench.py

Goal: send 5 prompts to two models, save one CSV row per call, then print a small table.

Before you run this, understand: a **benchmark** is a fixed set of inputs you run on every model in the same way. It turns "I think Opus is better" into numbers. Expected cost of one run: a few cents.

Create `scripts/bench.py`:

```python
"""Send 5 prompts to two Claude models; write bench.csv with tokens, cost and time."""
import csv, time
from dotenv import load_dotenv
from anthropic import Anthropic
from ailab.cost import cost_usd

load_dotenv()
client = Anthropic()
MODELS = ["claude-opus-5", "claude-haiku-4-5"]
PROMPTS = [
    "What is the capital of Jordan? Answer in one word.",
    "Classify the sentiment of this review as positive, negative or neutral. Reply with the label only.\n\nReview: The battery died after two days but support replaced it within a week.",
    "Extract name, email and company as JSON with exactly those keys from: 'Hi, Lina Haddad here from Najah Robotics, reach me at lina@najahrobotics.example'.",
    "A tank fills at 3 L/min and drains at 1.2 L/min. Starting empty, how many minutes until it holds 45 L? Show one line of working then the answer.",
    "What was the closing price of Bitcoin yesterday? If you cannot know, say so in one sentence.",
]

rows = []
for model in MODELS:
    for i, prompt in enumerate(PROMPTS, start=1):
        t0 = time.perf_counter()
        msg = client.messages.create(
            model=model, max_tokens=400, output_config={"effort": "low"},
            messages=[{"role": "user", "content": prompt}],
        )
        u = msg.usage
        rows.append({"model": model, "prompt": i, "input_tokens": u.input_tokens,
                     "output_tokens": u.output_tokens,
                     "cost_usd": round(cost_usd(model, u.input_tokens, u.output_tokens), 6),
                     "seconds": round(time.perf_counter() - t0, 2)})

with open("bench.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=list(rows[0])); w.writeheader(); w.writerows(rows)

print(f"{'model':16s} {'prompt':>6s} {'in':>5s} {'out':>5s} {'cost_usd':>9s} {'seconds':>7s}")
for r in rows:
    print(f"{r['model']:16s} {r['prompt']:6d} {r['input_tokens']:5d} {r['output_tokens']:5d} "
          f"{r['cost_usd']:9.5f} {r['seconds']:7.2f}")
print(f"\nwrote bench.csv ({len(rows)} rows), total cost ${sum(r['cost_usd'] for r in rows):.4f}")
```

Check: `bench.csv` has 10 rows (5 prompts × 2 models). The Haiku rows are cheaper and usually faster. Write three to five sentences in your README: which model was faster, which was cheaper, and by how much.

### Step 8: Hallucination and the limits of the model

Goal: watch the model guess, and write three rules for yourself.

Before you run this, understand: a model is trained to write text that *looks* right. It is not trained to be right. When it does not know, it often still writes a confident answer. This is called **hallucination**. A related limit is the **knowledge cutoff**: the model only saw data up to a fixed date, so it does not know recent events. The fix is never "add 'do not hallucinate' to the prompt". The fix is to give the model the facts, ask for evidence, and check outputs against something you trust.

Run this prompt with `sdk_claude.py`:

```bash
uv run scripts/sdk_claude.py "Give me the exact Python function in the httpx library that retries a request 5 times, with its signature."
```

Now write `docs/rules.md` in your project with three rules in your own words. Example shape:

1. If the answer contains a fact I did not give the model, I verify it before I use it.
2. For anything after the model's cutoff, I give it the data instead of asking it to remember.
3. I never ask the model to confirm my opinion; I ask for the strongest argument against it.

Check: the answer names a function. Open the httpx documentation and look for it. If it is not there, the model made it up, and you can say why. Your three rules are in `docs/rules.md`.

## Read and watch

- [Models overview](https://platform.claude.com/docs/en/about-claude/models/overview): the table of model ids, context windows and cutoff dates you will quote all internship.
- [Pricing](https://platform.claude.com/docs/en/about-claude/pricing): the source of the numbers in `cost.py`; re-check it before you quote a price.
- [Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming): what the pieces in Step 5 really are (server-sent events).
- Video: [Deep Dive into LLMs like ChatGPT](https://www.youtube.com/watch?v=7xTGNNLPyMI) (Andrej Karpathy, 3 h 31 min): watch the first hour (data and tokens) and the part on hallucinations. Watch at 1.25x.

## Turn in

- The `ai-lab` project: `src/ailab/cost.py`, all scripts in `scripts/`, `.env.example`, `.gitignore` with `.env` inside. No key anywhere in Git history.
- `bench.csv` and `README.md` with your three to five sentences and one line with the total API cost of the week.
- `docs/rules.md` with your three rules.
- Friday demo (10 minutes): run `sdk_claude.py`, then `stream_claude.py`. Open `bench.csv` and explain three rows. Show the hallucination answer. End with the cost line.

## Self-check

1. What is a token, and why do you count tokens before you send a long text?
2. What is the difference between a `system` message and a `user` message?
3. What happens when the answer reaches `max_tokens`, and how do you detect it?
4. Does streaming make the whole answer arrive faster? What does it improve?
5. Compute the cost of 2,000 calls with 1,500 input and 300 output tokens on `claude-haiku-4-5` and on `claude-opus-5`.

### Answers

1. A short chunk of text, often a word or part of a word; you pay per token, and the context window is a token budget.
2. `system` sets how the model should behave for the whole conversation; `user` is what the person asks in this turn.
3. The text is cut mid-sentence and `stop_reason` is `"max_tokens"`; raise the limit or ask for shorter output.
4. No, total time is the same; it lowers the time to the first visible word.
5. Haiku 4.5: 2,000 × (1,500 × $1 + 300 × $5) / 1,000,000 = $6.00. Opus 5: 2,000 × (1,500 × $5 + 300 × $25) / 1,000,000 = $30.00.
