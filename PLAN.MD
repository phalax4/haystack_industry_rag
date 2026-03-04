# PLAN.md: PDF Indexing API with Haystack & ChromaDB

## 1. Project Overview
This project implements a FastAPI-based microservice that accepts PDF uploads, processes them through a Haystack RAG (Retrieval-Augmented Generation) pipeline, and stores the resulting embeddings in a local **ChromaDB** instance using persistent storage.
This Implementation Plan is for an MVP Prototype, so it does not need production grade features and can simply run locally.

### Core Tech Stack
* **FastAPI**: Web framework for the API layer.
* **Haystack 2.x**: Orchestration for the document processing pipeline.
* **ChromaDB**: Vector database using `PersistentClient` for local file-based storage.
* **Sentence Transformers**: Local embedding generation (e.g., `all-MiniLM-L6-v2`).
* **PyPDF**: Extraction of text from PDF files.

### MVP Key Features
* Leverages Haystack DocumentCleaner
* Leverages Haystack In-built Hybrid Retrieval Model using Reciprocal Rank Fusion 
* Provides FAST API endpoints to upload and retrieve documents from the database using Haystack's Hybrid Retrieval model with a custom document cleaner pipeline that removes unwanted text elements such as headers or footers in PDF files for better retrieval results
* Able to place documents into user defined categories / groups
* FAST API endpoints are well defined so that a front end like TanStack can leverage the backend seamlessly
* Local first application
* Uses UV as the python environment package manager

### Features Not Needed For This Implementation
* User Authentication
* User management or multiple users
* API Authentication
* Docker / Kubernetes
* Concurrency
---

## 2. Directory Structure
```text
.
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI entry point, routes, and dependency injection
│   ├── config.py          # Centralized configuration management
│   ├── models/            # Pydantic models for request/response schemas
│   │   ├── __init__.py
│   │   └── document.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── haystack.py    # Haystack Pipeline & DocumentStore initialization
│   │   └── retrieval.py   # Retrieval pipeline (hybrid search)
│   ├── utils.py           # File validation and temporary storage logic
│   └── middleware/        # Error handling middleware
│       ├── __init__.py
│       └── exceptions.py
├── chroma_data/           # Local directory for ChromaDB persistence (created at runtime)
├── uploads/               # Temporary directory for uploaded PDFs (created at runtime)
├── requirements.txt       # Project dependencies
├── .env                   # Environment variables (Storage paths, Model names)
├── .env.example           # Template for environment variables
├── .gitignore             # Git ignore rules
├── README.md              # Project documentation
├── PLAN.MD                # This implementation plan
└── tests/                 # Test directory (optional, initialized as empty)
    └── __init__.py
```

**Directory Setup Plan**:
1. Create `app/` and subdirectories (`models/`, `services/`, `middleware/`)
2. Create `chroma_data/` and `uploads/` directories (managed by app logic)
3. Create `.env` from `.env.example` template

## 3. Implementation Phases

### Phase 1: Environment & Initialization
1.  **Dependencies**: Create a `requirements.txt` file containing:
    * `haystack-ai`: The core framework for Haystack 2.x
    * `chroma-haystack`: The integration bridge for ChromaDB
    * `pypdf`: Required by Haystack's PDF converter
    * `fastapi`, `uvicorn`, `python-multipart`: For the API layer and file uploads
    * `sentence-transformers`: To run embedding models locally
    * `pydantic`: For request/response validation
    * `python-dotenv`: For environment variable management

2.  **Create Environment Configuration**:
    * Create `.env.example` with template for:
      * `MODEL_NAME=sentence-transformers/all-MiniLM-L6-v2`
      * `CHROMA_PERSIST_PATH=./chroma_data`
      * `UPLOAD_DIR=./uploads`
      * `UVICORN_HOST=0.0.0.0`
      * `UVICORN_PORT=8000`
      * `MAX_FILE_SIZE=10485760` (10MB limit)
      * `PDF_SPLIT_LENGTH=250`
      * `PDF_SPLIT_OVERLAP=30`
      * `TOP_K_RETRIEVAL=5`
      * `RRF_K=60` (for Reciprocal Rank Fusion)
    * Document how users should create `.env` from `.env.example`

3.  **Storage Setup**: 
    * Ensure `chroma_data/` (DB persistence) and `uploads/` (temp file storage) directories are created by the application
    * Update `.gitignore` to exclude `chroma_data/`, `uploads/`, `.env`, and `__pycache__/`

### Phase 2: Haystack Service Layer (`app/services/haystack.py`)
1.  **Initialize ChromaDocumentStore**:
    * Configure `ChromaDocumentStore` with `persist_path` from config
    * *Note:* This triggers the `PersistentClient` internally, ensuring data is written to disk
    * Add connection health check on startup

