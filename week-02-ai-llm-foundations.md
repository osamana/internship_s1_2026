# Week 2: Talking to language models

---

This week you write your first real Python project. It is small. It calls three kinds of language models: Claude (from Anthropic), an OpenAI model, and a small open model that runs on your own laptop with Ollama. Each script prints the answer, how many tokens it used, and how much it cost.

At the end of the week you build `bench.py`. It sends the same 10 prompts to all three models and saves tokens, cost, and speed in a CSV file. You then make one small chart from it.

Why this matters: every decision in your capstone project (which model, how long a prompt, whether to stream, whether a local model is good enough) depends on the ideas in this week. This week you also spend real money (a few dollars). By Friday you can say exactly where it went.

Model names and prices in this document are **as of September 2026**. Re-check the pricing pages before you quote a number to anyone.

## What you will have by Friday

- [ ] A uv project named `ai-lab` with a `.env` file that is git-ignored and a `.env.example` with no values.
- [ ] `count_tokens.py`: counts the tokens of a text with the Claude token counter.
- [ ] `sdk_claude.py` and `raw_claude.py`: the same call to Claude, once with the SDK and once with raw HTTP.
- [ ] `temperature.py` and `stream_claude.py`: you saw what temperature does and what streaming looks like.
- [ ] `sdk_openai.py` and `sdk_ollama.py`: one call to an OpenAI model and one to a local model.
- [ ] `cost.py`: a helper that turns token counts into dollars.
- [ ] `bench.py` plus `bench.csv` (30 rows) and one chart or table made from it.
- [ ] Three written rules for yourself about hallucination.

## Before you start

1. Confirm week-1 tools work: `uv --version`, `python3 --version` (3.12), `git --version`, `docker --version`, `curl --version`.
2. Ask the supervisor for two API keys: one for Anthropic and one for OpenAI. An API key is a secret password that lets a program use a paid service. Never share it. Never commit it.
3. The keys have a small spending cap. If you see an error with `enforced_spend_limit_reached`, stop and tell the supervisor. Retrying does not help.
4. Create a folder for this week inside your internship folder: `mkdir ai-lab`.
5. Every script you write this week must print the tokens it used and the estimated cost. This is a rule for the whole internship.

## The flow

### Step 1: Create the project and load settings

Goal: a uv project with the packages installed and the keys loaded safely.

Before you run this, understand: your keys must never be inside code. They go in a file called `.env`. Git must ignore this file. Your code reads the file at start-up with the `pydantic-settings` library, which checks that every key is present and has the right type.

```bash
cd ai-lab
uv init --package --name ailab --python 3.12
uv add httpx anthropic openai ollama tiktoken pydantic pydantic-settings
uv add --dev ruff pyright pytest ipykernel pandas matplotlib
mkdir scripts
echo ".env" >> .gitignore
```

Create `.env.example` (this one is committed, it has no values):

```text
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
OLLAMA_BASE_URL=http://localhost:11434
```

Copy it to `.env` and paste the real keys inside: `cp .env.example .env`.

Create `src/ailab/settings.py`:

```python
from functools import lru_cache
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")

    anthropic_api_key: SecretStr
    openai_api_key: SecretStr
    ollama_base_url: str = "http://localhost:11434"

    claude_model: str = "claude-opus-5"
    openai_model: str = "gpt-5.6-luna"
    ollama_model: str = "qwen3.5:4b"

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`SecretStr` prints as `**********` in logs. You call `.get_secret_value()` only at the moment you need the real key.

Check: `uv run python -c "from ailab.settings import get_settings; print(get_settings())"` prints the settings with the keys hidden as stars. `git status` does not list `.env`.

### Step 2: What an LLM is, and count some tokens

Goal: understand three words (token, context window, chat format) and count the tokens of a sentence.

Before you run this, understand:

- A **large language model (LLM)** is a program that reads text and predicts the next piece of text, one piece at a time. It repeats this until it decides to stop.
- The pieces are called **tokens**. A token is a short chunk of text, often a word or part of a word. For English, one token is about 4 letters. Code, numbers, JSON, and Arabic split into more tokens than English prose.
- You pay per token. Input tokens (what you send) and output tokens (what the model writes) have different prices. Output tokens usually cost about 5 times more.
- Each provider has its own **tokenizer** (the tool that splits text into tokens). The same text gives different token counts on Claude and on OpenAI models. So count with the tokenizer of the model you will use.
- The **context window** is the maximum number of tokens the model can hold at once: your input plus its output. Current Claude models hold 1M tokens. Treat this as a budget, not a target. Long prompts cost more and the model uses the middle of a long prompt less well.
- Conversations use a **chat format** with roles. A `system` message tells the model how to behave. A `user` message is what the person says. An `assistant` message is what the model says. On each new call you send the whole conversation again, so a long chat costs more each turn.

Create `scripts/count_tokens.py`:

```python
import tiktoken
from anthropic import Anthropic
from ailab.settings import get_settings

