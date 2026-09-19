Here is a comprehensive note in Markdown format, designed to be saved directly into Obsidian. It explains the core concepts of building a Retrieval-Augmented Generation (RAG) system using SQLite VSS and Ollama, using simple language, Mermaid diagrams, and SVG graphics.

---

# Building a Local RAG System with SQLite VSS & Ollama

This note explains how to build a **Retrieval-Augmented Generation (RAG)** system that runs entirely on your local machine. We use **SQLite VSS** for the vector search engine and **Ollama** as the local LLM "brain."

## 1. The Big Idea: Why RAG?

LLMs (like Llama 3) are frozen in time. They don't know about your private files or recent events. RAG solves this by giving the LLM a "cheat sheet" of relevant information retrieved from your database before it answers.

**The Flow:** 
1. **Retrieve:** Find relevant chunks of text from your database.
2. **Augment:** Combine the user's question with the retrieved text.
3. **Generate:** The LLM reads the context and writes a factual answer.

```mermaid
graph LR
    A[User Query] --> B[Vector Search]
    B --> C[Retrieve Relevant Chunks]
    C --> D[Build Prompt with Context]
    D --> E[Ollama LLM]
    E --> F[Factual Answer]
```

---

## 2. System Architecture: The Five Pillars

Our RAG system consists of five main components working in concert.

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

### Key Components:
1.  **Vector Database (SQLite VSS):** Stores text chunks and their embeddings.
2.  **Embedding Engine (SentenceTransformers):** Converts text to 384-dimensional vectors.
3.  **Hybrid Search:** Combines vector similarity with keyword search (FTS5).
4.  **LLM Integration (Ollama):** Local inference for generating answers.
5.  **Orchestrator:** Coordinates retrieval and generation.

---

## 3. Database Schema: The Two-Table Pattern

We use a **two-table pattern** in SQLite. One table stores the raw content, and a virtual table stores the vectors.

**Critical Rule:** Use `INSERT OR REPLACE` carefully. It deletes the old row and creates a new one, which can break the VSS index if not handled properly. Always ensure the `rowid` is preserved.

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

## 4. Text Chunking: The Art of Slicing

LLMs have a limited context window. We must break large documents into smaller **chunks**. 

**Strategy:** Use a sliding window with **overlap**. This ensures that important information isn't cut in half at the boundary.

```mermaid
graph TD
    A[Source Text: 500 words] --> B[Chunk 1: Words 0-199]
    A --> C[Chunk 2: Words 150-349]
    A --> D[Chunk 3: Words 300-499]
    
    style B fill:#e1f5fe
    style C fill:#e1f5fe
    style D fill:#e1f5fe
```

*   **Chunk Size:** 200 words.
*   **Overlap:** 50 words.
*   **Why?** If a key sentence spans the end of Chunk 1 and the start of Chunk 2, the overlap ensures it appears in both.

---

## 5. Hybrid Search: The Best of Both Worlds

Relying only on vector search can miss exact keyword matches (like a specific error code or name). We combine **Semantic Search** (VSS) with **Keyword Search** (FTS5).

### The Algorithm:
1.  **Semantic Search:** Find chunks with similar meaning.
2.  **Keyword Search:** Find chunks with exact word matches.
3.  **Normalize:** Scale scores to a 0-1 range.
4.  **Fuse:** Combine scores using a weighted average (e.g., 70% Semantic, 30% Keyword).

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
*   **70% Semantic:** Captures the *intent* and *meaning*.
*   **30% Keyword:** Ensures specific technical terms aren't missed.
*   *Tip:* If your search is too vague, increase the keyword weight.

---

## 6. The RAG Pipeline: Orchestrating the Answer

This is where everything comes together. The pipeline takes a question, retrieves context, and asks the LLM to answer.

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

### The Prompt Structure:
The prompt is carefully crafted to prevent hallucinations (making things up).

```mermaid
graph TD
    subgraph Prompt_Assembly
    A[System Instruction: "Answer ONLY from context"] --> B[Retrieved Information: Source 1, Source 2...]
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
3. For every claim you make, you MUST cite the source number (e.g., [Source 1]).

RETRIEVED INFORMATION:
SOURCE 1: From r/explainlikeimfive... Machine learning is like teaching a computer...
SOURCE 2: From r/learnpython... NumPy is for numerical computing...

USER QUESTION:
What is machine learning?

ANSWER:
```

---

## 7. Visualizing the Vector Space

Here is a visual representation of how text chunks are clustered in vector space.

<svg width="500" height="350" xmlns="http://www.w3.org/2000/svg">
  <!-- Background Grid -->
  <defs>
    <pattern id="grid" width="30" height="30" patternUnits="userSpaceOnUse">
      <path d="M 30 0 L 0 0 0 30" fill="none" stroke="#e0e0e0" stroke-width="1"/>
    </pattern>
  </defs>
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- Axes -->
  <line x1="50" y1="300" x2="450" y2="300" stroke="#333" stroke-width="2" />
  <line x1="50" y1="300" x2="50" y2="50" stroke="#333" stroke-width="2" />
  <text x="460" y="315" font-family="Arial" font-size="14" fill="#333">Dimension 1</text>
  <text x="10" y="40" font-family="Arial" font-size="14" fill="#333">Dimension 2</text>

  <!-- Query Vector -->
  <circle cx="250" cy="150" r="8" fill="#FF5722" />
  <text x="260" y="145" font-family="Arial" font-size="14" font-weight="bold" fill="#FF5722">Query: "machine learning"</text>

  <!-- Cluster 1: Close Matches -->
  <circle cx="230" cy="170" r="6" fill="#4CAF50" />
  <circle cx="270" cy="130" r="6" fill="#4CAF50" />
  <circle cx="240" cy="140" r="6" fill="#4CAF50" />
  <text x="280" y="125" font-family="Arial" font-size="12" fill="#4CAF50">Close Matches</text>

  <!-- Cluster 2: Distant Chunks -->
  <circle cx="100" cy="80" r="6" fill="#9E9E9E" />
  <circle cx="120" cy="100" r="6" fill="#9E9E9E" />
  <circle cx="80" cy="110" r="6" fill="#9E9E9E" />
  <text x="130" y="95" font-family="Arial" font-size="12" fill="#9E9E9E">Distant Chunks</text>

  <!-- Distance Line -->
  <line x1="250" y1="150" x2="230" y2="170" stroke="#FF5722" stroke-width="1" stroke-dasharray="4" />
  <text x="200" y="175" font-family="Arial" font-size="10" fill="#FF5722">L2 Distance</text>
</svg>

*   **Red Dot:** Your search query.
*   **Green Dots:** Text chunks that are semantically similar (Close Matches).
*   **Grey Dots:** Unrelated chunks (Distant Chunks).

---

## 8. Summary Checklist

- [x] **Database:** SQLite with VSS and FTS5 extensions.
- [x] **Schema:** `posts` + `content_chunks` + `chunk_vss` + `chunks_fts`.
- [x] **Chunking:** 200 words with 50-word overlap.
- [x] **Embedding:** `all-MiniLM-L6-v2` (384 dimensions).
- [x] **Search:** Hybrid (70% Semantic + 30% Keyword).
- [x] **LLM:** Ollama (Llama 3.1:8b) with low temperature (0.1).
- [x] **Prompt:** Strict instructions to use ONLY retrieved context.

This architecture provides a private, local, and factual question-answering system. By grounding the LLM in retrieved data, we minimize hallucinations and ensure the answers are based on your actual documents.