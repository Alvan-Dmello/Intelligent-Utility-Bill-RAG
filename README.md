# 💡 Intelligent Utility Bill RAG System

A Python-based, Retrieval-Augmented Generation (RAG) system designed to automate the ingestion, vectorization, and natural language querying of utility bill PDFs stored in Oracle Cloud Infrastructure (OCI) Object Storage.

---

## 🌟 Key Features

* **Automated Data Ingestion:** Uses OCI's S3-compatible API to check for new PDF files via ETag comparison, ensuring idempotency and efficiency.
* **Vectorization Pipeline:** Processes PDF files from OCI, extracts text, chunks the content, and converts it into 768-dimensional vectors using the **`nomic-embed-text-v1.5`** model.
* **High-Performance Vector Store:** Utilizes **Milvus** for efficient storage and **COSINE** similarity search of bill data embeddings.
* **RAG Agent with Tool-Calling:** A LangChain-based LLM agent uses **Tool-Calling** to retrieve contextual data from the vector database before generating an answer, providing accurate, grounded, and citable responses.
* **Citable Outputs:** Search results are structured to provide citation tags (e.g., `[filename#chunk_id]`) back to the user interface.

## 💻 Tech Stack

| Component | Technology | Role |
| :--- | :--- | :--- |
| **Cloud/Storage** | Oracle Cloud Infrastructure (OCI) Object Storage, Boto3 | Storage for raw bills and S3-compatible access. |
| **ETL/Processing** | Python, `pypdf`, `SentenceTransformer` | Orchestration, PDF text extraction, and vector embedding. |
| **Vector Database** | Milvus | Scalable storage and indexing for 768-dimensional vectors. |
| **RAG/LLM Framework** | LangChain, ChatOpenAI (JetStream) | Agentic workflow, tool-calling logic, and chat history management. |
| **Embedding Model** | `nomic-ai/nomic-embed-text-v1.5` | Generates high-quality document and query embeddings. |

---

## 🛠️ Architecture and Deployment

```mermaid
graph TD
classDef etl fill:#DDEBF7,stroke:#369,stroke-width:2px;
classDef rag fill:#FEF0C7,stroke:#E66,stroke-width:2px;
classDef db fill:#FCE9E9,stroke:#F00,stroke-width:3px;
classDef interface fill:#E6FFED,stroke:#0A0;

subgraph ETL Pipeline  'SPACE'
direction LR
A[Manual Upload .pdf bills]:::interface
B(OCI Object Storage)
C[OCI Event Trigger / Cron Job]
D[PDF Text Extraction & Nomic-Embed]

A --> B --> C --> D;
class A,B,C,D etl
end

subgraph RAG Service (Local Deployment)
direction LR
F[User CLI (Command-Line Interface)]:::interface
I[LangChain Agent Orchestration]
H(LLM/Tool-Calling Agent)
E[(Milvus Vector DB)]:::db

F --> I;
I --> H;
H <--> E;
H --> I;

class F,I,H rag
end

%% Link the two subgraphs
D -- Vector Embeddings --> E;

%% Explicitly style the DB and Agent for emphasis (optional, can be removed if classes are enough)
style H fill:#E0E0FF,stroke:#333,stroke-width:2px,rx:8px,ry:8px
style E fill:#FFF0F0,stroke:#F66,stroke-width:3px
```

The system is logically split into two main components based on deployment:

### 1. The ETL (Extract, Transform, Load) Pipeline (OCI Instance)

These scripts run on a scheduled job (e.g., Cron Job) on an Oracle Compute Instance.

| File | Description |
| :--- | :--- |
| `auto_checker.py` | **Ingestion Orchestrator.** Connects to OCI Object Storage (via S3 endpoint) and Milvus. It iterates through the bill bucket, checks the `ETag` of each PDF against the Milvus index, and triggers vectorization *only* for new or modified files. It handles PDF text extraction (`pypdf`). |
| `pdf_to_vector.py` | **Vectorization Core.** Takes raw text from `auto_checker.py`, chunks it using a sliding window (`CHUNK_SIZE=1000`, `CHUNK_OVERLAP=200`), applies the `search_document:` prefix for the embedding model, and performs the Milvus batch insert. |

### 2. The RAG Agent (Command Line Interface)

These scripts run where the user chat interface is hosted, communicating with the deployed Milvus and LLM endpoints.

| File | Description |
| :--- | :--- |
| `rag_get_pdf_data.py` | **Retrieval Tool.** Initializes the `nomic-embed-text` model and the Milvus client. It exposes the **`search_pdfs`** tool (LangChain `@tool`) which takes a user query, embeds it, performs a COSINE search, and formats the hits into a JSON payload for the LLM. |
| `llm_with_rag.py` | **Agent Orchestrator.** Sets up the LangChain chat history and prompt template. It binds the `search_pdfs` tool to the LLM (`tool_llm`), manages the conversation flow, and executes the two-step **Tool-Calling** logic (LLM decides to call the tool, tool runs, results are fed back, LLM answers). |

---

## ⚙️ Quick Setup & Run

### Prerequisites

1.  **OCI Configuration:** Set up an OCI Compute instance, an Object Storage bucket, and acquire the S3-compatible credentials (`ORACLE_S3_ACCESS_KEY`, `ORACLE_S3_SECRET_KEY`, `ORACLE_S3_ENDPOINT`).
2.  **Milvus:** Deploy a Milvus server (e.g., on a separate OCI instance or locally).
3.  **LLM Endpoint:** An accessible endpoint for a tool-calling capable LLM (e.g., a managed service or a self-hosted model exposed via a compatible API like the **JetStream** used in the code).

### Environment Variables

Create a `.env` file for both your ETL and RAG components containing:

```ini
# --- OCI & Milvus (for ETL) ---
ORACLE_S3_ACCESS_KEY="..."
ORACLE_S3_SECRET_KEY="..."
ORACLE_S3_ENDPOINT="https://<namespace>.compat.objectstorage.<region>.oci.customer-oci.com"
ORACLE_INGEST_BUCKET="raw"
MILVUS_HOST="127.0.0.1" # or Milvus IP
MILVUS_PORT="19530"
COLLECTION_NAME="utility_bills"

# --- LLM Endpoint (for RAG) ---
JETSTREAM_BASE="http://<your-llm-service>"
JETSTREAM_API_KEY="..." # required if API is secured
JETSTREAM_MODEL="<your-llm-model>" # e.g., "Llama3-8b-Chat"
