# Week 5: Giving the assistant knowledge

In week 4 the inbox assistant could read emails, but it knew nothing about the company: ask it what the refund policy says and it guesses. This week you give it **knowledge**: eight short company documents (refunds, invoices, accounts, passwords, plans, export, privacy, support hours) and a way to find the right paragraph for any question, even when the question uses different words than the document. Then Claude answers from that paragraph and cites it. This is **retrieval-augmented generation (RAG)**: retrieve first, then generate. At the end the week-4 agent gets a `search_docs` tool, so it can answer a customer email with the real policy. Written against `anthropic` 2.4.0 (the code also runs on 1.15.0), `fastembed` 0.9.1 and `numpy` 2.5.4 on Python 3.12. The embedding model runs on your laptop, so only `ask.py` and `agent.py` cost money, and `ask.py` uses `claude-haiku-4-5` by default. The rules do not change: work in order, commit after every step, type every command yourself, read every error twice, and watch the cost line. Left out on purpose, they come later: a real vector database, PDF parsing, reranking, and frameworks.

## What you will have by Friday

- [ ] `data/docs/` with eight Markdown documents about the company.
- [ ] `src/ailab/rag.py` (chunks, keyword search, embeddings, cosine search over `index.json`) with `search_kw.py` and `search.py`: the same question, two kinds of search, and the lesson between them.
- [ ] `data/questions.csv` (10 questions with the expected source) and `eval_search.py` with hit@3 for both searches.
- [ ] `ask.py`: an answer with numbered sources and the cost line.
- [ ] `agent.py` with the `search_docs` tool, a saved draft that quotes the policy, and a README with the search table and the week's total cost.

## Before you start

Open `ai-lab`. These three commands prove week 4 still works and cost nothing:

```bash
uv run python -c "from ailab.tools import list_emails; print(len(list_emails()))"
uv run python -c "from dotenv import load_dotenv; import os; load_dotenv(); print(os.environ['ANTHROPIC_API_KEY'][:7])"
git status
```

1. The first prints `20`, the second prints the start of your key, the third says the working tree is clean.
2. Budget for the week: under $1. Embeddings are free (they run on your machine). One `ask.py` question on Haiku costs well under a cent; one agent run on Opus costs a few cents. If you see `enforced_spend_limit_reached`, stop and tell the supervisor.
3. Run every script from the project root (`ai-lab/`). Step 4 downloads a model of about 65 MB once, so do it on a good connection; after that, nothing this week needs the network except the two scripts that call Claude.

## The flow

### Step 1: Write the knowledge base

Goal: eight short Markdown documents about the same software company as your week-3 emails, written so they can answer those emails.

Before you run this, understand: a **knowledge base** is the set of documents your assistant may answer from. Each document has one `#` title and three or four `##` sections, and each section is two to four plain sentences, one sentence per line. Two rules make Steps 3 and 4 interesting: use the word *refund* only in `refunds.md` and never write *money back* anywhere; write *sign in*, never *log in*. Customers say "money back" and "log in", so the questions and the documents will use different words for the same thing.

Create `data/docs/` and write these two files exactly as shown. `data/docs/refunds.md`:

```markdown
# Refund policy

## Refund window
You can ask for a refund within 14 days of any payment, for any reason.
After 14 days we do not refund, but you can cancel so you are not billed again.
A refund goes to the card you paid with and takes 5 to 10 business days to appear.

## Duplicate charges
If your card shows two payments for the same month, the second one is a duplicate.
We refund a duplicate in full, with no 14-day limit.
Reply to your receipt email with both payment dates and we fix it the same day.

## How to request a refund
Write to support from the email address on the account and give the date of the payment.
We confirm the refund by email within one business day.
We cannot refund payments made through a reseller; contact the reseller instead.
```

`data/docs/passwords.md`:

