# 🤖 RAG Assistant with n8n, Ollama & Supabase

A Retrieval-Augmented Generation (RAG) assistant built using [n8n](https://n8n.io), [Ollama](https://ollama.ai), and [Supabase](https://supabase.com). This project enables local document processing and semantic retrieval using open-source AI models with full control over execution via Docker Desktop.

---

## 🧰 Features

- Semantic chunk and embed user-uploaded documents
- Store semantic embeddings in Supabase (via pgvector)
- Use RAG to answer questions based on your knowledge base
- Local Execution, private, and **open-source**

---

## 📦 Prerequisites

Install these before starting:

| Tool            | Purpose                            | Link                                       |
|-----------------|------------------------------------|--------------------------------------------|
| Docker Desktop  | Containerized n8n setup            | [Download](https://www.docker.com/products/docker-desktop) |
| Ollama          | Run local LLMs and embedding models | [Download](https://ollama.ai/download)     |
| Supabase        | Vector DB for embeddings           | [Signup](https://supabase.com)             |

---

## 🧱 Step-by-Step Setup

### 1. ✅ Install Ollama & Models

Install Ollama and open your terminal:

```bash
ollama pull qwen2.5
ollama pull deepseek-r1
ollama pull nomic-embed-text
```

Leave Ollama running:

```bash
ollama serve
```
⚠️ **Make sure Ollama is running and listening at** [http://localhost:11434](http://localhost:11434)

---

## 2. 🐳 Set Up n8n Using Docker Desktop (GUI Method)

### a. Open Docker Desktop

- Go to **"Containers / Apps"**
- Click **Add Container**

### b. Create n8n Container

- **Image**: `n8nio/n8n`
- **Name**: `n8n`
- **Ports**: Map `5678 → 5678`
- **Volumes**:
  - Create a persistent volume:
    - **Name**: `n8n_data`
    - **Mount to**: `/home/node/.n8n`

### c. Start the Container

Once launched, open your browser and go to:  
👉 [http://localhost:5678](http://localhost:5678)

Create your user account.

---

## 3. 📥 Import the Workflow

### a. Download the Workflow

Download `rag-agent-workflow.json` from this repo.

### b. Import into n8n

1. In n8n, go to **"Workflows" → "Import from File"**
2. Select `rag-agent-workflow.json`
3. Click **Import**

---

## 4. 🔐 Set Up Required Credentials in n8n

### a. Ollama

1. Go to **Credentials**
2. Click **+ New Credential**
3. Choose **HTTP Request Auth**
4. Fill in:
   - **Name**: `Ollama Local`
   - **Base URL**: `http://host.docker.internal:11434`

### b. Supabase

#### In Supabase Dashboard:

1. Go to **Project Settings → API**
2. Copy the following:
   - **Project URL**
   - **Service Role Key**

#### In n8n:

1. Add a new **HTTP Request Auth** credential
2. Fill in:
   - **Base URL**: Your Supabase URL
   - **Header**:  
     `apikey: <your-service-role-key>`

---

## 5. 🧠 Create Supabase Vector Table

Open you supabase SQL editor and run:

```sql
-- Enable the pgvector extension to work with embedding vectors
create extension vector;

-- Create a table to store your documents
create table documents (
  id bigserial primary key,
  content text, -- corresponds to Document.pageContent
  metadata jsonb, -- corresponds to Document.metadata
  embedding vector(768) -- 1536 works for OpenAI embeddings, change if needed
);

-- Create a function to search for documents
create function match_documents (
  query_embedding vector(768),
  match_count int default null,
  filter jsonb DEFAULT '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
#variable_conflict use_column
begin
  return query
  select
    id,
    content,
    metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

**Note:** Update the `VECTOR(768)` dimension if your embedding model outputs a different size of embedding vector, leave as is for `nomic-embed-text`.

## 🧪 Running the Assistant

### 📝 Submitting Notes
1. Open your workflow in n8n.

2. Click **Execute Workflow.

3. Fill in:

    - **Title:** Title of your note

    - **Content:** Raw body text

4. Click **Execute Workflow.

5. The workflow will:

    - Chunk content using Qwen 2.5

    - Embed using nomic-embed-text

    - Store each chunk in Supabase

## 🔍 Asking Questions
1. Click on the chat icon in the `RAG Access Agent`
2. The assistant will:
     - Answer directly if confident
     - Or fetch from supabase for additional context.

## 💡 Tips

  - host.docker.internal lets Docker containers access services running on your host (like Ollama).

  - Always run ollama before using n8n (just typing Ollama rather than ollama serve can be enough).

  - You can install additional models for different tasks and update the n8n workflow accordingly.
