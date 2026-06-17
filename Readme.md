# 📄 PDF RAG API

> Upload a PDF or PowerPoint, ask questions about it, export the results — powered by Gemini embeddings, Pinecone vector search, and a persistent conversation history in PostgreSQL.

---

## What It Does

This is a full **Retrieval-Augmented Generation (RAG)** backend. You upload a document, it gets chunked and embedded into a vector database, and from then on you can ask natural language questions about it. Every exchange is saved to a conversation history you can retrieve later.

On top of RAG, the API doubles as a **document utility suite**: convert PDFs to DOCX, render PDFs as HTML, and export HTML back to PDF or DOCX — all in one service.

---

## Architecture

```
  Document Upload (PDF / PPTX)
          │
          ▼
  ┌───────────────────┐
  │   Text Extraction  │   pypdf (PDF) · python-pptx (PPTX)
  └────────┬──────────┘
           │
           ▼
  ┌───────────────────┐
  │  Text Chunking     │   RecursiveCharacterTextSplitter
  │  1000 chars        │   200 char overlap
  │  200 overlap       │
  └────────┬──────────┘
           │
           ▼
  ┌───────────────────┐
  │ Google Embeddings  │   text-embedding-004
  │  (per chunk)       │   → 1024-d vectors
  └────────┬──────────┘
           │
           ▼
  ┌───────────────────┐
  │  Pinecone Upsert   │   tagged with userId
  │  pdf-rag-index     │
  └───────────────────┘

  ─────────────────────────────────────────

  User Question
       │
       ▼
  ┌──────────────┐     ┌───────────────────┐
  │  Embed Query  │────▶│ Pinecone top-3    │
  └──────────────┘     │ filtered by userId │
                       └────────┬──────────┘
                                │ context chunks
                                ▼
                       ┌───────────────────┐
                       │ Gemini 2.5 Flash   │
                       │ prompt + context   │
                       └────────┬──────────┘
                                │
                                ▼
                       ┌───────────────────┐
                       │  PostgreSQL        │
                       │  save Q + A        │
                       └───────────────────┘
```

---

## API Endpoints

### Document Ingestion

#### `POST /ask`
Upload a PDF or PPTX and register it as a new conversation.

**Request:** `multipart/form-data`

| Field    | Type   | Description                         |
|----------|--------|-------------------------------------|
| `file`   | binary | PDF or PPTX file                    |
| `userid` | string | Query param — owner of this upload  |

**Response:**
```json
{ "conversation_id": "uuid-here" }
```

Internally: extracts text → chunks it → embeds each chunk → upserts to Pinecone with `userId` metadata → creates a conversation record in PostgreSQL.

---

### Querying

#### `POST /query`
Ask a question about an uploaded document.

**Query params:** `userid`, `conversation_id`

**Request body:**
```json
{
  "question": "What are the key findings in section 3?",
  "context": null
}
```

**Response:**
```json
{
  "question": "What are the key findings in section 3?",
  "answer": { ... }
}
```