2.  **Define Indexing Pipeline**:
    * **Converter**: `PyPDFToDocument` to extract text from the uploaded binary
    * **Cleaner**: `DocumentCleaner` with custom rules to remove:
      * Headers, footers, page numbers
      * Unnecessary whitespace and newlines
      * Common artifacts from PDF generation
    * **Splitter**: `DocumentSplitter` using `split_by="word"` with configurable:
      * `split_length=250` (words per chunk)
      * `split_overlap=30` (words overlap)
    * **Embedder**: `SentenceTransformersDocumentEmbedder` using model from config
    * **Writer**: `DocumentWriter` to commit the final objects to the `ChromaDocumentStore`
    * **Error Handling**: Handle cases where PyPDFToDocument returns no text

3.  **Pipeline Initialization Strategy**:
    * Initialize pipeline once on app startup to avoid cold-start latency
    * Wrap initialization in try-except to handle model loading failures gracefully
    * Add startup event to verify DocumentStore connection before serving requests



### Phase 3: API Development (`app/main.py`)
1.  **App Setup**: Initialize FastAPI app and the indexing pipeline on startup
2.  **Custom Exception Middleware**: Create middleware for global error handling and logging
3.  **POST `/index-pdf` Endpoint**:
    * Accept `UploadFile` and validate `.pdf` extension
    * Stream the file to the `uploads/` directory
    * Trigger the Haystack pipeline using the local file path
    * **Cleanup**: Use a `try...finally` block to delete the temporary file from `uploads/` after indexing is complete
    * Return success/failure response with document count
4.  **GET `/health` Endpoint**:
    * Implement a simple check to return total document count
5.  **GET `/documents` Endpoint** (Optional):
    * Return metadata about indexed documents (supports filtering by category)
6.  **GET `/models` Endpoint** (Optional):
    * Return model information and pipeline status

---

## 4. Key Implementation Details & Constraints

### ChromaDB Persistence Logic
To ensure the "local file-based" requirement is met correctly, the `DocumentStore` must be initialized with an explicit path:
```python
from haystack_integrations.document_stores.chroma import ChromaDocumentStore

# This ensures the use of Chroma's PersistentClient
document_store = ChromaDocumentStore(persist_path=config.CHROMA_PERSIST_PATH)
```

### Pipeline Connectivity

The indexing pipeline must connect components in this order for Haystack 2.x:
   * converter -> cleaner
   * cleaner -> splitter
   * splitter -> embedder
   * embedder -> writer

The retrieval pipeline must connect components as follows:
   * text_embedder (query) -> embedding_retriever (semantic search)
   * query_text (query) -> bm25_retriever (keyword search)
   * Both retrievers -> rrf_aggregator -> prompt_builder -> generator

### Error Handling Strategy

1. **Empty PDFs**: Handle cases where PyPDFToDocument returns no text
2. **Concurrent Uploads**: Since ChromaDB's PersistentClient handles file locks, configure FastAPI properly:
   - Use default `uvicorn` settings for optimal concurrency
   - Implement timeout limits for concurrent operations
   - Catch `ChromaWriteException` or similar errors
3. **Model Loading Failures**: 
   - Validate model availability before startup
   - Gracefully exit or log error on startup
4. **File Upload Errors**:
   - Validate file size against `MAX_FILE_SIZE`
   - Validate MIME type
   - Handle disk space issues
5. **Invalid Queries**: Return clear error messages for malformed requests

### API Documentation Strategy

1. **Automatic Documentation**: FastAPI provides `/docs` (Swagger UI) and `/redoc` endpoints
2. **Request/Response Models**: Define Pydantic models for all endpoints:
   - `IndexPDFRequest` and `IndexPDFResponse`
   - `QueryRequest` and `QueryResponse`
   - `HealthResponse`
3. **Example Payloads**: Include example JSON in docstrings for quick testing


-----
8
## 5. Success Criteria

- [ ] API successfully receives a PDF via multipart/form-data
- [ ] PDF is converted into chunks and stored in persistent ChromaDB
- [ ] Restarting the API does not lose the indexed data
- [ ] Errors (invalid PDFs, corrupt files) are handled with appropriate HTTP status codes
- [ ] `/health` endpoint returns correct document count
- [ ] `/query` endpoint returns relevant answers with source attribution
- [ ] Temporary files are properly cleaned up after indexing
- [ ] Environment variables are properly loaded from `.env` file

-----

## 6. Retrieval & Query Pipeline

