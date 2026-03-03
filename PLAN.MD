# PLAN.md: PDF Indexing API with Haystack & ChromaDB

## 1. Project Overview
This project implements a FastAPI-based microservice that accepts PDF uploads, processes them through a Haystack RAG (Retrieval-Augmented Generation) pipeline, and stores the resulting embeddings in a local **ChromaDB** instance using persistent storage.

### Core Tech Stack
* **FastAPI**: Web framework for the API layer.
* **Haystack 2.x**: Orchestration for the document processing pipeline.
* **ChromaDB**: Vector database using `PersistentClient` for local file-based storage.
* **Sentence Transformers**: Local embedding generation (e.g., `all-MiniLM-L6-v2`).
* **PyPDF**: Extraction of text from PDF files.

---

## 2. Directory Structure
```text
.
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI entry point, routes, and dependency injection
│   ├── services/
│   │   ├── __init__.py
│   │   └── haystack.py    # Haystack Pipeline & DocumentStore initialization
│   └── utils.py           # File validation and temporary storage logic
├── chroma_data/           # Local directory for ChromaDB persistence
├── uploads/               # Temporary directory for uploaded PDFs
├── requirements.txt       # Project dependencies
└── .env                   # Environment variables (Storage paths, Model names)