```markdown
# Passwords

## Resetting your password
Click Forgot password on the sign-in page and enter your email address.
The reset link arrives within 5 minutes and is valid for one hour.
If no email arrives, check your spam folder and confirm you used the address on the account.

## Locked accounts
After 5 wrong passwords in a row, the account is locked for 30 minutes.
A password reset does not unlock the account; wait 30 minutes or ask support to unlock it.
During a lock, the sign-in page shows "Too many attempts".

## Password rules
A password needs at least 12 characters and cannot be one of your last 5 passwords.
We recommend a password manager and two-factor authentication.
Passwords are stored hashed; support cannot see or send you your password.
```

Write the other six yourself, with exactly these sections: `accounts.md` (Creating an account, Signing in, Deleting an account), `invoices.md` (When you are billed, Fixing a wrong invoice, Payment methods), `plans.md` (Free plan, Pro plan, Team plan, Changing plans; the Pro plan is $49 per month), `export.md` (Export to PDF, Export to CSV, Known problems: version 4.2.1 on Windows 11 closes on PDF export, fixed in 4.2.2), `privacy.md` (What we store, Who can see your data, Deleting your data), `support-hours.md` (When we answer, Response times, Urgent issues). Emails 01, 02, 04 and 06 from week 3 must be answerable from these files.

Check: `ls data/docs | wc -l` prints `8`; `grep -il refund data/docs/*` prints only `data/docs/refunds.md`; `grep -rilE "money back|log in" data/docs` prints nothing. You have not called the model yet. Commit.

### Step 2: Cut the documents into chunks

Goal: `rag.py` with `load_chunks()`: one chunk per `##` section, each with a source you can cite.

Before you run this, understand: a **chunk** is one piece of a document, small enough to be about one thing. You do not send all eight documents with every question: that costs input tokens every time, it buries the one relevant paragraph among 24 others, and a real knowledge base of thousands of pages would not fit at all. You cut once, then send only the two or three chunks that matter. The cut is at the `## ` heading, so each chunk keeps its heading as its first line and gets a **source** like `refunds.md#Refund window`: file name, `#`, heading. The source is what the answer will cite. `split_doc` also knows two other ways to cut (`file` and `sentence`); you use them in Step 8.

Create `src/ailab/rag.py`:

```python
from pathlib import Path

DOCS = Path("data/docs")


def split_doc(name: str, text: str, how: str) -> list[tuple[str, str]]:
    """Return (source, text) pairs for one document. how: heading, file or sentence."""
    if how == "file":
        return [(name, text.strip())]
    parts = []
    for section in text.split("\n## ")[1:]:
        heading, _, body = section.partition("\n")
        source = f"{name}#{heading.strip()}"
        if how == "sentence":
            lines = [s for s in body.strip().split("\n") if s]
            parts += [(f"{source}#{i}", s) for i, s in enumerate(lines, start=1)]
        else:
            parts.append((source, f"{heading.strip()}\n{body.strip()}"))
    return parts


def load_chunks(how: str = "heading") -> list[dict]:
    chunks = []
    for path in sorted(DOCS.glob("*.md")):
        text = path.read_text(encoding="utf-8")
        for source, part in split_doc(path.name, text, how):
            chunks.append({"id": len(chunks), "source": source, "text": part})
    return chunks
```

Check: `uv run python -c "from ailab.rag import load_chunks; c = load_chunks(); print(len(c)); print(c[6])"` prints `25`, then one chunk: a dict with `id`, a `source` like `invoices.md#When you are billed`, and a `text` that starts with the heading. `load_chunks("file")` has 8 chunks. Commit.

### Step 3: Search by words first

Goal: `search_kw.py`: the simplest search there is, and the reason it is not enough.

Before you run this, understand: **keyword search** scores a chunk by how many of the question's words appear in it. `words()` lowercases the text, pulls out runs of letters and digits with a regular expression, and drops words of one or two letters like *I* or *to*. The score is the share of the question's words found in the chunk: 1.0 means all of them. Both searches this week return a list of `(score, chunk)` pairs, best first. Keyword search is fast, free and easy to explain, and it has a blind spot you are about to see.

Add to `rag.py` (put `import re` at the top):

