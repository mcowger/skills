# RAG and the Knowledge system

## Contents
1. [Overview](#overview)
2. [How RAG works](#how-rag-works)
3. [Knowledge Bases](#knowledge-bases)
4. [Vector DBs](#vector-dbs)
5. [Embeddings](#embeddings)
6. [Chunking](#chunking)
7. [Reranking](#reranking)
8. [External content: risk control](#external-content-risk-control)
9. [Setup](#setup)
10. [Optimizing RAG quality](#optimizing-quality)
11. [Common issues](#common-issues)

---

## Overview

RAG (Retrieval-Augmented Generation) in Open WebUI lets models answer based on uploaded documents. Architecture:

```
User's question
    ↓
[Query embedding] → [Search in vector DB] → [Top-K chunks]
    ↓                                                ↓
[Optional: reranking]  ←─────────────────────────┘
    ↓
[Prompt assembly: system context + retrieved chunks + question]
    ↓
[Sent to model] → [Response with citations]
```

---

## How RAG works

### Uploading a document

1. The user uploads a file (PDF, DOCX, TXT, MD, CSV, etc.)
2. Open WebUI extracts the text (via various parsers)
3. The text is split into **chunks** of a given size
4. Each chunk is converted into an **embedding** (a vector of numbers)
5. Embeddings are stored in a **vector DB**

### Search on query

1. The user's question is converted into an embedding
2. The vector DB finds the N nearest chunks (cosine similarity)
3. (Optional) A reranker re-scores the relevance of the retrieved chunks
4. The top-K chunks are added to the prompt as context
5. The model generates a response using this context

The chunks themselves remain untrusted input: they help answer questions about the documents, but must not override the system prompt, tool policy, or the agent's safety rules.

---

## Knowledge Bases

A Knowledge Base is a named collection of documents with metadata.

### Creating via UI

Admin → Knowledge → Create New → Upload files

### Creating via API

```python
import requests

headers = {"Authorization": "Bearer YOUR_TOKEN"}

# 1. Create a knowledge base
kb = requests.post("http://localhost:8080/api/v1/knowledge/", json={
    "name": "Project Documentation",
    "description": "Technical documentation for project X"
}, headers=headers).json()

# 2. Upload a file
with open("doc.pdf", "rb") as f:
    file = requests.post("http://localhost:8080/api/v1/files/",
        files={"file": f}, headers=headers).json()

# 3. Attach the file to the knowledge base
requests.post(f"http://localhost:8080/api/v1/knowledge/{kb['id']}/files", json={
    "file_id": file["id"]
}, headers=headers)
```

### Attaching to a model

A knowledge base can be attached to a specific model via a model card. Then, for every request to that model, the system will automatically search the attached knowledge bases.

### Attaching to a chat

A user can attach a knowledge base to a specific chat via the UI (document icon in the chat).

### Sharing

Knowledge bases support Access Grants — you can share a base with a specific user or group.

---

## Vector DBs

### Supported

| DBMS | Variable | Notes |
|------|-----------|-------------|
| **Chroma** (default) | `VECTOR_DB=chroma` | Built-in, no extra setup |
| **Milvus** | `VECTOR_DB=milvus` | High performance, GPU acceleration |
| **Weaviate** | `VECTOR_DB=weaviate` | GraphQL API, modular |
| **Qdrant** | `VECTOR_DB=qdrant` | Rust, fast, cloud-native |
| **OpenSearch** | `VECTOR_DB=opensearch` | Built on Elasticsearch |
| **Pgvector** | `VECTOR_DB=pgvector` | PostgreSQL extension |

### Chroma (default)

```env
VECTOR_DB=chroma
CHROMA_DATA_DIR=./data/vector_db
```

Stores data locally. Suitable for a single instance. Doesn't scale horizontally.

### Pgvector (recommended for production with PostgreSQL)

```env
VECTOR_DB=pgvector
# Uses the main DATABASE_URL
```

Advantage: everything in a single DB (PostgreSQL), a single backup, a single point of maintenance.

---

## Embeddings

### Local (sentence-transformers)

```env
RAG_EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
SENTENCE_TRANSFORMERS_BACKEND=torch    # torch (CPU) | cuda (GPU) | onnx
```

Popular models:
- `all-MiniLM-L6-v2` — fast, 384 dims, good quality for English
- `all-mpnet-base-v2` — more accurate, 768 dims, slower
- `multilingual-e5-large` — multilingual, including Russian

### External (API)

You can use OpenAI embeddings or other API-compatible services via the admin panel configuration.

### GPU acceleration

```env
SENTENCE_TRANSFORMERS_BACKEND=cuda
SENTENCE_TRANSFORMERS_MODEL_KWARGS={"device": "cuda:0"}
```

---

## Chunking

```env
RAG_CHUNK_SIZE=1500      # chunk size in characters
RAG_CHUNK_OVERLAP=100    # overlap between chunks
```

### How to choose parameters

- **Small chunks (500-1000)**: more precise search, but may lose context
- **Large chunks (1500-3000)**: more context, but less precise search
- **Overlap**: helps avoid losing information at chunk boundaries

Recommendation: start with 1500/100 and tune it for your data.

---

## Reranking

Reranking is a second pass that re-scores the relevance of retrieved chunks. It improves quality but adds latency.

```env
RAG_RERANKING_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
```

How it works:
1. Vector search finds the top-20 chunks
2. The reranker scores each chunk against the query
3. The top-K by the new ranking are returned

---

## External content: risk control

Treat external web content as untrusted input. For internal and regulated environments, leave external retrieval disabled and use only vetted internal knowledge sources.

- Don't upload documents from unverified URLs into a knowledge base without separate review.
- Don't execute shell/SQL/code from a document just because it ended up in the RAG context.
- For sensitive environments, prefer an allowlist of sources and manual curation of the knowledge base.

```env
OFFLINE_MODE=true
```

---

## Setup

### Via the admin panel

Admin → Settings → Documents — here you can configure:
- Embedding model
- Chunking parameters
- Reranking
- Top-K (how many chunks to add to context)

### Via environment variables

```env
# Core
RAG_EMBEDDING_MODEL=sentence-transformers/all-MiniLM-L6-v2
RAG_CHUNK_SIZE=1500
RAG_CHUNK_OVERLAP=100
RAG_EMBEDDING_TIMEOUT=60

# Vector DB
VECTOR_DB=chroma

# Reranking
RAG_RERANKING_MODEL=

# Context
RAG_SYSTEM_CONTEXT=true    # add system context to the RAG prompt
```

---

## Optimizing quality

### 1. Choosing an embedding model

For Russian-language documents, use a multilingual model:
```env
RAG_EMBEDDING_MODEL=intfloat/multilingual-e5-large
```

### 2. Proper chunking

- For structured documents (code, tables): smaller chunk_size (800-1000)
- For narrative text: larger chunk_size (1500-2000)
- Overlap 10-15% of chunk_size

### 3. Enable reranking

Reranking noticeably improves quality. Costs ~100-200ms of extra latency.

### 4. Prepare documents

- Remove junk pages (tables of contents, cover pages)
- If a PDF contains scans, run OCR beforehand
- Split very large documents into logical parts

### 5. Tune Top-K

- Low (3-5): more precise, but may miss relevant information
- High (10-15): more context, but the model may "drown" in irrelevant content

---

## Common issues

### RAG can't find the needed information
1. Verify the file was actually uploaded and indexed
2. Try a different embedding model (especially for non-English text)
3. Reduce chunk_size for more granular search
4. Enable reranking

### Slow indexing
1. Use a GPU: `SENTENCE_TRANSFORMERS_BACKEND=cuda`
2. Increase the timeout: `RAG_EMBEDDING_TIMEOUT=120`
3. Index files in batches rather than all at once

### OOM when uploading large files
1. Increase the container's memory limit
2. Split the file into parts before uploading
3. Use an external embedding service (OpenAI, Cohere)

### Hallucinations despite RAG
1. Verify the chunks are actually relevant (check citations)
2. Add to the model's system prompt: "Answer only based on the provided context"
3. Increase Top-K for more context
