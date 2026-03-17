# Features

Current scope: CS students at Pitt. This document reflects MVP only.

---

## Authentication

Users sign in with a `@pitt.edu` email address via NextAuth. No other email domains are accepted.

- Sign in / sign up with Pitt email
- Magic link or OAuth via Google scoped to `@pitt.edu`
- Session management
- Sign out

Non-Pitt emails are rejected at the auth layer before an account is created.

---

## User Profile

Each user has a profile that stores their academic context. This is what enables personalized answers rather than generic ones.

- View and edit profile
- Upload current transcript (PDF from the Pitt registrar)
- Re-upload transcript when it updates each semester
- Transcript is parsed on upload and stored as structured data — completed courses, credit counts, current standing, declared major

The raw PDF is not stored long term. Only the structured data extracted from it is persisted.

---

## Chat

The core feature. A conversational interface that answers academic questions grounded in verified Pitt data.

- Ask questions in natural language
- Responses are grounded strictly in data from the knowledge base — the model does not answer from general knowledge
- Every answer cites its source (e.g. "according to the 2024-25 CS degree requirements")
- When the system is not confident in an answer it says so and directs the user to verify with an advisor or the relevant office
- Chat history is saved per user and accessible across sessions

**What the chat can answer (MVP)**

- Degree requirements for CS at Pitt
- Course offerings and descriptions for the current semester
- Prerequisites for specific courses
- Questions about a user's personal progress based on their uploaded transcript - e.g. "am I on track to graduate?", "what do I still need to take?"

**What the chat will not answer (MVP)**

- Questions outside of CS at Pitt
- Financial aid, housing, or non-academic topics
- Anything requiring real-time registrar data the system does not have

---

## Knowledge Base (Internal - not user facing)

The data sources the chat draws from. Managed by the team, not by users.

- CS degree requirements (manually ingested, updated each academic year)
- Course catalog and offerings per semester (scraped)
- Academic policies and deadlines relevant to CS students

---

## Out of Scope for MVP

- Other majors or schools at Pitt
- Integration with live Pitt systems (SIS, registrar API)
- Advisor-facing tooling
- Mobile app
- Multi-school support