```python
def words(text: str) -> set[str]:
    """Lowercase words of 3 letters or more, without punctuation."""
    return {w for w in re.findall(r"[a-z0-9]+", text.lower()) if len(w) > 2}


def keyword_search(query: str, chunks: list[dict], k: int = 3) -> list[tuple]:
    q = words(query)
    scored = [(len(q & words(c["text"])) / max(len(q), 1), c) for c in chunks]
    return sorted(scored, key=lambda hit: hit[0], reverse=True)[:k]
```

Create `scripts/search_kw.py`:

```python
import sys

from ailab.rag import keyword_search, load_chunks

query = " ".join(sys.argv[1:]) or "money back"
for score, chunk in keyword_search(query, load_chunks()):
    print(f"{score:.2f}  {chunk['source']}")
```

Check: `uv run scripts/search_kw.py "How much does the Pro plan cost"` puts `plans.md#Pro plan` first (0.43 with my documents). Now `uv run scripts/search_kw.py "money back"` and `uv run scripts/search_kw.py "I was charged twice"`: nothing from `refunds.md` appears, and the scores are 0.00 (or a wrong chunk that happens to contain *back*). The documents say *refund* and *duplicate*; the customer says *money back* and *charged twice*. Words are not meaning. Commit.

### Step 4: Search by meaning with embeddings

Goal: embed every chunk once with a local model, then find the chunks closest in meaning to a question.

Before you run this, understand: an **embedding** is a list of numbers that describes the meaning of a text; here 384 numbers per text. Texts that mean similar things get similar numbers, so *money back* lands near *refund* although they share no word. The model that makes the numbers is small (about 65 MB) and runs on your laptop through the `fastembed` library: no key, no server, no cost. **Cosine similarity** is the closeness score between two such lists: 1.0 means the same direction (the same meaning), about 0 means unrelated. The formula is the dot product divided by the two lengths, and that is all `cosine()` does. The plan: `build_index.py` embeds every chunk once and saves chunks and vectors together in `index.json` (an **index** is a saved structure built for fast lookup); `search()` embeds the question and compares it with every vector. `@cache` makes `model()` load the model only once per run.

Run `uv add fastembed` (it brings `numpy`, the array library used in `cosine()`), then `echo "index.json" >> .gitignore`: the index is generated, not committed. Add to `rag.py` (at the top: `import json`, `from functools import cache`, `import numpy as np` and `from fastembed import TextEmbedding`; under `DOCS` the three lines `INDEX = Path("index.json")`, `EMBED_MODEL = "BAAI/bge-small-en-v1.5"` and `MODEL_DIR = Path.home() / ".cache" / "fastembed"`):

```python
@cache
def model() -> TextEmbedding:
    return TextEmbedding(EMBED_MODEL, cache_dir=str(MODEL_DIR))


def embed(texts: list[str]) -> list[list[float]]:
    return [vector.tolist() for vector in model().embed(texts)]


def cosine(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))


def search(query: str, k: int = 3) -> list[tuple]:
    index = json.loads(INDEX.read_text(encoding="utf-8"))
    q = embed([query])[0]
    scored = [(cosine(q, v), c) for v, c in zip(index["vectors"], index["chunks"])]
    return sorted(scored, key=lambda hit: hit[0], reverse=True)[:k]
```

Create `scripts/build_index.py`:

```python
import json
import sys

from ailab.rag import INDEX, embed, load_chunks

how = sys.argv[1] if len(sys.argv) > 1 else "heading"
chunks = load_chunks(how)
vectors = embed([c["text"] for c in chunks])
INDEX.write_text(json.dumps({"how": how, "chunks": chunks, "vectors": vectors}))
print(f"wrote {INDEX}: {len(chunks)} chunks ({how}), {len(vectors[0])} numbers each")
```

Create `scripts/search.py`:

```python
import sys

from ailab.rag import search

query = " ".join(sys.argv[1:]) or "money back"
for score, chunk in search(query):
    print(f"{score:.3f}  {chunk['source']}")
```