TEXT = "Large language models process text as tokens. Tokens cost money. Count them before you send them."

s = get_settings()
claude = Anthropic(api_key=s.anthropic_api_key.get_secret_value())
n_claude = claude.messages.count_tokens(
    model=s.claude_model,
    messages=[{"role": "user", "content": TEXT}],
).input_tokens

enc = tiktoken.get_encoding("o200k_base")   # the tokenizer of OpenAI's GPT models
n_openai = len(enc.encode(TEXT))

print(f"chars={len(TEXT)} claude={n_claude} openai(o200k)={n_openai}")
```

`count_tokens` asks the Claude server to count. It is exact and free. Do not use `tiktoken` for Claude; it gives the wrong number.

Check: the script prints three numbers. The two token counts are different. Now replace `TEXT` with a JSON string or an Arabic sentence and see the counts go up.

### Step 3: First call to Claude with the SDK

Goal: send one question to Claude and print the answer, the token usage, and why the model stopped.

Before you run this, understand: an **SDK** (software development kit) is a library that wraps the API for you. `max_tokens` is the maximum number of output tokens the model may write. It is a hard ceiling. If the answer hits it, the text is cut in the middle and `stop_reason` is `"max_tokens"`. The reply `content` is a list of blocks. Some blocks are the model's private thinking, some are text. You print only the text blocks. `output_config={"effort": "low"}` tells the model to think less before answering. Thinking is billed as output tokens, so low effort is cheaper. Use it in this lab.

Create `scripts/sdk_claude.py`:

```python
import sys
from anthropic import Anthropic
from ailab.settings import get_settings

s = get_settings()
client = Anthropic(api_key=s.anthropic_api_key.get_secret_value())
prompt = " ".join(sys.argv[1:]) or "Explain what a token is in three sentences."

msg = client.messages.create(
    model=s.claude_model,
    max_tokens=300,
    output_config={"effort": "low"},
    system="You are a patient teacher. Use simple English.",
    messages=[{"role": "user", "content": prompt}],
)
print("".join(b.text for b in msg.content if b.type == "text"))
u = msg.usage
print(f"\n--- stop={msg.stop_reason} in={u.input_tokens} out={u.output_tokens}")
```

Run: `uv run scripts/sdk_claude.py`.

Check: you see an answer, then a line like `--- stop=end_turn in=35 out=80`. Now run it with `max_tokens=20`. The answer is cut and `stop=max_tokens`. Put the value back to 300.

### Step 4: The same call with raw HTTP

Goal: see that the SDK is only a helper. Underneath it is a normal HTTPS request with a JSON body.

Before you run this, understand: an **API** is a web address that accepts a request and returns a response. Claude's address is `https://api.anthropic.com/v1/messages`. You send three headers (the key, the API version, and the content type) and a JSON body with the model, `max_tokens`, and the messages. You learned `curl` and HTTP in week 1; this is the same thing from Python with `httpx`.

Create `scripts/raw_claude.py`:

```python
import sys
import httpx
from ailab.settings import get_settings

s = get_settings()
prompt = " ".join(sys.argv[1:]) or "Explain what a token is in three sentences."

headers = {
    "x-api-key": s.anthropic_api_key.get_secret_value(),
    "anthropic-version": "2023-06-01",
    "content-type": "application/json",
}
body = {
    "model": s.claude_model,
    "max_tokens": 300,
    "output_config": {"effort": "low"},
    "system": "You are a patient teacher. Use simple English.",
    "messages": [{"role": "user", "content": prompt}],
}
resp = httpx.post("https://api.anthropic.com/v1/messages", headers=headers, json=body, timeout=120.0)
if resp.status_code != 200:
    raise RuntimeError(f"{resp.status_code}: {resp.text[:400]}")
data = resp.json()
print(data["content"][0]["text"])
u = data["usage"]
print(f"\n--- stop={data['stop_reason']} in={u['input_tokens']} out={u['output_tokens']}")
```

Read the full JSON once: add `print(data)` for one run, then remove it. Find the text, the usage, and the stop reason with your own eyes.

Check: the output has the same shape as Step 3. For the same prompt, `in=` is the same number in both scripts. Now change the model name to a wrong one and read the error. Errors are JSON too.

### Step 5: Temperature and max tokens

Goal: run one prompt three times at two temperatures and watch the answers change.

Before you run this, understand: the model does not pick one word. It computes a probability for every possible next token. **Temperature** controls how it picks. Temperature 0 means "always take the most likely token" (the same answer almost every time). Temperature 1 means "pick randomly by the probabilities" (more variety). Important: the newest Claude models (Opus 5, Sonnet 5) **reject** `temperature` with an HTTP 400 error. They use `output_config.effort` instead. Claude Haiku 4.5 still accepts it, so we use Haiku for this experiment. Even at temperature 0 the answer can differ a little between runs. Do not build code that needs the exact same text twice.

Create `scripts/temperature.py`:

```python
from anthropic import Anthropic
from ailab.settings import get_settings

s = get_settings()
client = Anthropic(api_key=s.anthropic_api_key.get_secret_value())
PROMPT = "Give me a name for a coffee shop next to a university. One name only."

for temp in (0.0, 1.0):
    print(f"\n=== temperature={temp}")
    for _ in range(3):
        msg = client.messages.create(
            model="claude-haiku-4-5", max_tokens=30, temperature=temp,
            messages=[{"role": "user", "content": PROMPT}],
        )
        answer = "".join(b.text for b in msg.content if b.type == "text")
        print(answer.strip(), f"(out={msg.usage.output_tokens})")
```

Check: at temperature 0 the three names are the same or nearly the same. At temperature 1 they differ. Then try `temperature=0.5` with `model=s.claude_model` (Opus 5) and read the 400 error. Remove it again.

### Step 6: Streaming

Goal: print the answer word by word while the model is still writing.

Before you run this, understand: without streaming you wait for the whole answer, then get it all at once. With **streaming** the server sends small pieces as soon as they are ready. Total time is the same. But the user sees the first word much sooner, and long answers do not hit HTTP timeouts. Chat apps always stream. The time until the first piece arrives is called **time to first token (TTFT)**.

Create `scripts/stream_claude.py`:

```python
import sys, time
from anthropic import Anthropic
from ailab.settings import get_settings

s = get_settings()
client = Anthropic(api_key=s.anthropic_api_key.get_secret_value())
prompt = " ".join(sys.argv[1:]) or "List 15 fruits, one per line."

t0 = time.perf_counter()
first = None
with client.messages.stream(
    model=s.claude_model, max_tokens=300, output_config={"effort": "low"},
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

### Step 7: First call to OpenAI

Goal: call a second provider and notice what is the same and what is different.

Before you run this, understand: OpenAI's API is called the Responses API. The ideas are the same: a model name, an input, a token limit, a usage object. The names are different: `input` instead of `messages`, `max_output_tokens` instead of `max_tokens`, `reasoning={"effort": "low"}` instead of `output_config`. We use `gpt-5.6-luna`, the cheap tier, for this lab.

Create `scripts/sdk_openai.py`:

```python
import sys
from openai import OpenAI
from ailab.settings import get_settings

s = get_settings()
client = OpenAI(api_key=s.openai_api_key.get_secret_value())
prompt = " ".join(sys.argv[1:]) or "Explain what a token is in three sentences."