Retrieves top-3 matching chunks from Pinecone (filtered to the user's documents), builds a prompt, calls Gemini, and saves both the question and answer to the conversation history.

---

#### `POST /resolve`
Ask a free-form question with your own manually provided context — no vector search.

**Request body:**
```json
{
  "question": "Summarize this for me",
  "context": "...your text here..."
}
```

Useful for client-side RAG or when you already have the context and just need the LLM.

---

### Conversation History

#### `GET /conversations`
Fetch all conversations for a user.

**Query param:** `userid`

**Response:**
```json
{
  "conversations": [
    { "id": "...", "userId": "...", "title": "report.pdf", "created_at": "..." }
  ]
}
```

#### `GET /messages`
Fetch the full message history for a conversation.

**Query param:** `conversation_id`

**Response:**
```json
{
  "messages": [
    { "id": 1, "role": "user", "content": "...", "created_at": "..." },
    { "id": 2, "role": "assistant", "content": "...", "created_at": "..." }
  ]
}
```

---

### Document Utilities

#### `POST /ocr`
Render a PDF as structured HTML (preserving layout) using PyMuPDF.

**Request:** `multipart/form-data` — PDF file  
**Response:**
```json
{ "html": "<div class='pdf-container'>...</div>" }
```

Each page is wrapped in `<div class='pdf-page' data-page='N'>`.

---

#### `POST /export-docx`
Convert an HTML string to a downloadable DOCX file.

**Request body:** `{ "html": "<h1>Hello</h1><p>World</p>" }`  
**Response:** `.docx` file download

---

#### `POST /export-pdf`
Convert an HTML string to a downloadable PDF file.

**Request body:** `{ "html": "<h1>Hello</h1><p>World</p>" }`  
**Response:** `.pdf` file download

---

#### `POST /convert-to-docx`
Convert an uploaded PDF file directly to DOCX format.

**Request:** `multipart/form-data` — PDF file (max 32 MB)  
**Response:** `.docx` file download

Validates extension, enforces size limit, converts via `pdf2docx`, streams the result, and cleans up temp files.

---

## Database Schema

Two tables in a PostgreSQL database named `RAG`:

```sql
-- Tracks each uploaded document as a conversation
CREATE TABLE conversations (
    id          UUID PRIMARY KEY,
    user_id     TEXT NOT NULL,
    title       TEXT,              -- the original filename
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Stores every message in a conversation
CREATE TABLE messages (
    id                SERIAL PRIMARY KEY,
    conversation_id   UUID REFERENCES conversations(id),
    role              TEXT,        -- 'user' or 'assistant'
    content           TEXT,
    created_at        TIMESTAMP DEFAULT NOW()
);
```

---

## Setup

### Prerequisites

- Python 3.10+
- PostgreSQL running locally (database: `RAG`, user: `postgres`, password: `postgres`)
- [Pinecone](https://pinecone.io) index named `pdf-rag-index`
- Google AI API key (for Gemini + embeddings)

### Install

```bash
pip install -r requirements.txt
```

**`requirements.txt` should include:**
```
fastapi
uvicorn
langchain-text-splitters
langchain-google-genai
pinecone-client
pypdf
pdf2docx
pydantic
psycopg2-binary
python-dotenv
pymupdf
html2docx
xhtml2pdf
python-pptx
python-multipart
```

### Environment Variables

```env
api_key_rag=your_pinecone_api_key
GOOGLE_API_KEY=your_google_ai_api_key
```

### Pinecone Index

| Setting   | Value           |
|-----------|-----------------|
| Name      | `pdf-rag-index` |
| Dimension | `768`           |
| Metric    | `cosine`        |

> **Note:** `text-embedding-004` outputs 768-d vectors by default. Verify dimension when creating the index.

### Run

```bash
uvicorn app:app --host 0.0.0.0 --port 8000 --reload
```

API docs: `http://localhost:8000/docs`

---

## Project Structure

```
.
├── app.py                  # Main FastAPI application
├── .env                    # API keys (never commit this)
├── uploads/                # Temp storage for PDF→DOCX conversions
└── requirements.txt
```

---

## Known Limitations & Future Work

- **Single DB connection** — `psycopg2` connection is opened once at startup; a connection pool (e.g. `asyncpg`) would be safer under load
- **No user authentication** — `userid` is trusted as a plain query param; no verification
- **Chunk IDs are not unique across uploads** — `chunk-0`, `chunk-1`, etc. will collide if the same user uploads multiple files; IDs should include the `conversation_id`
- **CORS is fully open** — `allow_origins=["*"]` is fine for development; lock this down before deploying
- **DOCX cleanup race condition** — temp DOCX is cleaned up in `finally` before the download completes; a background task scheduler is the proper fix
- **No streaming responses** — LLM answers are returned as a single block; streaming would improve perceived latency

---

## The Pipeline at a Glance

| Step | Tool | Purpose |
|------|------|---------|
| Text extraction | `pypdf`, `python-pptx` | Pull raw text from documents |
| Chunking | LangChain `RecursiveCharacterTextSplitter` | Split into 1000-char chunks with overlap |
| Embedding | Google `text-embedding-004` | Encode chunks and queries into vectors |
| Vector storage | Pinecone | Fast similarity search with metadata filtering |
| Generation | Gemini 2.5 Flash | Answer questions from retrieved context |
| Persistence | PostgreSQL | Store conversations and message history |
| Export | PyMuPDF, html2docx, xhtml2pdf, pdf2docx | Document format conversion |

---

*Built with FastAPI, LangChain, Gemini, and Pinecone. Your PDFs now talk back.*
