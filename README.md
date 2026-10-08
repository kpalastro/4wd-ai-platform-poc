# 4WD Supacentre — AI platform proof of concept

A working proof of concept built against the live 4WD Supacentre storefront: a **chat
assistant** and a **voice agent** that reach the real product catalogue, the real help-centre
content and the real store inventory, through **one shared, governed tool layer**.

Built against the AI Platform Engineer brief — online, in-store and over the phone.

---

## Watch the demo
[![Watch on YouTube](https://img.youtube.com/vi/smthb5sDGU4/hqdefault.jpg)](https://youtu.be/smthb5sDGU4)
**[Watch on YouTube →](https://www.youtube.com/watch?v=smthb5sDGU4)** 
— a live Chat Bot and Voice Bot Assistance

---
[![The voice agent on a live call — product search, store lookup and policy answers](https://img.youtube.com/vi/MViqQgkk8-Q/maxresdefault.jpg)](https://youtu.be/MViqQgkk8-Q)
**[Watch on YouTube →](https://youtu.be/MViqQgkk8-Q)** — a live voice call handling product
search, store lookup and policy questions. The agent is speaking to a real Ultravox session
whose tools call back into this repository's Python.

The same recording is committed here as
[`deliverables/4wd_audio_agent_demo_1080p.mp4`](deliverables/4wd_audio_agent_demo_1080p.mp4),
if you would rather watch it without leaving the repository.

## Read the architecture

**[`deliverables/ai-platform-architecture.pdf`](deliverables/ai-platform-architecture.pdf)** — the full reference
architecture: why a platform and not three projects, the seven planes, the integration
plane, three requests traced end to end, build-vs-buy, a twelve-month roadmap, governance,
and the decisions that shape everything else.

The source is [`ai-platform-architecture.html`](ai-platform-architecture.html) if you would
rather read it in a browser.

## Run it

> The POC source lives in `4wd-poc/`. It is held back from this repository for now and will
> be added shortly — the steps below are how to run it once it lands.

```bash
cd 4wd-poc
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env       # add an Anthropic credential and your own Algolia key
python sync_data.py        # product + store snapshots, and the help-centre corpus
uvicorn server:app --reload --port 8000
```

Open <http://127.0.0.1:8000>. The **tool console** on `/voice` runs all six tools with no
API key and no call, which is the fastest way to see what the assistant knows and what it
knows it does not.

## What it does

Six tools, one registry, three channels — chat, voice, and any MCP client:

| Tool | Answers |
|---|---|
| `search_products` | what is sold, what it costs, what is available online |
| `get_product` | the full record for one product |
| `list_categories` | how the catalogue is organised |
| `search_knowledge` | shipping, returns, Click & Collect, warranty — RAG over the real help centre |
| `find_stores` | the nearest stores, ranked |
| `check_store_stock` | live per-store availability across all 45 company stores |

## The three decisions worth arguing about

**Products are tools, not RAG.** The catalogue is already a search index. Copying 400,000
records into a vector store replaces an exact, cheap, always-current lookup with an
approximate, stale one. A price is a fact — you fetch it, you don't recall it.

**Policy is RAG.** Prose that changes on a legal review cycle belongs in semantic search
with citations. Both live behind the same registry, so a guardrail added in one place covers
the chat agent, the voice agent and any MCP client at once.

**Store inventory is reachable — it just is not where it looks.** The storefront is Magento,
and Magento's API is not on the storefront's host: the apex `/graphql` returns 403 from
CloudFront, which makes it look closed. The SPA actually talks to a different host that
needs no key and serves live per-store availability for all 45 stores. The first version of
this POC had a tool that apologised for not knowing. It now answers — and the grounding
check that used to flag every store claim now passes correct ones while still blocking
invented ones.

## What it deliberately does not do

- **No vehicle fitment claims.** The catalogue carries no make, model, year or variant. Ask
  whether a part fits a Hilux and the assistant says it cannot confirm it, rather than
  guessing. An assistant that guesses here is worse than one that doesn't answer.
- **No opening hours.** The store API does not expose them.
- **No third-party stockists.** The ~144 stockists publish no stock at all.
- **No stock holds.** The tool reports stock; it does not reserve it.

## Grounding

`core/guardrails.py` checks every dollar figure, SKU and store name in an answer against the
tool results that produced them, **including text streamed before a tool returns** — because
that text is already on the customer's screen. A failure costs one re-prompt; an unresolved
failure is surfaced rather than hidden.

Tests, all offline — no key, no network, no cost:

```bash
python tests/test_agent_loop.py     # the loop against a scripted fake client
python tests/test_catalogue.py      # stock-flag coercion and the store-only rule
python tests/test_stores.py         # postcode ranking and the never-dump rule
python tests/test_voice_prompt.py   # the voice turn-taking rule
node tests/test_voice_contract.mjs  # the client-tool contract, real Ultravox SDK
```

## Layout

```
deliverables/
  ai-platform-architecture.pdf    the reference architecture (rendered)
  4wd_audio_agent_demo_1080p.mp4  the demo recording
ai-platform-architecture.html   the reference architecture (source)
4wd-poc/                        held back from this repo for now
  config.py          env-driven settings; one place to change model, effort, endpoints
  catalogue.py       Algolia client, normalisation, offline snapshot fallback
  stores.py          Magento client, postcode->state ranking, store directory
  sync_data.py       builds the product, corpus and store snapshots
  mcp_server.py      the same tools over MCP stdio
  server.py          /api/chat (SSE), /api/voice/session, /api/tool/{name}, pages
  core/
    tools.py         the tool registry - the only place channels meet systems of record
    agent.py         the tool-use loop, streaming, grounding, fallback
    guardrails.py    grounding verification
    bm25.py          the RAG ranker (a swappable stand-in for a vector store)
  tests/             the five suites above
  static/            index.html, chat.html, voice.html
```

## Caveats

This is a proof of concept. It holds chat history in process memory, has no authentication,
and its RAG ranker is BM25 rather than embeddings — all named in the README inside `4wd-poc/`
along with the rest. The tool layer, the grounding checks and the architecture are the parts
meant to be judged.
