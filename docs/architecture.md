# Architecture

This document describes how the major components of Advise.me fit together at a high level.

---

## Overview

Advise.me is a web application with three main components: a Next.js frontend, a FastAPI backend, and a PostgreSQL database with vector search capabilities. The frontend handles the user interface and authentication. The backend handles all AI and data logic. The database stores both structured relational data and vector embeddings for semantic search.

```
┌─────────────────────────────────────────┐
│              Browser                    │
│           (Next.js frontend)            │
│                                         │
│  - Auth (NextAuth + @pitt.edu)          │
│  - Transcript upload                    │
│  - Chat interface                       │
└────────────────┬────────────────────────┘
                 │ HTTP / REST
┌────────────────▼────────────────────────┐
│           FastAPI Backend               │
│                                         │
│  - Auth validation                      │
│  - Transcript parsing pipeline          │
│  - RAG pipeline                         │
│  - LLM orchestration                    │
│  - Chat history management              │
└────────────────┬────────────────────────┘
                 │ SQLAlchemy / psycopg2
┌────────────────▼────────────────────────┐
│         PostgreSQL + pgvector           │
│                                         │
│  - User accounts                        │
│  - Parsed transcript data               │
│  - Chat sessions and history            │
│  - Knowledge base documents             │
│  - Vector embeddings                    │
└─────────────────────────────────────────┘
```

---

## Authentication Flow

1. User enters their `@pitt.edu` email in the frontend
2. NextAuth validates the email domain — non-Pitt emails are rejected immediately
3. A magic link or Google OAuth flow completes sign in
4. NextAuth issues a session token stored in a secure cookie
5. All subsequent requests from the frontend include this session token
6. The FastAPI backend validates the token on every protected route before processing anything

---

## Transcript Upload and Parsing Pipeline

This runs once when a user uploads their transcript and again each time they re-upload.

```
User uploads PDF
       │
       ▼
Next.js sends file to FastAPI /upload endpoint
       │
       ▼
FastAPI parses PDF
  - Extracts completed courses
  - Extracts credit counts
  - Extracts current standing and declared major
       │
       ▼
Structured data saved to users table in Postgres
       │
       ▼
Raw PDF is discarded — not persisted
```

The raw PDF is never stored. Only the structured data extracted from it is saved to the database. This minimizes FERPA exposure.

---

## Knowledge Base Ingestion Pipeline

This is a backend process run by the team — not triggered by users. It populates the data that the chat draws from.

```
Source data (degree requirements, course catalog, policies)
       │
       ▼
Text is chunked into smaller segments
       │
       ▼
Each chunk is embedded via OpenAI embeddings API
       │
       ▼
Chunks and their embeddings stored in Postgres (pgvector)
alongside metadata — source, document type, major, last updated
```

For MVP, degree requirements are manually ingested. Course catalog data is scraped using the existing scraper.

---

## RAG Pipeline (Chat)

This runs on every chat message from a user.

```
User sends message
       │
       ▼
FastAPI receives message + user session
       │
       ▼
User's structured transcript data is fetched from Postgres
(completed courses, credits, standing)
       │
       ▼
Message is embedded via OpenAI embeddings API
       │
       ▼
pgvector similarity search finds the most relevant
chunks from the knowledge base
       │
       ▼
Context is assembled:
  - Relevant knowledge base chunks
  - User's transcript data (if relevant to the question)
  - Recent chat history
       │
       ▼
LLM generates a response grounded strictly in the assembled context
  - Must cite sources
  - Must express uncertainty if context is insufficient
  - Must not answer from general knowledge
       │
       ▼
Response and updated chat history saved to Postgres
       │
       ▼
Response streamed back to frontend
```

---

## Services Summary

| Service | Technology | Responsibility |
|---|---|---|
| Frontend | Next.js | UI, auth, file upload, chat interface |
| Backend | FastAPI (Python) | RAG pipeline, transcript parsing, LLM orchestration |
| Database | PostgreSQL + pgvector | Relational data, vector embeddings, chat history |
| Embeddings + LLM | OpenAI API | Generating embeddings, chat responses |

---

## Local Development

All services run locally via Docker Compose. See [CONTRIBUTING.md](../CONTRIBUTING.md) for setup instructions.

```
localhost:3000   → Next.js frontend
localhost:8000   → FastAPI backend
localhost:5432   → PostgreSQL
```