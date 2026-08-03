# Local RAG Chatbot with n8n, Ollama, PostgreSQL and pgvector

A fully local Retrieval-Augmented Generation (RAG) chatbot that answers questions using information retrieved from an indexed PDF document.

The project uses **n8n** for workflow orchestration, **Ollama** for local AI models, **PostgreSQL with pgvector** for vector storage and similarity search, and **Llama 3.2** for answer generation.

> No external AI API is required. Document processing, embeddings, retrieval and answer generation run locally.

---

## Project Overview

A standard Large Language Model can answer general questions from its pretrained knowledge, but it is not automatically connected to private or document-specific information.

This project adds a retrieval layer before the LLM:

1. A PDF is loaded and divided into smaller chunks.
2. Each chunk is converted into an embedding.
3. The chunk text and embedding are stored in PostgreSQL.
4. A user question is converted into an embedding.
5. PostgreSQL retrieves the most relevant document chunks.
6. Llama 3.2 uses the retrieved context to generate the final answer.

The completed indexing workflow created approximately:

- **96 document chunks**
- **96 embeddings**
- **96 PostgreSQL rows**

---

## RAG Architecture

```text
                           INDEXING WORKFLOW

PDF Document
     |
     v
Read File in n8n
     |
     v
Default Data Loader
     |
     v
Recursive Character Text Splitter
     |
     v
EmbeddingGemma
     |
     v
PostgreSQL + pgvector
```

```text
                           RETRIEVAL WORKFLOW

User Question
     |
     v
n8n Chat Trigger
     |
     v
EmbeddingGemma creates question embedding
     |
     v
PostgreSQL compares question with stored vectors
     |
     v
Vector Store Retriever returns relevant chunks
     |
     v
Question and Answer Chain
     |
     v
Llama 3.2 generates the response
```

---

## Four Main Stages

### 1. Load and Prepare

The source PDF is read from the local documents directory. The Default Data Loader extracts readable text, and the Recursive Character Text Splitter divides it into smaller overlapping chunks.

Current splitter configuration:

```text
Chunk size: 800 characters
Chunk overlap: 120 characters
```

### 2. Embed and Store

EmbeddingGemma converts every text chunk into a numerical vector representing its semantic meaning.

PostgreSQL stores:

- Original chunk text
- Embedding vector
- Optional metadata

The `pgvector` extension allows PostgreSQL to store and compare vectors.

### 3. Retrieve Relevant Context

The user's question is converted into an embedding using the same embedding model used during indexing.

PostgreSQL compares the question vector with the stored document vectors and returns the closest matching chunks.

### 4. Generate the Answer

The Question and Answer Chain passes the following to Llama 3.2:

- User question
- Retrieved document chunks
- System instructions

Llama 3.2 then generates a clear response based on the retrieved context.

---

## Technology Stack

| Technology | Purpose |
|---|---|
| **n8n** | Workflow orchestration and chatbot flow |
| **Docker Compose** | Runs n8n and PostgreSQL containers |
| **Ollama** | Hosts the AI models locally |
| **Llama 3.2:3b** | Generates the final answer |
| **EmbeddingGemma** | Creates document and question embeddings |
| **PostgreSQL** | Stores document chunks and related data |
| **pgvector** | Adds vector storage and similarity search to PostgreSQL |
| **Git/GitHub** | Version control and workflow backup |

---

## n8n Workflows

### Workflow 01 — n8n Data Flow Practice

Introduces basic n8n node execution and data movement.

### Workflow 02 — n8n Error Practice

Used to understand workflow errors, logs and debugging.

### Workflow 03 — Ollama Test Chat

Tests communication between n8n and the locally hosted Ollama service.

### Workflow 04 — Read Local File Practice

Confirms that n8n can access files mounted from the Mac into the Docker container.

```text
Mac path:
documents/

Container path:
/home/node/.n8n-files/
```

### Workflow 05 — Document Indexing

