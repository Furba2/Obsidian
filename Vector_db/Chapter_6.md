# Building Local RAG System with SQLite VSS & Ollama

>**(RAG)** system that run entirely on local machine. 

>**SQLite VSS** for vector search engine 

>**Ollama** as  local LLM "brain." 

```mermaid
graph TD
    A[User Query] --> B[Vector Search]
    B --> C[Retrieve Relevant Chunks]
    C --> D[Build Prompt with Context]
    D --> E[Ollama LLM]
    E --> F[Factual Answer]
```

---

## System Architecture

```mermaid
graph TD
    subgraph Data_Layer
    A[SQLite VSS] -->|Vector Storage| B[Embedding Engine]
    end

    subgraph Search_Layer
    B --> C[Hybrid Search]
    C -->|Semantic + Keyword| D[Retrieval]
    end

    subgraph Generation_Layer
    D --> E[RAG Orchestrator]
    E -->|Prompt + Context| F[Ollama LLM]
    F --> G[Final Answer]
    end
```

---
1.  **Vector Database (SQLite VSS):** Store text chunks and their embeddings.
2.  **Embedding Engine (SentenceTransformers):** Convert text to 384-dimensional vectors.
3.  **Hybrid Search:** Combine vector similarity with keyword search (FTS5).
4.  **LLM Integration (Ollama):** Local inference for generating answers.
5.  **Orchestrator:** Coordinate retrieval and generation.

---

## Database Schema: Two-Table Pattern

One table store raw content and virtual table store vectors.


```mermaid
erDiagram
    posts ||--o{ content_chunks : "has"
    content_chunks ||--|| chunk_vss : "indexed by"
    content_chunks ||--|| chunks_fts : "indexed by"
    
    posts {
        string post_id PK
        string title
        string content
        string subreddit
        string author
        int score
    }
    
    content_chunks {
        int chunk_id PK
        string post_id FK
        int chunk_index
        string content
        blob chunk_vector "384 dims"
    }
    
    chunk_vss {
        int chunk_id
        vector chunk_vector
    }
    
    chunks_fts {
        int rowid
        string content
    }
```

### SQL Setup:
```sql
-- Main content table
CREATE TABLE posts (
    post_id TEXT PRIMARY KEY,
    title TEXT,
    content TEXT,
    subreddit TEXT,
    author TEXT,
    score INTEGER
);

-- Chunks table with vector BLOB
CREATE TABLE content_chunks (
    chunk_id INTEGER PRIMARY KEY,
    post_id TEXT,
    chunk_index INTEGER,
    content TEXT,
    chunk_vector BLOB,
    FOREIGN KEY (post_id) REFERENCES posts(post_id)
);

-- Virtual table for Vector Search (VSS)
CREATE VIRTUAL TABLE chunk_vss USING vss0(
    chunk_vector(384),
    chunk_id INTEGER
);

-- Virtual table for Keyword Search (FTS5)
CREATE VIRTUAL TABLE chunks_fts USING fts5(
    content,
    content='content_chunks',
    content_rowid='chunk_id'
);
```

---

## Text Chunking

LLM have limited context window. We must break large documents into smaller **chunks**. 

Use sliding window with **overlap**. This ensures that important information isn't cut in half at boundary.

```mermaid
graph TD
    A[500 words] --> B[0-199]
    A --> C[150-349]
    A --> D[300-499]
```

*   **Chunk Size:** 200 words.
*   **Overlap:** 50 words.
*   **Why?** end of Chunk 1 and start of Chunk 2, overlap ensures it appears in both.

---

## Hybrid Search

**Semantic Search** with **Keyword Search** (FTS5).

### The Algorithm:
1.  **Semantic Search:** Find chunks with similar meaning.
2.  **Keyword Search:** Find chunks with exact word match.
3.  **Normalize:** Scale scores to 0-1 range.
4.  **Fuse:** Combine scores using weighted average (e.g., 70% Semantic, 30% Keyword).

```mermaid
graph TD
    A[User Query] --> B[Embedding Model]
    B --> C[Vector Search]
    A --> D[Keyword Tokenizer]
    D --> E[FTS5 Search]
    
    C --> F[Vector Scores 0-1]
    E --> G[Keyword Scores 0-1]
    
    F --> H[Weighted Sum: 0.7 * Vector + 0.3 * Keyword]
    G --> H
    H --> I[Top K Chunks]
    
    style H fill:#f9f,stroke:#333
```

### Why 70/30?
*   **70% Semantic:** Capture *intent* and *meaning*.
*   **30% Keyword:** Ensure specific technical terms aren't missed.
*   *Tip:* If your search is too vague, increase keyword weight.

---

## RAG Pipeline

```mermaid
sequenceDiagram
    participant User
    participant Orchestrator
    participant SQLite
    participant Ollama
    
    User->>Orchestrator: "What is machine learning?"
    Orchestrator->>Orchestrator: Embed Query
    Orchestrator->>SQLite: Hybrid Search (Vector + FTS)
    SQLite-->>Orchestrator: Return Top 5 Chunks
    Orchestrator->>Orchestrator: Format Context
    Orchestrator->>Ollama: Prompt: "Answer using ONLY this context..."
    Ollama-->>Orchestrator: Generated Answer
    Orchestrator-->>User: Display Answer
```

### Prompt Structure
>carefully crafted to prevent hallucination.

```mermaid
graph LR
    subgraph Prompt_Assembly
    A["System Instruction: 'Answer ONLY from context'"] --> B[Retrieved Information: Source 1, Source 2...]
    B --> C[User Question]
    C --> D[Answer Space]
    end
    style A fill:#e1bee7
    style B fill:#bbdefb
    style C fill:#c8e6c9
    style D fill:#ffccbc
```

**Example Prompt:**
```
You are a helpful assistant. Answer the user's question using ONLY the retrieved information below.
1. If the information is not in the context, state that you do not know.
2. Do not use outside knowledge.

USER QUESTION:
What is machine learning?

ANSWER:
```

---