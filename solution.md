# Solution Overview

This implementation extends the base chat application into a **Retrieval-Augmented Generation (RAG) + Tool-Calling agent system** with real-time streaming via SSE.

The system allows the assistant to:

* Answer questions from internal engineering documents (RAG)
* Search and analyze GitHub repositories (Tool Calling)
* Stream responses, tool activity, sources, and token usage in real time

---

# How to Run

## Prerequisites

* Node.js >= 20
* Yarn >= 4
* Python >= 3.12
* Poetry >= 2

## Environment Variables

Add the following in `api/.env`:

```
OPENAI_API_KEY=sk-...
GITHUB_ACCESS_TOKEN=ghp_...
```

## Run the App

```bash
./dev.sh
```

* UI → http://localhost:5173
* API → http://localhost:8000

## Ingest Knowledge Base

After starting the backend, run:

```
GET http://127.0.0.1:8000/rag/ingest
```

This builds the vector index from markdown documents.

---

# Core Approach

## 1. Agent-Based Architecture

Instead of a simple RAG pipeline, I implemented an **agentic loop**:

1. User query → sent to LLM
2. LLM decides:

   * Answer directly
   * Call a tool
3. If tool is called:

   * Backend executes it
   * Result is fed back to LLM
4. Loop continues until LLM produces final answer
5. Final response is streamed to UI

This enables:

* Multi-step reasoning
* Combining RAG + GitHub tools dynamically

---

# Tools Defined

The following tools are exposed to the LLM:

### 1. `search_knowledge_base`

* Performs semantic search over internal markdown docs
* Backed by ChromaDB
* Returns relevant chunks with metadata

### 2. `search_github_repos`

* Searches GitHub repositories by keyword

### 3. `search_files_in_repo`

* Finds files within a repository

### 4. `get_github_file_content`

* Fetches file content (e.g., README)

---

# RAG Implementation

## Libraries Used

* **ChromaDB** → vector store
* **LangChain** → text splitting + embeddings
* **OpenAI Embeddings** → `text-embedding-3-small`

---

## Chunking Strategy

A **2-stage chunking pipeline** is used:

### Stage 1: Markdown Structure Splitting

* Uses `MarkdownHeaderTextSplitter`
* Preserves document hierarchy (h1, h2, h3)

### Stage 2: Recursive Chunking

* Chunk size: 600
* Overlap: 80
* Uses `RecursiveCharacterTextSplitter`

Each chunk is enriched with:

* `source_file`
* `header_path`
* contextual prefix:

  ```
  [Section: Auth > Token Refresh]
  ```

---

## Similarity Search

* Embedding model: `text-embedding-3-small`
* Vector DB: ChromaDB (persistent)
* Distance metric: cosine similarity
* Top-K retrieval: 5 chunks

---

# Tool Calling Flow

The backend implements a full **tool-calling loop**:

1. Send messages + tool schemas to LLM
2. LLM decides tool calls (`tool_choice="auto"`)
3. Execute tool
4. Return tool result as:

   ```
   role: "tool"
   ```
5. Continue loop until no tool calls remain
6. Stream final response

This allows chaining like:

* RAG → GitHub → synthesis

---

# SSE Streaming Design

The backend streams structured events to the frontend:

## Events

### `token`

* Partial LLM output (real-time text)

### `tool_call`

* Emitted when a tool is invoked
* Payload:

  ```
  { name, args }
  ```

### `sources`

* Retrieved RAG chunks with:

  * document
  * chunk
  * score

### `usage`

* Token usage (prompt + completion)

### `done`

* Final saved message

### `error`

* Error handling

---

## Frontend Handling

* Uses a custom `streamSSE()` client
* Parses event stream manually via `ReadableStream`
* Updates UI incrementally:

  * Streams tokens live
  * Shows tool activity
  * Displays sources
  * Tracks token usage

---

# Frontend Enhancements

## 1. Agent Visibility

The UI surfaces:

* Tool calls being executed
* Queries used for tools
* Retrieved sources
* Token usage

This makes the system transparent and debuggable.

---

## 2. Markdown Rendering

* Integrated `react-markdown`
* Added `remark-gfm` for:

  * tables
  * lists
  * bold formatting

---

## 3. Code Rendering

* Used `react-syntax-highlighter`
* Enables:

  * syntax-highlighted code blocks
  * IDE-like readability

---

# LLM Workflow (Simplified)

1. Receive full conversation history
2. Decide:

   * answer directly
   * call tools
3. Iterate through tool loop
4. Generate final response
5. Stream tokens to UI

Model used:

* `gpt-4o-mini`

---

# Key Design Decisions

## Why Agent Loop (instead of simple RAG)?

* Enables dynamic decision making
* Supports multi-step workflows
* Combines internal + external knowledge

## Why Header-Based Chunking?

* Maintains semantic structure
* Improves retrieval relevance

## Why SSE?

* Real-time UX
* Fine-grained control (tokens, tools, usage)

---

# Trade-offs

### Pros

* Flexible agent system
* Real-time streaming UX
* Clean separation of ingestion and retrieval

### Cons

* No reranking (can reduce retrieval quality)
* Fixed chunk size (not adaptive)
* Potential token growth in long agent loops

---



# Summary

This system transforms a basic chat app into a:

* RAG-powered assistant
* Tool-using agent
* Real-time streaming interface

It demonstrates:

* LLM orchestration
* Retrieval design
* Tool integration
* Frontend streaming UX

---

# Try Complex Queries (Demonstrating Agent Capabilities)

To fully evaluate the system, try the following queries that require **multi-step reasoning, RAG retrieval, and tool chaining**:

---

## 1. RAG + Tool Combination

**Query:**

```
What do we use in our data pipeline spec for CDC? Are there any newer alternatives? Compare them.
```

### What this tests:

* Retrieves internal knowledge (RAG) about CDC (e.g., Debezium)
* Triggers GitHub search for alternative tools
* Combines both into a comparative answer

### Expected behavior:

1. `search_knowledge_base` → find CDC references
2. `search_github_repos` → find modern alternatives
3. Optional:

   * `get_github_file_content` → fetch README details
4. LLM synthesizes comparison

---

## 2. Multi-Tool Deep Dive

**Query:**

```
What are the core components of our data pipeline, and what libraries can we use to build those components? Give me their GitHub README file summary.
```

### What this tests:

* Understanding system architecture from docs (RAG)
* Mapping components → real-world tools
* Fetching and summarizing GitHub READMEs

### Expected behavior:

1. `search_knowledge_base` → extract pipeline components
2. `search_github_repos` → find relevant tools
3. `get_github_file_content` → read README files
4. LLM aggregates into structured answer

---