Builds the searchable knowledge base.

```text
Read PDF
    |
    v
Extract text
    |
    v
Split into chunks
    |
    v
Generate embeddings
    |
    v
Store in PostgreSQL
```

### Workflow 06 — RAG Chatbot

Retrieves relevant chunks from PostgreSQL and passes them to Llama 3.2 to generate the final answer.

---

## Prerequisites

Install the following before starting:

- Docker Desktop
- Ollama
- Git
- A modern browser

Confirm installation:

```bash
docker --version
docker compose version
ollama --version
git --version
```

---

## Environment Configuration

Create a `.env` file in the project root:

```env
POSTGRES_USER=rag_user
POSTGRES_PASSWORD=replace_with_a_secure_password
POSTGRES_DB=rag_db
```

Create a safe `.env.example` for GitHub:

```env
POSTGRES_USER=rag_user
POSTGRES_PASSWORD=your_password_here
POSTGRES_DB=rag_db
```

Ensure `.env` is ignored:

```gitignore
.env
```

Do not commit real passwords or credentials.

---

## Ollama Models

Download the local chat model:

```bash
ollama pull llama3.2:3b
```

Download the embedding model:

```bash
ollama pull embeddinggemma
```

Check installed models:

```bash
ollama list
```

Expected models:

```text
llama3.2:3b
embeddinggemma
```

Start the Ollama application if required:

```bash
open -a Ollama
```

Test the local Ollama API:

```bash
curl http://localhost:11434/api/tags
```

---

## Start the Project

### 1. Open Docker Desktop

```bash
open -a Docker
```

### 2. Enter the project directory

```bash
cd ~/Desktop/local_rag_week1
```

Update the path if the project is stored elsewhere.

### 3. Start the containers

```bash
docker compose up -d
```

### 4. Check container status

```bash
docker compose ps
```

Expected services:

```text
n8n
postgres
```

### 5. Open n8n

```bash
open http://localhost:5678
```

---

## Enable pgvector

Enable the vector extension once for the PostgreSQL database:

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "CREATE EXTENSION IF NOT EXISTS vector;"
```

Verify it:

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "SELECT extname, extversion FROM pg_extension WHERE extname = 'vector';"
```

---

## n8n Credentials

### Ollama Credential

Use the following base URL from inside the n8n Docker container:

```text
http://host.docker.internal:11434
```

Models:

```text
Chat model: llama3.2:3b
Embedding model: embeddinggemma
```

### PostgreSQL Credential

```text
Host: postgres
Database: rag_db
User: rag_user
Password: value from .env
Port: 5432
SSL: Disabled
```

---

## Import the Workflows

In n8n:

1. Open the workflow menu.
2. Select **Import from File**.
3. Import the JSON files from `n8n-workflows/`.
4. Reconnect local credentials where required.
5. Confirm that all nodes show valid connections.

Export updated workflows back to the same folder after major changes.

---

## Run the Indexing Workflow

Before using the chatbot, run Workflow 05 once.

The workflow should:

```text
Read the PDF
    |
    v
Extract its text
    |
    v
Split it into chunks
    |
    v
Create embeddings
    |
    v
Insert rows into n8n_vectors
```

Verify the indexed row count:

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "SELECT COUNT(*) AS chunks FROM n8n_vectors;"
```

Expected result for the current test document:

```text
96
```

Inspect sample chunks:

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "SELECT id, LENGTH(text) AS characters, LEFT(text, 150) AS preview FROM n8n_vectors LIMIT 5;"
```

---

## Use the RAG Chatbot

Run Workflow 06 through the n8n chat interface.

Example document-related questions:

```text
What is Retrieval-Augmented Generation?

What are retrieval, generation and indexing?

What is the difference between RAG-Sequence and RAG-Token?
```

Expected flow:

```text
Question
    |
    v
Question embedding
    |
    v
Vector similarity search
    |
    v
Relevant chunks
    |
    v
Llama 3.2
    |
    v
Final answer
```

