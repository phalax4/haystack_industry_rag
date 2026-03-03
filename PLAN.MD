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

## 3. Implementation Phases

### Phase 1: Environment & Initialization
1.  **Dependencies**: Create a `requirements.txt` file containing:
    * `haystack-ai`: The core framework for Haystack 2.x.
    * `chroma-haystack`: The integration bridge for ChromaDB.
    * `pypdf`: Required by Haystack's PDF converter.
    * `fastapi`, `uvicorn`, `python-multipart`: For the API layer and file uploads.
    * `sentence-transformers`: To run embedding models locally.
2.  **Storage Setup**: Ensure the `chroma_data/` (DB persistence) and `uploads/` (temp file storage) directories are initialized and included in `.gitignore`.

### Phase 2: Haystack Service Layer (`app/services/haystack.py`)
1.  **Initialize ChromaDocumentStore**:
    * Configure `ChromaDocumentStore` with `persist_path="./chroma_data"`. 
    * *Note:* This triggers the `PersistentClient` internally, ensuring data is written to disk.
2.  **Define Indexing Pipeline**:
    * **Converter**: `PyPDFToDocument` to extract text from the uploaded binary.
    * **Cleaner**: `DocumentCleaner` to strip whitespace and artifacts.
    * **Splitter**: `DocumentSplitter` using `split_by="word"` with a `split_length` of 250 and `split_overlap` of 30.
    * **Embedder**: `SentenceTransformersDocumentEmbedder` using `sentence-transformers/all-MiniLM-L6-v2`.
    * **Writer**: `DocumentWriter` to commit the final objects to the `ChromaDocumentStore`.



### Phase 3: API Development (`app/main.py`)
1.  **App Setup**: Initialize the FastAPI app and the indexing pipeline on startup to avoid cold-start latency during requests.
2.  **POST `/index-pdf` Endpoint**:
    * Accept `UploadFile` and validate the `.pdf` extension.
    * Stream the file to the `uploads/` directory.
    * Trigger the Haystack pipeline using the local file path.
    * **Cleanup**: Use a `try...finally` block to delete the temporary file from `uploads/` after indexing is complete.
3.  **GET `/health` Endpoint**:
    * Implement a simple check to return the total count of documents currently in the Chroma vector store.

---

## 4. Key Implementation Details & Constraints

### ChromaDB Persistence Logic
To ensure the LLM implements the "local file-based" requirement correctly, the `DocumentStore` must be initialized with a explicit path:
```python
from haystack_integrations.document_stores.chroma import ChromaDocumentStore

# This ensures the use of Chroma's PersistentClient
document_store = ChromaDocumentStore(persist_path="./chroma_data")


### Pipeline Connectivity

The LLM must explicitly connect the components in this order to ensure the data flow is valid for Haystack 2.x:

   * converter -> cleaner

   * cleaner -> splitter

   *  splitter -> embedder

    * embedder -> writer

Error Handling

    Empty PDFs: Handle cases where PyPDFToDocument returns no text.

    Concurrency: Since ChromaDB's PersistentClient handles file locks, ensure the FastAPI app is configured to handle potential write-lock exceptions if multiple uploads happen simultaneously

-----

## 5. Success Criteria

    [ ] API successfully receives a PDF via multipart/form-data.

    [ ] PDF is converted into chunks and stored in ./chroma_data.

    [ ] Restarting the API does not lose the indexed data.

    [ ] Errors (invalid PDFs, corrupt files) are handled with appropriate HTTP status codes.

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
