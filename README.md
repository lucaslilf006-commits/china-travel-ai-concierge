# China Travel AI Concierge — a lightweight RAG demo

A single-file, browser-based chatbot that answers foreign travelers' questions
about booking hotels in China, grounded in a small hand-written knowledge base
built from real hotel-resale experience (previously sold hotel bookings on
Xianyu/闲鱼).

Built as a portfolio project for internship applications in AI/AIGC-adjacent
roles (e.g. Trip.com/Ctrip's AI product & content tracks), to demonstrate a
working understanding of the retrieve → augment → generate pipeline.

## What it does

- Ask a question in the chat box (e.g. "What's the safest way to book a hotel
  in China?").
- The app retrieves the most relevant documents from a small knowledge base
  using real vector embeddings and shows them in a side panel, so the
  retrieval step is visible, not a black box.
- The retrieved documents are injected into the model's system prompt, and
  Claude generates a grounded, conversational answer — it does not just
  paste back the raw document text.
- Documents are risk-tagged (Recommended / Medium risk / High risk /
  Illegal, avoid). A system-prompt rule tells the model to only actively
  recommend "Recommended" channels, mention riskier ones with a clear
  warning, and never recommend the illegal one.
- The model can also call a `check_hotel_price` tool for budget questions,
  demonstrating a minimal agent/tool-use loop.

## Architecture

```
User question
     │
     ├─► Keyword retrieval    (word-overlap scoring — the naive baseline)
     │
     └─► Embedding retrieval  (real vector search: a sentence-embedding
                                model runs client-side via transformers.js/
                                WebAssembly, cosine similarity over the
                                resulting vectors — no API key needed;
                                used for the final answer)
                    │
                    ▼
          Top-k documents injected into the system prompt
                    │
                    ▼
          Claude decides whether to call check_hotel_price
          (emulated via a plain-text JSON protocol — see limitations)
                    │
                    ▼
          Claude generates the final answer (fetch → api.anthropic.com)
```

Both retrieval methods run on every question so they can be compared
side by side in the UI — this was useful for understanding where naive
keyword matching fails (e.g. it misses documents that share no literal
words with the question).

## Tech stack

- Vanilla HTML / CSS / JS, no build step, no framework
- [transformers.js](https://github.com/xenova/transformers.js) running
  `Xenova/all-MiniLM-L6-v2` fully client-side (WebAssembly) for retrieval —
  no API key, no backend
- Anthropic API (`claude-sonnet-4-6`) called directly from the browser via
  `fetch` for the final answer
- Knowledge base is a hard-coded array of ~11 short documents (a stand-in
  for a real vector database at this scale)

## Running it outside Claude.ai (e.g. on GitHub Pages)

Retrieval works anywhere with no setup. Generating the final answer calls
the Anthropic API directly from the browser, which only works without a key
inside Claude.ai's own preview environment. To make the hosted version
demoable (e.g. for a recruiter clicking a GitHub Pages link), the page has
an optional field to paste an Anthropic API key — it's kept in memory in
that browser tab only, never written to the code or any storage, and is
sent only to `api.anthropic.com` using Anthropic's documented
`anthropic-dangerous-direct-browser-access` header for client-side use.

## Known limitations / next steps

- **Risk-dominated queries can crowd out safe recommendations.** When a
  question's wording leans heavily toward "risk" (e.g. "what's my safest
  option"), retrieval sometimes fills all top-k slots with risk-tagged
  documents and misses the "Recommended" channel documents entirely. The
  model doesn't fabricate a source for the gap, it falls back to general
  knowledge, but the answer becomes less grounded than it should be.
  Possible fixes: guarantee at least one "Recommended"-tagged document is
  retrieved when relevant, or rerank with a small diversity constraint
  across tags.
- **The browser demo's API proxy doesn't support native function calling.**
  The first implementation of the price-check tool used the standard
  Anthropic `tools` API parameter; in the Claude.ai preview environment the
  parameter was silently dropped, so the model had no real tool to call —
  but since the system prompt told it a tool existed, it wrote out a
  plausible-looking (but non-functional) `<function_calls>` block as plain
  text, then admitted it couldn't get live data. This is a useful lesson in
  agent design: a model can produce text that *looks like* a tool call
  purely from pattern-matching, even when no real capability is wired up
  behind it — "the model expressing intent to call a tool" and "the system
  actually executing that call" are two different things, and a gap
  between them fails silently unless you check for it. The fix implemented
  here emulates function calling with a plain-text JSON protocol instead
  (the model is asked to output a specific JSON shape when it wants to call
  the tool; the app parses that JSON, runs the function locally, and feeds
  the result back in a second call). A production version with a real API
  key would use native tool calling instead.
- **Knowledge base is hand-written and small (~11 documents).** Fine for a
  demo; a production version would need real ingestion and chunking of a
  much larger corpus.

## Why this project

Built to have a concrete, explainable AI project for internship
applications — most of the value is in being able to walk through each
part of the pipeline (retrieval, grounding, safety guardrails, agent
tool-calling) and its trade-offs, not just in the demo working.