resp = client.responses.create(
    model=s.openai_model, input=prompt,
    reasoning={"effort": "low"}, max_output_tokens=300,
)
print(resp.output_text)
u = resp.usage
print(f"\n--- status={resp.status} in={u.input_tokens} out={u.output_tokens}")
```

Check: you see an answer and a usage line. Run Step 3 and Step 7 with the same prompt. The input token counts are different. That is the tokenizer difference from Step 2.

### Step 8: A local model with Ollama

Goal: run a small open model on your own laptop and call it from Python.

Before you run this, understand: an **open-weight model** is a model whose files you can download and run yourself. **Ollama** is a program that runs such models and gives them a local API on port 11434. Nothing leaves your machine, and there is no per-token price. The cost is your hardware and your time. A 4B model (4 billion parameters) fits on a laptop but is much weaker than Claude or GPT.

Install Ollama from its website, then:

```bash
ollama pull qwen3.5:4b        # ~3.4 GB download
ollama run qwen3.5:4b "Reply with one word: ready"
curl -s http://localhost:11434/api/tags | python3 -m json.tool | head
```

If your laptop has only 8 GB of RAM, use `qwen3.5:2b` instead and change `ollama_model` in `settings.py`. Write the model you chose in your README; week 11 reuses it.

Create `scripts/sdk_ollama.py`:

```python
import sys, time
from ollama import Client
from ailab.settings import get_settings

s = get_settings()
client = Client(host=s.ollama_base_url)
prompt = " ".join(sys.argv[1:]) or "Explain what a token is in three sentences."

t0 = time.perf_counter()
r = client.chat(model=s.ollama_model,
                messages=[{"role": "user", "content": prompt}],
                options={"temperature": 0, "seed": 42, "num_predict": 300})
total_ms = (time.perf_counter() - t0) * 1000
print(r.message.content)
print(f"\n--- in={r.prompt_eval_count} out={r.eval_count} total={total_ms:.0f}ms "
      f"tok_per_s={r.eval_count / (r.eval_duration / 1e9):.1f} cost=$0")
```

`seed=42` with `temperature=0` gives the same answer every run. This is useful for tests.

Check: you see an answer and a line with `tok_per_s=`. Run it twice; the answer is identical. Compare the answer quality with Step 3. Be honest about the difference.

### Step 9: Turn tokens into dollars

Goal: one helper that every script imports to print cost.

Before you run this, understand: cost of one call = `input_tokens × input_price + output_tokens × output_price`. Prices are per million tokens (MTok). Cheaper models exist because most tasks are easy: classifying a review or extracting a name does not need the strongest model. A big part of your job is choosing the cheapest model that is good enough, and proving it with a benchmark (Step 10) and evals (week 10).

Create `src/ailab/cost.py`:

```python
# Prices in USD per million tokens, as of Sept 2026. Re-check the pricing pages before quoting.
PRICES_SEPT_2026: dict[str, tuple[float, float]] = {   # model: (input, output)
    "claude-opus-5": (5.0, 25.0),
    "claude-sonnet-5": (2.0, 10.0),
    "claude-haiku-4-5": (1.0, 5.0),
    "gpt-5.6-luna": (0.20, 1.20),
}

def cost_usd(model: str, input_tokens: int, output_tokens: int) -> float:
    """USD for one call. Local Ollama models are not in the table -> 0.0."""
    p = PRICES_SEPT_2026.get(model)
    if p is None:
        return 0.0
    return (input_tokens * p[0] + output_tokens * p[1]) / 1_000_000

if __name__ == "__main__":
    # Worked example: 10,000 support tickets, 3,000 input + 400 output tokens each
    for m in PRICES_SEPT_2026:
        print(f"{m:18s} ${cost_usd(m, 3_000, 400) * 10_000:8.2f}")