Check: `uv run scripts/build_index.py` downloads the model the first time (a "Fetching 5 files" progress bar, about a minute; a warning about unauthenticated requests to the HF Hub is harmless). The model is saved in `~/.cache/fastembed` (about 65 MB) and never downloaded again. Then it prints `wrote index.json: 25 chunks (heading), 384 numbers each`. Run it again: under a second. Now `uv run scripts/search.py "money back"`: `refunds.md#Refund window` is first with a score around 0.62, and `refunds.md#How to request a refund` is right behind it. Try `"I was charged twice"` (`refunds.md#Duplicate charges` first) and `"I cannot log in after resetting my password"` (`passwords.md#Locked accounts` first). Commit.

### Step 5: Measure both searches

Goal: ten questions with the expected source, written before you look at results, and one table for both searches.

Before you run this, understand: this is week 3's labels again. You write the questions and the expected `file#heading` first; if you look at results first, you will write questions the search already gets right. **hit@3** is the share of questions where the expected chunk is among the top 3 results. Use the customer's words, not the document's: at least four questions must use a word the documents do not. One more number matters: the characters in the top 3 chunks, because that is what you will pay to send to Claude in Step 6. A result counts as a hit when its source and the expected source start the same way, so `refunds.md` matches `refunds.md#Refund window`; that makes the script work for the other chunkings in Step 8 too. In the script, `scores[name][0] += hit` works because Python counts `True` as 1.

Create `data/questions.csv` with the header line `question,expected` and ten rows like `Can I get my money back?,refunds.md#Refund window`, `I cannot log in after resetting my password,passwords.md#Locked accounts` and `Do you work on weekends?,support-hours.md#When we answer`. Then create `scripts/eval_search.py`:

```python
import csv
import json

from ailab.rag import INDEX, keyword_search, search

index = json.loads(INDEX.read_text(encoding="utf-8"))
with open("data/questions.csv", encoding="utf-8") as f:
    questions = list(csv.DictReader(f))

scores = {"keyword": [0, 0], "embedding": [0, 0]}  # hits, characters sent
for q in questions:
    expected = q["expected"]
    kw = keyword_search(q["question"], index["chunks"])
    emb = search(q["question"])
    for name, results in (("keyword", kw), ("embedding", emb)):
        places = [c["source"] for _, c in results]
        hit = any(p.startswith(expected) or expected.startswith(p) for p in places)
        scores[name][0] += hit
        scores[name][1] += sum(len(c["text"]) for _, c in results)
        if not hit:
            print(f"miss ({name}): {q['question']} -> {places}")

n = len(questions)
print(f"\nchunking={index['how']} chunks={len(index['chunks'])} questions={n}")
for name, (hits, chars) in scores.items():
    print(f"{name:9s} hit@3={hits}/{n} chars/question={chars // n}")
```

Check: `uv run scripts/eval_search.py` prints one `miss` line for each question a search got wrong, then a two-row table. With my documents and questions: keyword `hit@3=5/10`, embedding `hit@3=10/10`, about 700 characters per question for both. Your numbers will differ, but embeddings should win clearly. Read every `miss` line and say in one sentence why the words did not match. Put the two rows in your README. Commit.

### Step 6: Answer with sources

Goal: `ask.py`: retrieve three chunks, let Claude answer only from them, and print the answer, the sources and the cost.

Before you run this, understand: this is the whole of RAG: first *retrieve* the chunks that match the question, then *generate* the answer from them. The model is not asked to remember your policy; it is handed the policy. The system prompt has three rules: use only the documents, cite them as `[1]`, `[2]`, and say "I do not know" when they do not answer. The chunks go in the user message as numbered `<doc>` blocks with their source, the same trick as the `<email>` tags in week 3. Haiku is the default: the hard part, finding the right text, is already done, so the cheap model is enough. Pass a model name as the second argument to compare.

Create `scripts/ask.py`:

```python
import sys

from anthropic import Anthropic
from dotenv import load_dotenv

from ailab.cost import cost_usd
from ailab.rag import search

load_dotenv()
client = Anthropic()
MODEL = sys.argv[2] if len(sys.argv) > 2 else "claude-haiku-4-5"
EXTRA = {"output_config": {"effort": "low"}} if MODEL == "claude-opus-5" else {}
SYSTEM = """You answer customer questions for a small software company.
Use only the documents inside <doc> tags. After each fact, cite the document number like [1].
If the documents do not answer the question, say that you do not know. Be brief."""

question = sys.argv[1] if len(sys.argv) > 1 else "Can I get my money back?"
hits = search(question)
docs = ""
for i, (_, chunk) in enumerate(hits, start=1):
    docs += f'<doc id="{i}" source="{chunk["source"]}">\n{chunk["text"]}\n</doc>\n'
messages = [{"role": "user", "content": f"{docs}\nQuestion: {question}"}]
msg = client.messages.create(
    model=MODEL, max_tokens=500, system=SYSTEM, messages=messages, **EXTRA
)
print("".join(b.text for b in msg.content if b.type == "text"))
print("\nSources:")
for i, (score, chunk) in enumerate(hits, start=1):
    print(f"[{i}] {chunk['source']} (similarity {score:.2f})")
cost = cost_usd(MODEL, msg.usage.input_tokens, msg.usage.output_tokens)
print(f"--- in={msg.usage.input_tokens} out={msg.usage.output_tokens} cost=${cost:.5f}")
```

Check: `uv run scripts/ask.py "Can I get my money back?"` prints a short answer that mentions 14 days and cites `[1]`, then `Sources:` with `[1] refunds.md#Refund window` first, then the cost line: a few hundred input tokens, well under a tenth of a cent on Haiku. Now `uv run scripts/ask.py "Do you offer student discounts?"`: the answer says it does not know, because no document mentions discounts. The similarity numbers are lower than before, which is a hint, but the rule in the system prompt is what makes the model admit it. Run both questions again with `claude-opus-5` as the second argument and compare the cost lines. Commit.

### Step 7: Give the agent the knowledge

Goal: a `search_docs` tool, so the week-4 assistant can answer an email with the real policy.

Before you run this, understand: the agent does not need to know how search works. It needs a tool whose description says when to use it: whenever a reply depends on a policy, a price or a rule. The tool returns the top three chunks as a list of dicts, and `run_tool` turns that into JSON, as it does for `triage_email`. The input is one string called `query`, the same as `search_emails`, so the definition reuses that schema. One sentence in the system prompt tells the model that the documents exist and that a reply should quote its source; without it the model often answers from memory. Retrieval as a tool, called when the model decides it needs it, is how most real agents use RAG.

Add to `tools.py` (put `from ailab import rag` at the top, with the other `ailab` imports):

```python
def search_docs(query: str) -> list[dict]:
    return [{"source": c["source"], "text": c["text"]} for _, c in rag.search(query)]


SEARCH_DOCS = {
    "name": "search_docs",
    "description": "Search the company documents (refunds, invoices, accounts, passwords, plans and prices, export, privacy, support hours) and return the three most relevant sections with their source. Call this whenever a reply depends on a policy, a price or a rule, and quote the source in the reply. Describe the customer's problem in plain words, for example: charged twice for one month.",
    "input_schema": SEARCH_EMAILS["input_schema"],  # the same single input: query
}
```

In `agent.py`, add `"search_docs": inbox.search_docs` to `TOOLS` and `inbox.SEARCH_DOCS` to `DEFS`. Then add one sentence to `SYSTEM`, before `Be brief.`: `When a reply depends on company policy, search the documents and quote the source.`

Check: `rm -f trace.jsonl`, then `uv run scripts/agent.py "The customer in 01.txt was charged twice. What does our refund policy say about that? Draft a reply that quotes the policy and save it."`. You should see `[step 1] read_email({'file': '01.txt'})`, then `[step 2] search_docs({'query': ...})`, then `save_draft` with the file, the reply and the `Save this draft? [y/N]` question. The reply quotes the duplicate-charges rule. Type `y`: the final answer names the policy, `drafts/01.txt` exists, and `cat trace.jsonl` shows three tool lines (the `search_docs` result lists sections of `refunds.md`) and the final answer. A run costs a few cents on Opus. Commit.

### Step 8: Change the chunking and measure again

Goal: the same ten questions against three ways of cutting the documents, and a decision backed by numbers.