### Phase 4: Retrieval Implementation (`app/services/retrieval.py`)
1. **Initialize Retrieval Pipeline**:
    * **Embedder**: `SentenceTransformersTextEmbedder` (must use the same model as the Indexing Pipeline, e.g., `all-MiniLM-L6-v2`).
    * **Retriever**: `ChromaEmbeddingRetriever` linked to the existing `ChromaDocumentStore`.
    * **Template**: `PromptBuilder` to structure the LLM prompt (e.g., "Answer the question using the following context: {{ context }}").
    * **Generator**: `OpenAIGenerator` or a local `HuggingFaceLocalGenerator` to synthesize the final answer.

2. **Pipeline Connectivity**:
    * `text_embedder` (query) -> `retriever` (search)
    * `retriever` (docs) -> `prompt_builder` (context)
    * `prompt_builder` -> `generator` (answer)



### Phase 5: API Query Endpoint (`app/main.py`)
1. **POST `/query` Endpoint**:
    * **Input**: A JSON body containing the `question` (string).
    * **Logic**: 
        * Run the retrieval pipeline.
        * Extract the `replies` from the Generator.
        * (Optional) Return the `metadata` from the retrieved documents so the user knows which PDF the answer came from.
    * **Output**: A JSON response with the `answer` and `sources`.

---

## 7. Advanced Configuration & Constraints

### Consistency in Embedding
The LLM must ensure that the `SentenceTransformersTextEmbedder` (used for queries) and the `SentenceTransformersDocumentEmbedder` (used for indexing) utilize the exact same model string. If they differ, the vector math will fail, and search results will be irrelevant.

### Metadata Handling
When indexing, ensure `PyPDFToDocument` preserves the filename in the document metadata. This allows the Retrieval endpoint to return the specific source file name alongside the answer.

### Local vs. Cloud LLM
* **Local Option**: Use `HuggingFaceLocalGenerator` with a model like `Llama-3-8B` if you want a 100% local "air-gapped" solution.
* **Cloud Option**: Use `OpenAIGenerator` for faster development and higher reasoning capabilities, requiring an `OPENAI_API_KEY` in the `.env` file.

-----

## 8. Testing Strategy

### Phase 6: Basic Testing Plan
1. **Integration Tests** (Manual for MVP):
   - Test PDF indexing pipeline end-to-end
   - Test query retrieval with sample documents
   - Test ChromaDB persistence across restarts
2. **Manual Testing Checklist**:
   - [ ] Upload various PDF formats (text, scanned, multi-page)
   - [ ] Query with different question types
   - [ ] Verify source attribution in query responses
   - [ ] Test concurrent uploads (if possible)
   - [ ] Verify cleanup of temporary files
   - [ ] Restart API and verify persistence
   - [ ] Access API documentation at `/docs`
   - [ ] Test health endpoint returns correct document count

-----

## 9. Future Enhancements (Post-MVP)

### 1. API Layer Improvements
- Add rate limiting and authentication headers
- Implement pagination for document listing
- Add search query suggestions/facets
- Support for file deletion and index updates

### 2. Pipeline Enhancements
- Support for multiple document formats (DOCX, TXT, HTML)
- Custom chunking strategies based on document type
- Advanced metadata filtering (date ranges, categories)
- Document versioning and rollback

### 3. Performance Optimizations
- Document batching for bulk uploads
- Asynchronous document processing
- Model quantization for faster inference
- Caching for frequently queried documents

### 4. Monitoring & Observability
- Structured logging with request IDs
- Performance metrics (latency, throughput)
- Error tracking and alerting
- API usage analytics

### 5. Deployment Considerations
- Docker containerization for easy deployment
- CI/CD pipeline setup
- Production-grade security (HTTPS, input sanitization)
- Kubernetes deployment manifests

### 6. Concurrency Configuration

- **Uvicorn Workers**: Start with appropriate worker count based on CPU cores
- **Async/Sync**: Haystack pipelines are synchronous, so request handling may block workers
- **Timeouts**: Set appropriate timeouts for document processing
- **Queueing**: Consider adding a background task queue (e.g., Celery) for very large documents
-----

## 10. Development Setup Instructions

### Initial Setup
```bash
# 1. Clone the repository
git clone <repository-url>
cd haystack_industry_rag

# 2. Create virtual environment using UV (recommended)
uv venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# 3. Install dependencies
uv pip install -r requirements.txt

# 4. Create environment file
cp .env.example .env

# 5. Customize .env with your settings (model paths, ports, etc.)

# 6. Run the application
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

# 7. Access API documentation at http://localhost:8000/docs
```

### Verification Checklist
- [ ] `chroma_data/` directory is created automatically
- [ ] `uploads/` directory is created automatically
- [ ] Model loads successfully on startup
- [ ] `/health` endpoint returns 200 OK
- [ ] `/docs` endpoint is accessible
- [ ] Sample PDF upload works
- [ ] Query endpoint returns meaningful results