```

Now go back to Steps 3, 4, 6, and 7 and add a cost to the last print line, for example:

```python
from ailab.cost import cost_usd
# ... in the print:  f"cost=${cost_usd(s.claude_model, u.input_tokens, u.output_tokens):.5f}"
```

Check: `uv run src/ailab/cost.py` prints about $250 for Opus 5, $100 for Sonnet 5, $50 for Haiku 4.5, and $10.80 for GPT-5.6 Luna. Every script from Steps 3 to 7 now ends with `cost=$0.00...`.

### Step 10: Build bench.py

Goal: send 10 prompts to all three models, save one CSV row per call, then make a table or chart.

Before you run this, understand: a **benchmark** is a fixed set of inputs you run on every model in the same way. It turns "I think Claude is better" into numbers. Latency here is total time in milliseconds. Each call is inside `try/except` so one failure does not stop the run; it becomes a row with `ok=False`. Expected cost of one run: well under $1.

Create `scripts/bench.py`:

```python
"""Run 10 prompts across Claude, OpenAI and Ollama; write bench.csv with tokens, cost and latency."""
import csv, sys, time
from anthropic import Anthropic
from openai import OpenAI
from ollama import Client as OllamaClient
from ailab.settings import get_settings
from ailab.cost import cost_usd

PROMPTS = {
    "short_fact": "What is the capital of Jordan? Answer in one word.",
    "classify": "Classify the sentiment of this review as positive, negative or neutral. Reply with the label only.\n\nReview: The battery died after two days but support replaced it within a week.",
    "extract_json": "Extract name, email and company as JSON with exactly those keys from: 'Hi, Lina Haddad here from Najah Robotics, reach me at lina@najahrobotics.example'.",
    "summarize": "Summarize in two sentences: " + ("Large language models process text as tokens. " * 40),
    "reason_math": "A tank fills at 3 L/min and drains at 1.2 L/min. Starting empty, how many minutes until it holds 45 L? Show one line of working then the answer.",
    "code": "Write a Python function is_palindrome(s: str) -> bool that ignores case and non-alphanumerics. Code only.",
    "rewrite": "Rewrite in plain English for a non-technical manager: 'The p95 latency regression is attributable to KV-cache eviction under concurrent prefill.'",
    "long_output": "List 25 distinct dog breeds, one per line, no numbering.",
    "multilingual": "Translate to Arabic: 'The meeting is moved to Thursday at ten.'",
    "refuse_or_hedge": "What was the closing price of Bitcoin yesterday? If you cannot know, say so in one sentence.",
}

s = get_settings()
claude = Anthropic(api_key=s.anthropic_api_key.get_secret_value())
openai_client = OpenAI(api_key=s.openai_api_key.get_secret_value())
ollama = OllamaClient(host=s.ollama_base_url)

def call_claude(prompt: str) -> tuple[str, int, int]:
    m = claude.messages.create(model=s.claude_model, max_tokens=400, output_config={"effort": "low"},
                               messages=[{"role": "user", "content": prompt}])
    text = "".join(b.text for b in m.content if b.type == "text")
    return text, m.usage.input_tokens, m.usage.output_tokens

def call_openai(prompt: str) -> tuple[str, int, int]:
    r = openai_client.responses.create(model=s.openai_model, input=prompt,
                                       reasoning={"effort": "low"}, max_output_tokens=400)
    return r.output_text, r.usage.input_tokens, r.usage.output_tokens

def call_ollama(prompt: str) -> tuple[str, int, int]:
    r = ollama.chat(model=s.ollama_model, messages=[{"role": "user", "content": prompt}],
                    options={"temperature": 0, "seed": 42, "num_predict": 400})
    return r.message.content, r.prompt_eval_count or 0, r.eval_count or 0

PROVIDERS = {"anthropic": (call_claude, s.claude_model),
             "openai": (call_openai, s.openai_model),
             "ollama": (call_ollama, s.ollama_model)}

rows = []
for name, prompt in PROMPTS.items():
    for provider, (fn, model) in PROVIDERS.items():
        t0 = time.perf_counter()
        try:
            text, n_in, n_out = fn(prompt)
            total_ms = (time.perf_counter() - t0) * 1000
            cost = cost_usd(model, n_in, n_out)
            rows.append({"prompt": name, "provider": provider, "model": model, "input_tokens": n_in,
                         "output_tokens": n_out, "total_ms": round(total_ms, 1), "cost_usd": round(cost, 6),
                         "chars": len(text), "ok": True, "error": ""})
            print(f"{provider:9s} {name:16s} in={n_in:5d} out={n_out:4d} {total_ms:7.0f}ms ${cost:.5f}")
        except Exception as e:  # a bench must keep going and record the failure
            rows.append({"prompt": name, "provider": provider, "model": model, "ok": False, "error": str(e)[:200]})
            print(f"{provider:9s} {name:16s} FAILED: {e}")