Before you run this, understand: chunk size is a trade-off. A chunk that is too big (a whole file) sends paragraphs the question did not ask about: more input tokens on every call, and a source that only names the file. A chunk that is too small (one sentence) is cheap but loses its context: "the second one is a duplicate" means little without the sentence before it. `split_doc` already knows all three ways, `build_index.py` takes the name as its argument, and `eval_search.py` reads the name back from `index.json`.

```bash
uv run scripts/build_index.py file && uv run scripts/eval_search.py
uv run scripts/build_index.py sentence && uv run scripts/eval_search.py
uv run scripts/search.py "I was charged twice"
uv run scripts/build_index.py && uv run scripts/eval_search.py
```

Check: three tables, with `chunks=8`, `chunks=75` and `chunks=25`. With my documents and questions, embeddings scored `hit@3=10/10` with all three, but `file` sent about 2,300 characters per question against about 700 for `heading` and 230 for `sentence`. The third command runs on the sentence index and shows what the model would get: single sentences, cut off from their neighbours. With my documents the top result was the sentence that defines a duplicate, and the sentence that says we refund it in full did not make the top three. The last command rebuilds the heading index, which the rest of the project uses; `index.json` must be the heading one when you finish. Put the three embedding rows (chunking, chunks, hit@3, characters) in your README with one sentence on which chunking you keep and why. Commit.

## Read and watch

- [Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval): read the first half; it draws the pipeline you built and gives the rule for when a knowledge base is small enough to skip retrieval (under about 200,000 tokens).
- [Search results](https://platform.claude.com/docs/en/build-with-claude/search-results): the API's own content block for retrieved chunks, with citations attached to each sentence of the answer; see "If you finish early".
- Video: [What is Retrieval-Augmented Generation (RAG)?](https://www.youtube.com/watch?v=T-D1OfcDW1M) (IBM Technology, 6 min): the whole idea in six minutes; match each part to one of your scripts.

## Turn in

- The `ai-lab` project with `data/docs/`, `data/questions.csv`, `src/ailab/rag.py`, the five new scripts, the updated `tools.py` and `agent.py`, `trace.jsonl` from the Step 7 run, and `drafts/01.txt`. `index.json` is not committed; `uv run scripts/build_index.py` rebuilds it. No key anywhere in Git. README: the two-row table from Step 5, the three-row table from Step 8 with your one-sentence decision, and one line with the week's total API cost.
- Friday demo (10 minutes): `search_kw.py` and `search.py` on `"money back"`; `eval_search.py`; `ask.py` on one question from the documents and one outside them; `agent.py` on the 01.txt question with `y`. End with the cost line.

## Self-check

1. Why do you split documents into chunks instead of sending all eight files with every question?
2. Keyword search scores "money back" at 0.00 against the refund policy. Why, and what does embedding search do differently?
3. Why do you write `questions.csv` before you run `eval_search.py`?
4. In `ask.py`, what stops the model from inventing a student discount?
5. A knowledge base has 10 documents of 300 tokens each. Would you build an index, or send everything with each call? Why?

### Answers

1. Cost (you pay for every input token on every call), focus (one relevant paragraph instead of 24 that are not), and size (a real knowledge base does not fit in a prompt).
2. The chunk contains neither "money" nor "back", and keyword search only matches words; embeddings place the two texts near each other because they mean the same thing.
3. Written after, the questions drift toward what the search already finds, and the number measures nothing; the same reason as the week-3 labels.
4. The system prompt (use only the documents, say "I do not know" when they do not answer) plus the fact that the retrieved chunks say nothing about discounts, so there is nothing to cite.
5. 3,000 tokens is small: put the documents in the system prompt (cached, if you ask many questions) and skip retrieval. Retrieval pays off when the documents are large, change often, or must be filtered per user.

## If you finish early

- No retrieval: put all eight documents in the `system` prompt of `ask.py` with `cache_control`, as in week 3 Step 8 (on Opus, where the prompt is long enough to cache), and compare answers and cost with the retrieval version. With a knowledge base this small, this is a fair alternative; the Contextual Retrieval post says where the line is.
- Real citations: the API has a `search_result` content block (the Search results page). Pass each chunk as one block with `citations: {"enabled": True}` instead of `<doc>` tags, and print the `citations` the model attaches to its text.
