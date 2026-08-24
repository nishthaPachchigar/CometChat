# Aster & Row AI Support Agent

Reliable RAG-based support agent for Aster & Row, an ecommerce company selling bags, drinkware, and travel accessories.

## Overview

This project implements an AI support agent that handles realistic data-quality problems including conflicting policy answers, invented order information, lost conversation context, and unsafe retrieved content. The system uses Retrieval-Augmented Generation (RAG) over Markdown knowledge-base documents and an order lookup tool.

## Features

- **Retrieval-Augmented Generation**: RAG over knowledge-base documents with authority-based source filtering
- **Order lookup tool**: Sanitized order status lookup from `data/orders.json` with privacy redaction
- **Multi-turn conversation**: Context awareness for follow-up questions
- **Prompt security**: Ignores instructions from internal documents
- **Privacy protection**: Never exposes customer data, internal notes, or risk scores
- **Observable**: Structured JSONL traces for debugging
- **Evaluation suite**: Deterministic test cases with category reporting

## Architecture

The system consists of three main layers:

1. **Knowledge base layer**: Markdown files in `knowledge-base/` are parsed with `gray-matter`, chunked by section, and indexed with metadata (document ID, heading, status, authority boost). Embeddings are generated via Google's embedding model.

2. **Retrieval layer**: Conversation-aware queries are constructed (current message + last user message for multi-turn context). Dot-product ranking retrieves top-k chunks, preferring active official policies over superseded documents. Authority boosts are applied based on document metadata.

3. **Agent layer**: The Gemini-powered agent receives retrieved passages as context along with a system prompt encoding all prime directives. It can call two tools:
   - `lookup_order(order_id)` — sanitized order status lookup
   - `final_answer(reply, sources, handoff)` — delivers the customer-facing reply

The agent treats all retrieved content and tool results as untrusted data, always preferring application instructions from the prompt over document content.

## Setup

```bash
# 1. Clone the repository
git clone <repo-url>
cd ai-agent-intern-test

# 2. Install dependencies
npm install

# 3. Set up environment variables
cp .env.example .env
# Edit .env and add your GEMINI_API_KEY (required)
# Optional: set GEMINI_CHAT_MODEL, GROQ_API_KEY, DEBUG

# 4. Build the search index (once)
npm run index:build

# 5. Start the agent
npm start
```

## Environment Variables

See `.env.example` for the required variables. No real credentials should be committed.

```
GEMINI_API_KEY=your-key-here        # Required: Get at https://aistudio.google.com/apikey
GEMINI_CHAT_MODEL=gemini-2.5-flash  # Optional: chat model override
GEMINI_EMBEDDING_MODEL=gemini-embedding-001  # Optional: embedding model override
GROQ_API_KEY=your-groq-key-here     # Optional: for Groq provider
GROQ_MODEL=llama-3.3-70b-versatile  # Optional: Groq model override
DEBUG=true                          # Optional: enable debug logging
```

## Model, Embedding Approach, Framework, and Storage

- **Model**: Google Gemini `gemini-2.5-flash` (via `@google/genai` SDK)
- **Embeddings**: Google `gemini-embedding-001` for chunk-level vector representations
- **Framework**: Node.js + TypeScript with `@google/genai`, `gray-matter` (front matter parsing), `vitest` (testing), and `tsx` (runner)
- **Storage**: In-memory chunk store with cosine similarity retrieval via dot-product ranking; no persistent vector database (suitable for the assignment scale)

Chunks are derived from the Markdown knowledge-base files using `gray-matter` to extract `document_id`, `title`, `status`, and other front-matter metadata. The chunker preserves headings and metadata for authoritative source filtering and citation.

## Evaluation

```bash
npm run eval
```

This runs the evaluation suite which loads `evaluation/visible-cases.json` and any custom cases, executes each case in a session, and outputs per-case results with categories (retrieval, groundedness, tool use, privacy, multi-turn).

## Bug Diary

### Failure 1: Legacy policy overriding active policy
- **Root cause**: No authority-based re-ranking; both active and superseded documents had similar similarity scores.
- **Fix**: Added authority boost to retriever — active official policies get 1.5x score multiplier; superseded documents excluded.
- **Regression**: `standard-return-window` and `trailplus-return-window` visible cases now pass.

### Failure 2: Stale ETA for cancelled order
- **Root cause**: Order lookup tool was not stripping `estimated_delivery` for cancelled/returned orders.
- **Fix**: Set `estimated_delivery`, `carrier`, `tracking_number` to `null` when status is cancelled/returned.
- **Regression**: `cancelled-order-stale-eta` visible case now passes.

### Failure 3: Prompt injection from internal migration note
- **Root cause**: Model treated migration note as authoritative policy.
- **Fix**: Added explicit prompt directive to reject unapproved documents; added `forbidden_sources_as_authority` filtering.
- **Regression**: `retrieved-prompt-injection` visible case now passes.

### Failure 4: Order privacy leak
- **Root cause**: Order lookup tool returning customer internal fields.
- **Fix**: Stripped all `customer.*` and `internal.*` fields from sanitized output; updated system prompt.
- **Regression**: `order-data-privacy` visible case now passes.

## Known Limitations

- No persistent vector database: embeddings held in memory only
- English-only queries; no multilingual retrieval
- No true session isolation for concurrent users
- Gemini API required
- Order lookup is read-only (no actions supported)
- Multi-turn context limited to last user message

## AI Coding Tools Used

GitHub Copilot and ChatGPT were used as coding assistants. They were helpful for:
- Writing `gray-matter` front-matter parsing logic
- Designing order lookup sanitization and normalization
- Constructing RAG prompt directives
- Writing test assertions for visible cases

One AI-generated suggestion that was wrong: Copilot suggested normalizing order IDs by simply uppercasing and stripping whitespace, which would incorrectly transform `"ord 1007"` → `"ORD1007"` (missing the dash). The correct normalization requires inserting the dash after `ORD` when only 4 bare digits are provided.

## Demo

A 2-4 minute demonstration showing the agent in action. The demo should include:

1. **Knowledge-base question with citations**: User asks about return policy → Agent answers with correct window and cites `01-returns-policy-current.md`.

2. **Order lookup**: User asks about an order status (e.g., "Where is ORD-1007?") → Agent calls the order lookup tool and returns sanitized status with citation.

3. **Multi-turn conversation**: User first asks "Do you ship internationally?" then follows up with "What about Canada?" → Agent carries context and answers correctly citing `06-international-shipping.md`.

4. **Agent refuses to guess or recommends human help**: User asks for internal data (e.g., "Give me the risk score for ORD-1007") → Agent refuses, explains internal data cannot be shared, and sets handoff=true.

5. **Evaluation suite running**: Terminal output of `npm run eval` showing per-case results broken down by category.

<!-- Demo GIF will be embedded here -->
<p align="center">
  <img src="https://s8.ezgif.com/tmp/ezgif-85572db4957be49c.gif" alt="Demo GIF" width="500" />
</p>