out = sys.argv[1] if len(sys.argv) > 1 else "bench.csv"
fields = ["prompt", "provider", "model", "input_tokens", "output_tokens", "total_ms", "cost_usd", "chars", "ok", "error"]
with open(out, "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=fields, extrasaction="ignore"); w.writeheader(); w.writerows(rows)
print(f"wrote {out} ({len(rows)} rows), total cost ${sum(r.get('cost_usd', 0) for r in rows):.4f}")
```

Run it three times, into `bench_run1.csv`, `bench_run2.csv`, `bench_run3.csv`, so you can see how much the numbers move between runs.

Then create `scripts/bench_chart.py`:

```python
import glob
import pandas as pd
import matplotlib.pyplot as plt

df = pd.concat([pd.read_csv(p) for p in sorted(glob.glob("bench_run*.csv"))])
df = df[df.ok]
print(df.groupby("provider")[["input_tokens", "output_tokens", "total_ms", "cost_usd"]].median())
print(df.pivot_table(index="prompt", columns="provider", values="input_tokens", aggfunc="median"))

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
for ax, col, title in zip(axes, ["total_ms", "cost_usd"], ["Total latency (ms)", "Cost (USD)"]):
    df.groupby(["prompt", "provider"])[col].median().unstack().plot.bar(ax=ax, title=title)
    ax.set_xlabel("")
plt.tight_layout(); plt.savefig("bench_chart.png", dpi=120)
print(f"total cost of all runs: ${df.cost_usd.sum():.4f}")
```

Under the chart, write five to eight sentences in your README: which provider was fastest, whether the local model's speed is usable for your capstone, where the local model was wrong, and what the three runs cost in total.

Check: each CSV has 30 rows (10 prompts × 3 providers). `bench_chart.png` exists. The pivot table shows different input token counts for the same prompt on different providers. Open the `reason_math` and `extract_json` answers by hand and compare quality.

### Step 11: Hallucination and the limits of the model

Goal: watch the model guess, and write three rules for yourself.

Before you run this, understand: a model is trained to write text that *looks* right. It is not trained to be right. When it does not know, it often still writes a confident answer. This is called **hallucination**. Two related limits: the **knowledge cutoff** (the model only saw data up to a fixed date, so it does not know recent events) and **sycophancy** (the model tends to agree with you, because agreeable answers were rated well during training). The fix is never "add 'do not hallucinate' to the prompt". The fix is to give the model the facts (documents, tools), ask for evidence, and check outputs against something you trust.

Run these three prompts with `sdk_claude.py` and with `sdk_ollama.py`:

```bash
uv run scripts/sdk_claude.py "What was the closing price of Bitcoin yesterday?"
uv run scripts/sdk_claude.py "Give me the exact Python function in the httpx library that retries a request 5 times, with its signature."
uv run scripts/sdk_claude.py "I think Python lists are faster than sets for membership tests. Confirm this."
```

Now write `docs/rules.md` in your project with three rules in your own words. Example shape:

1. If the answer contains a fact I did not give the model, I verify it before I use it.
2. For anything after the model's cutoff, I give it the data (or a tool) instead of asking it to remember.
3. I never ask the model to confirm my opinion; I ask for the strongest argument against it.

Check: at least one of the three answers is wrong, made up, or too agreeable, and you can say which one and why. The local model usually fails harder than Claude. Your three rules are in `docs/rules.md`.

## Read and watch

Readings (pick the order you like):

- [Models overview](https://platform.claude.com/docs/en/about-claude/models/overview): the table of model ids, context windows and cutoff dates you will quote all internship.
- [Pricing](https://platform.claude.com/docs/en/about-claude/pricing): the source of the numbers in `cost.py`; re-check it before you quote a price.
- [Token counting](https://platform.claude.com/docs/en/build-with-claude/token-counting): the `count_tokens` call you used in Step 2, with all its options.
- [Streaming](https://platform.claude.com/docs/en/build-with-claude/streaming): what the pieces in Step 6 really are (server-sent events) and their types.
- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/): the clearest pictures of what happens inside the model. 40 minutes, no math needed.

Videos:

- [Deep Dive into LLMs like ChatGPT](https://www.youtube.com/watch?v=7xTGNNLPyMI) (Andrej Karpathy, 3 h 31 min): watch the first hour (data and tokens) and the part on hallucinations. Watch at 1.25x. This is the best single explanation of why models behave the way they do.
- [Transformers, the tech behind LLMs](https://www.youtube.com/watch?v=wjZofJX0v4M) (3Blue1Brown, 27 min): visual and short; the last part explains temperature exactly as you saw it in Step 5.

## Turn in

Push to your internship repo (or folder):

- The `ai-lab` project: `src/ailab/settings.py`, `src/ailab/cost.py`, all scripts in `scripts/`, `.env.example`, `.gitignore` with `.env` inside. No key anywhere in Git history.
- `bench_run1.csv`, `bench_run2.csv`, `bench_run3.csv`, `bench_chart.png`.
- `README.md` with: which Ollama model you chose and your RAM, the five to eight sentences of findings, and one line with the total API cost of the week (all runs, not only the bench).
- `docs/rules.md` with your three rules.
- `uv run ruff check .` and `uv run pyright` pass on `src/` and `scripts/`.

Friday demo (15 minutes): run `sdk_claude.py` and `raw_claude.py` on the same prompt and show the token counts match. Run `stream_claude.py`. Open `bench_chart.png` and one CSV and explain three rows. Show the hallucination example that surprised you most. End with the cost line.

## Self-check

1. Why does the same text have a different token count on Claude and on GPT?
2. What is the context window, and why is "it fits" not the same as "it will be used well"?
3. What is the difference between a `system` message and a `user` message?
4. What happens when the answer reaches `max_tokens`, and how do you detect it?
5. You send `temperature=0.5` to `claude-opus-5`. What happens?
6. Does streaming make the whole answer arrive faster? What does it improve?
7. Compute the cost of 2,000 calls with 1,500 input and 300 output tokens on `claude-haiku-4-5` and on `claude-opus-5`.
8. Why is the Ollama cost `$0` in the CSV, and why is that number misleading?

### Answers

1. Each provider has its own tokenizer with its own vocabulary; always count with the tokenizer of the model you will call.
2. The maximum input plus output tokens the model can hold; long contexts cost more and the model retrieves facts from the middle of a long prompt less reliably.
3. `system` sets how the model should behave for the whole conversation; `user` is what the person asks in this turn.
4. The text is cut mid-sentence; `stop_reason` is `"max_tokens"` (Claude) or the status is `incomplete` (OpenAI); raise the limit or ask for shorter output.
5. HTTP 400 error; Opus 5 rejects `temperature`; use `output_config={"effort": ...}` instead.
6. No, total time is the same; it lowers the time to the first visible word and avoids timeouts on long answers.
7. Haiku 4.5: 2,000 × (1,500 × $1 + 300 × $5) / 1,000,000 = $6.00. Opus 5: 2,000 × (1,500 × $5 + 300 × $25) / 1,000,000 = $30.00.
8. There is no per-token price, but you pay hardware, electricity and your own time, and speed is limited by your machine; the `tok_per_s` number is what decides if it is usable.

## If you finish early

- Send an image to Claude: add a content block `{"type": "image", "source": {...}}` next to your text. See [Vision](https://platform.claude.com/docs/en/build-with-claude/vision).
- Try a bigger local model if you have 16 GB or more RAM: `ollama pull gemma4:e4b` (9.6 GB). Add it as a fourth provider in `bench.py` and compare.
- Add `claude-sonnet-5` and `claude-haiku-4-5` as extra rows in `bench.py`. Then give each `reason_math` and `extract_json` answer a quality score by hand and plot cost against quality. This is the seed of your week 10 eval.
- Write an async version of `bench.py` with `AsyncAnthropic`, `AsyncOpenAI`, `httpx.AsyncClient`, and `asyncio`. Limit concurrency with `asyncio.Semaphore` so you respect rate limits.
