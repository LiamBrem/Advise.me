# Database

Advise.me uses PostgreSQL with the pgvector extension. This document describes every table, its columns, and the reasoning behind design decisions.

---

## Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS vector;
```

All primary keys use UUIDs rather than serial integers to avoid enumerable IDs in API routes.

---

## Tables

### `users`

Stores authenticated user accounts. One row per Pitt student.

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email           TEXT NOT NULL UNIQUE,          -- must be @pitt.edu
    name            TEXT,
    major           TEXT,                          -- extracted from transcript
    standing        TEXT,                          -- freshman / sophomore / junior / senior
    credits_completed INTEGER,                     -- total credits completed
    transcript_uploaded_at TIMESTAMPTZ,            -- null if no transcript uploaded yet
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Notes**
- `email` is validated at the auth layer to enforce `@pitt.edu` domain before a row is ever created
- `major`, `standing`, and `credits_completed` are populated by the transcript parsing pipeline and updated on each re-upload
- `transcript_uploaded_at` is used to show the user when their data was last synced

---

### `transcript_courses`

Stores the individual courses extracted from a user's uploaded transcript. One row per course per user.

```sql
CREATE TABLE transcript_courses (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    course_code     TEXT NOT NULL,                 -- e.g. "CS 1501"
    course_name     TEXT NOT NULL,                 -- e.g. "Algorithm Implementation"
    credits         NUMERIC(3,1) NOT NULL,         -- e.g. 3.0
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transcript_courses_user_id ON transcript_courses(user_id);
```

**Notes**
- All rows for a user are deleted and replaced on each transcript re-upload - no attempt to diff or merge
- Grades are intentionally not stored - not needed for the features we are building and reduces sensitivity of stored data
- `course_code` format follows Pitt convention: department abbreviation + space + number (e.g. `CS 1501`)

---

### `chat_sessions`

Each conversation a user starts is a separate session. Users can have multiple sessions over time.

```sql
CREATE TABLE chat_sessions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           TEXT,                          -- short auto-generated title from first message
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_chat_sessions_user_id ON chat_sessions(user_id);
```

**Notes**
- `title` is generated from the first message of the session for display in the sidebar - e.g. "Am I on track to graduate?"
- `updated_at` is bumped on every new message so sessions can be sorted by most recent activity

---

### `chat_messages`

Stores every message in every session. One row per message.

```sql
CREATE TABLE chat_messages (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id      UUID NOT NULL REFERENCES chat_sessions(id) ON DELETE CASCADE,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant')),
    content         TEXT NOT NULL,
    sources         JSONB,                         -- citations returned with assistant messages
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_chat_messages_session_id ON chat_messages(session_id);
```

**Notes**
- `role` is either `user` or `assistant` - mirrors the OpenAI message format directly
- `sources` is a JSONB array of citations attached to assistant responses, e.g.:
  ```json
  [
    { "source": "CS degree requirements 2024-25", "document_type": "degree_requirements" },
    { "source": "Course catalog Fall 2025", "document_type": "course_catalog" }
  ]
  ```
- Messages are ordered by `created_at` when fetched for context assembly

---

### `knowledge_base_documents`

Tracks the source documents that have been ingested into the knowledge base. One row per document version.

```sql
CREATE TABLE knowledge_base_documents (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title           TEXT NOT NULL,                 -- e.g. "CS Degree Requirements 2024-25"
    document_type   TEXT NOT NULL,                 -- degree_requirements | course_catalog | policy
    major           TEXT,                          -- e.g. "CS" (null if not major-specific)
    school          TEXT,                          -- e.g. "SCI"
    academic_year   TEXT,                          -- e.g. "2024-25"
    semester        TEXT,                          -- e.g. "Fall 2025" (null if not semester-specific)
    is_active       BOOLEAN NOT NULL DEFAULT TRUE, -- only active documents are searched at query time
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Notes**
- Versioning is handled via `is_active` - when a new version of a document is ingested, the old document's `is_active` is set to `false` and a new row is inserted
- At query time, the vector similarity search filters to `is_active = true` chunks only
- Old document versions are retained for audit purposes but never surfaced to users

---

### `knowledge_base_chunks`

The actual chunked and embedded content from each document. This is what pgvector searches against at query time.

```sql
CREATE TABLE knowledge_base_chunks (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    document_id     UUID NOT NULL REFERENCES knowledge_base_documents(id) ON DELETE CASCADE,
    content         TEXT NOT NULL,                 -- raw text of this chunk
    embedding       vector(1536),                  -- OpenAI text-embedding-3-small output
    chunk_index     INTEGER NOT NULL,              -- position of this chunk within the document
    metadata        JSONB,                         -- inherits document metadata for fast filtering
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_chunks_document_id ON knowledge_base_chunks(document_id);
CREATE INDEX idx_chunks_embedding ON knowledge_base_chunks
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

**Notes**
- `embedding` dimension is 1536 to match OpenAI `text-embedding-3-small` output
- `metadata` duplicates key fields from the parent document (type, major, academic year) so the similarity search can filter without a join - e.g. filter to `metadata->>'document_type' = 'course_catalog'` before ranking by cosine similarity
- `chunk_index` preserves the original ordering within a document for debugging and re-ingestion
- The `ivfflat` index enables fast approximate nearest neighbor search. `lists = 100` is a sensible starting value - tune upward as the number of chunks grows

---

## Relationships

```
users
  ├── transcript_courses    (one user → many courses)
  └── chat_sessions         (one user → many sessions)
        └── chat_messages   (one session → many messages)

knowledge_base_documents
  └── knowledge_base_chunks (one document → many chunks)
```

---

## Design Decisions

**UUIDs over serial IDs** - prevents enumerable IDs from leaking information about user counts or record counts through the API.

**Transcript data is relational, not embedded** - completed courses are stored as structured rows in `transcript_courses`, not as text embedded in the vector store. This makes it easy to do precise lookups like "has this user completed CS 1501?" without relying on semantic search.

**Grades not stored** - we only need to know what courses a student has completed and how many credits they carry. Storing grades adds data sensitivity without adding feature value at MVP.

**Document versioning via `is_active`** - simple and auditable. Old versions are never deleted, just deactivated. Makes it easy to see what was active at any point in time without a complex versioning scheme.

**`metadata` duplicated on chunks** - denormalizing document metadata onto each chunk avoids joins at query time and keeps the similarity search fast. The tradeoff is slightly more storage, which is acceptable.