---

## Grounding Prompt

The Question and Answer Chain can use a prompt similar to:

```text
You are a document-grounded question-answering assistant.

Answer the user's question only when the answer is supported by the retrieved context below.

Retrieved context:
{context}

Do not use outside or pretrained knowledge.

If the retrieved context does not contain enough information to answer the question, respond:

I could not find this information in the indexed document.

Do not attempt to provide a general answer.
```

The `{context}` placeholder is required because n8n inserts the retrieved document chunks there.

---


## Useful Commands

### Start everything

```bash
open -a Docker
open -a Ollama
cd ~/Desktop/local_rag_week1
docker compose up -d
docker compose ps
open http://localhost:5678
```

### Stop services safely

```bash
docker compose stop
```

### Start stopped services

```bash
docker compose start
```

### Restart services

```bash
docker compose restart
```

### Stop and remove containers

```bash
docker compose down
```

Named volumes are normally preserved.

### Never use casually

```bash
docker compose down -v
```

This can remove named volumes containing:

- n8n workflows and credentials
- PostgreSQL data
- Stored embeddings

### View logs

```bash
docker compose logs --tail=100 n8n
```

```bash
docker compose logs --tail=100 postgres
```

### Follow logs live

```bash
docker compose logs -f n8n
```

Exit live logs using:

```text
Ctrl + C
```

### Check local documents

```bash
ls -lh documents
```

### Check files inside n8n

```bash
docker compose exec n8n ls -lh /home/node/.n8n-files
```

### Check PostgreSQL readiness

```bash
docker compose exec postgres \
pg_isready -U rag_user -d rag_db
```

### List database tables

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "\dt"
```

### Inspect vector table structure

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "\d n8n_vectors"
```

---

## Avoid Duplicate Indexing

Running the indexing workflow more than once may insert duplicate chunks.

For example:

```text
First run: 96 rows
Second run: 192 rows
```

Check before indexing again:

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "SELECT COUNT(*) FROM n8n_vectors;"
```

To intentionally clear all indexed chunks:

```bash
docker compose exec postgres \
psql -U rag_user -d rag_db \
-c "TRUNCATE TABLE n8n_vectors;"
```

Then run the indexing workflow once.

This deletes the indexed rows but does not delete:

- The local PDF
- n8n workflows
- PostgreSQL itself
- Ollama models

---

## Troubleshooting

### Chatbot answers from general knowledge

The workflow is RAG-enabled, but the LLM may still use its pretrained knowledge.

Use a strict grounding prompt and, in a later version, introduce:

- Similarity thresholds
- Relevance checks
- Source citations
- Refusal logic for unrelated questions


---

## Current Limitations

- The current knowledge base contains one PDF
- Metadata filtering is not enabled
- The chatbot does not yet show page-level citations
- Prompt instructions alone do not guarantee perfect grounding
- Re-indexing is manual
- Duplicate-document detection is not implemented
- A similarity threshold has not yet been added

---

## Future Improvements

- Add multiple company documents
- Introduce validated metadata
- Display document names and page references
- Add similarity-threshold filtering
- Reject unrelated questions before sending them to the LLM
- Add automatic file upload and indexing
- Add duplicate-document detection
- Add conversation memory
- Build a cleaner user-facing chat interface
- Evaluate accuracy using a structured test-question set
- Add role-based access for private company documents

---

## Key Learning

RAG does not replace an LLM.

It adds a retrieval process before answer generation:

```text
Retriever finds the information
+
LLM explains the information
```

In this project:

```text
EmbeddingGemma
= converts text into vectors

PostgreSQL + pgvector
= finds the most relevant document chunks

Llama 3.2
= generates the final answer

n8n
= connects and controls the complete workflow
```

---

## Author

**Ahmad Jabir Dada**

Local RAG learning project built using n8n, Ollama, PostgreSQL and pgvector.
