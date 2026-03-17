# Data & RAG Pipeline

This document describes in detail how data enters Advise.me, how it is processed and stored, and how it is retrieved at query time to generate chat responses.

There are two distinct pipelines:

1. **Ingestion Pipeline** - how source data gets into the knowledge base
2. **Retrieval Pipeline** - how that data is used to answer a user's question at chat time

---

## 1. Ingestion Pipeline

This pipeline is run by the team to populate and update the knowledge base. It is not triggered by users.

### Sources

| Source | Format | How it enters the system | Update frequency |
|---|---|---|---|
| CS degree requirements | Manual document | Team writes and uploads | Per academic year |
| Course catalog and offerings | Scraped HTML | Existing scraper | Each semester |
| Academic policies and deadlines | Manual document | Team writes and uploads | As needed |

### Steps

```
Raw source data (text, HTML, scraped content)
       │
       ▼
── Cleaning ──────────────────────────────────────
Remove HTML tags, normalize whitespace,
strip irrelevant boilerplate
       │
       ▼
── Chunking ──────────────────────────────────────
Split cleaned text into smaller overlapping segments
Target chunk size: ~500 tokens
Overlap: ~50 tokens between chunks
(overlap preserves context across chunk boundaries)
       │
       ▼
── Metadata tagging ──────────────────────────────
Each chunk is tagged with:
  - source (e.g. "CS degree requirements 2024-25")
  - document_type (e.g. "degree_requirements", "course_catalog", "policy")
  - major (e.g. "CS")
  - school (e.g. "SCI")
  - last_updated timestamp
       │
       ▼
── Embedding ─────────────────────────────────────
Each chunk is sent to OpenAI embeddings API
Returns a 1536-dimension vector representing
the semantic meaning of the chunk
       │
       ▼
── Storage ───────────────────────────────────────
Chunk text, metadata, and embedding vector
stored together in Postgres via pgvector
```

### Re-ingestion

When source data changes (e.g. new semester course catalog), the old chunks for that source are deleted and replaced. Each chunk carries a `source` identifier in its metadata to make this straightforward.

---

## 2. Transcript Parsing Pipeline

This pipeline runs when a user uploads their transcript. It is separate from the knowledge base - transcript data is personal to each user and stored in the users table, not in the vector knowledge base.

```
User uploads transcript PDF
       │
       ▼
── PDF parsing ───────────────────────────────────
Extract raw text from PDF
(Pitt registrar PDFs are text-based, not scanned)
       │
       ▼
── Structured extraction ─────────────────────────
Parse raw text to extract:
  - Completed courses (course code, name, grade, credits)
  - Total credits completed
  - Current GPA
  - Academic standing (freshman / sophomore / junior / senior)
  - Declared major
       │
       ▼
── Validation ────────────────────────────────────
Sanity check extracted data
Flag anything that looks malformed or incomplete
       │
       ▼
── Storage ───────────────────────────────────────
Structured data saved to user profile in Postgres
Raw PDF is discarded — never persisted
```

---

## 3. Retrieval Pipeline (RAG)

This pipeline runs on every chat message. It assembles context from the knowledge base and the user's personal data, then passes it to the LLM to generate a grounded response.

```
User sends a message
       │
       ▼
── Input assembly ────────────────────────────────
Fetch user's parsed transcript data from Postgres
Fetch recent chat history for this session
       │
       ▼
── Query embedding ───────────────────────────────
User's message is sent to OpenAI embeddings API
Returns a vector representing the query's meaning
       │
       ▼
── Vector similarity search ──────────────────────
pgvector finds the top-k most semantically similar
chunks from the knowledge base
(k = 5-8 chunks depending on question type)
Filtered by metadata where relevant
e.g. if question is about a specific course,
filter to document_type = "course_catalog"
       │
       ▼
── Context assembly ──────────────────────────────
Build the prompt context from:
  [1] Retrieved knowledge base chunks (with source citations)
  [2] User's transcript data (completed courses, credits, standing)
  [3] Last N messages of chat history
  [4] System instructions (grounding rules, citation requirements,
      uncertainty handling)
       │
       ▼
── LLM call ──────────────────────────────────────
Send assembled context + user message to OpenAI
System prompt enforces:
  - Answer only from provided context
  - Cite the source for every factual claim
  - If context is insufficient, say so explicitly
    and direct user to verify with an advisor
  - Never answer from general knowledge
       │
       ▼
── Response validation ───────────────────────────
Check that response contains citations
Check that response does not exceed confidence
bounds for the context provided
       │
       ▼
── Storage ───────────────────────────────────────
User message and assistant response saved
to chat history in Postgres
       │
       ▼
Response streamed back to frontend
```

---

## Grounding and Uncertainty

The most important property of the retrieval pipeline is that the LLM never answers from general knowledge. Every factual claim must be traceable to a specific chunk in the knowledge base.

When the retrieved chunks do not contain enough information to answer a question confidently, the system responds with something like:

> "I don't have reliable information about that. I'd recommend checking directly with your advisor or the Pitt registrar."

This is intentional. One confidently wrong answer about a graduation requirement is worse than a dozen honest "I don't knows."

---

## Chunking Strategy Notes

Chunk size and overlap are tunable. The starting values (500 tokens, 50 token overlap) are sensible defaults for academic policy text. Course catalog entries are shorter and more structured — these may benefit from a different chunking strategy (one entry per chunk rather than sliding window).

This will be tuned based on retrieval quality once real queries are flowing through the system.