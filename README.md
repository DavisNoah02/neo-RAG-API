# neo-RAG-API

A local RAG (Retrieval-Augmented Generation) API built with FastAPI, ChromaDB, and Ollama that answers personal questions by grounding AI responses in your own documents — running entirely on your machine with zero cloud costs.

## What is this?

Large language models only know what they were trained on, so they can't answer questions about *you*. This project solves that by building a full RAG pipeline: your documents are chunked, embedded into vectors, and stored in a local vector database. When a question comes in, the most relevant chunks are retrieved, injected into the prompt, and a local LLM generates a grounded answer.

## The RAG Pipeline

```
Question → FastAPI /ask endpoint
              │
              ▼
     1. RETRIEVE  — ChromaDB converts the question to an embedding and
                    finds the 2 most similar chunks from your profile
              │
              ▼
     2. AUGMENT   — Retrieved chunks are combined with the question
                    into a single prompt
              │
              ▼
     3. GENERATE  — qwen2.5:0.5b generates an answer using only
                    the provided context
              │
              ▼
     JSON response: { question, answer, context_used }
```

`context_used` is returned alongside the answer so you can verify exactly which documents grounded the response.

## Project Structure

```
neo-rag_api/
├── main.py                  # FastAPI app — GET /ask endpoint (full RAG pipeline)
├── build_knowledge_base.py  # Loads, chunks, embeds, and stores profile.txt
├── profile.txt              # Source document — the personal knowledge base
├── chroma_db/               # Persistent ChromaDB storage (created at runtime)
└── venv/                    # Python virtual environment
```

## Tech Stack

| Component | Tool | Purpose |
|---|---|---|
| Web framework | **FastAPI** | REST API with auto-generated Swagger UI at `/docs` |
| Server | **Uvicorn** | Runs the FastAPI app (`--reload` for development) |
| Vector database | **ChromaDB** | Stores embeddings, performs semantic search, metadata filtering |
| LLM runtime | **Ollama** | Runs models locally — no cloud API keys required |
| Chat model | **qwen2.5:0.5b** (~400MB) | Generates grounded answers |
| Embedding model | **nomic-embed-text** (~274MB) | Converts text to 768-dimensional vectors for search |

## Prerequisites

- Python 3.13+
- [Ollama](https://ollama.com) installed and running (`curl http://localhost:11434`)

Pull the required models:

```bash
ollama pull qwen2.5:0.5b
ollama pull nomic-embed-text
```

## Setup

```bash
# 1. Create and activate a virtual environment (isolates project dependencies)
python3 -m venv venv
source venv/bin/activate

# 2. Install dependencies
pip install fastapi uvicorn chromadb ollama

# 3. Write your own profile.txt document

# 4. Build the knowledge base (chunks, embeds, and stores profile.txt in ChromaDB)
python build_knowledge_base.py
```

## Running the API

```bash
uvicorn main:app --reload
```

Then open **http://127.0.0.1:8000/docs** for FastAPI's interactive Swagger UI.

### Endpoint

**`GET /ask?question=<your question>`**

```json
{
  "question": "What are my career goals?",
  "answer": "Your career goal is to become a DevOps engineer...",
  "context_used": [
    "My career goal is to become a ...",
    "I'm currently learning about cloud computing, AI, and DevOps."
  ]
}
```

Example via curl:

```bash
curl "http://127.0.0.1:8000/ask?question=What%20is%20my%20name?"
```

## How the Knowledge Base Works

`build_knowledge_base.py` performs the ingestion side of RAG:

1. **Load** — reads `profile.txt`
2. **Chunk** — splits the document into paragraphs (smaller chunks let the AI find the most relevant section instead of stuffing the whole document into the prompt)
3. **Embed** — ChromaDB sends each chunk to `nomic-embed-text` via Ollama, producing a 768-dimensional vector per chunk
4. **Store** — text and vectors are persisted to `./chroma_db` in a collection called `personal_profile`

At query time, the question is embedded with the same model, and ChromaDB performs **cosine similarity search** to find the chunks whose vectors are closest in high-dimensional space — meaning-based retrieval rather than keyword matching.

## Key Concepts

- **RAG (Retrieval-Augmented Generation)** — retrieve relevant context, augment the prompt, generate an answer. Reduces hallucination by grounding responses in your data.
- **Embeddings** — lists of numbers representing text as coordinates in vector space; semantically similar text lands nearby.
- **Vector database** — stores embeddings and answers "what's most similar to this?" queries.
- **Virtual environment** — an isolated Python dependency space so this project's packages never conflict with other projects on your machine.
- **Local-first** — data never leaves your machine, so the pipeline can safely process personal or confidential documents.

## Privacy Note

`profile.txt` in this project is a sample document. In a production system, access controls (API keys, OAuth) would be required before exposing any private data through a queryable API.

## Roadmap

- [ ] **Secret Mission:** extend into a multi-user AI directory — `POST /documents` for profile submissions and a `user` filter on `/ask` using ChromaDB metadata filtering (multi-tenancy)
- [ ] Containerize with Docker
- [ ] Deploy to Kubernetes
- [ ] Automate with GitHub Actions

## Cleanup

```bash
# Stop the server with Ctrl+C, then:
deactivate                          # exit the virtual environment
rm -rf chroma_db                    # delete vector data (project files kept)
ollama rm nomic-embed-text          # optionally free ~274MB (needed for later projects in the series)
```
