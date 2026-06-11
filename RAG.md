# RAG (Retrieval-Augmented Generation)

## Basic Architecture (Databricks-Oriented)

```text
                PDF / DOCX / PPTX
                        |
                        v
               Databricks Volume
                        |
                        v
               ai_parse_document
                        |
                        v
               Semantic Chunking
                        |
                        v
                Embedding Model
                        |
                        v
            Delta Table (chunks + embeddings)
                        |
                        v
         Vector Search (Mosaic AI Vector Search)
                        |
--------------------------------------------------------
                        |
                        v
                  User Question
                        |
                        v
                 Query Embedding
                        |
                        v
                Similarity Search
                        |
                        v
              Top-K Relevant Chunks
                        |
                        v
               Prompt Construction
                        |
                        v
                         LLM
                        |
                        v
                    Final Answer
```

## 1. Basic Definition

### What Is RAG?

RAG stands for:

- Retrieval
- Augmented
- Generation

Instead of relying only on what an LLM learned during training, RAG retrieves relevant information from enterprise documents before generating an answer.

Without retrieval:

```text
Question -> LLM -> Answer
```

With retrieval:

```text
Question -> Vector Search -> Relevant Chunks -> LLM -> Answer
```

### Why Do We Need RAG?

Without RAG:

- LLM knowledge cutoff
- No enterprise-specific knowledge
- Higher hallucination risk

Example question: "What is our company's leave policy?"

With RAG:

```text
Company PDF -> Retrieved Context -> Passed to LLM -> Grounded Answer
```

Benefits:

- Uses private company data
- Reduces hallucinations
- Avoids fine-tuning for knowledge updates
- Easier to keep content current

## 2. Core Components

| Component | Purpose |
| --- | --- |
| Parsing | Extract text from documents |
| Chunking | Break documents into searchable units |
| Embedding | Convert text into vectors |
| Vector DB | Store vectors |
| Retriever | Find relevant chunks |
| LLM | Generate final response |

## 3. Vector Database Options

### Databricks Native (Enterprise)

- Mosaic AI Vector Search

Best when:

- Already using Databricks
- Using Unity Catalog
- Using Delta Lake

Advantages:

- Delta Sync Index
- Unity Catalog governance
- Managed service
- Native RAG integration

### Enterprise Alternatives

- Pinecone: fully managed, popular enterprise choice, cloud-native
- Weaviate: vector and graph capabilities, hybrid search
- Qdrant: developer-friendly, strong metadata filtering

### Open Source Options

- Milvus: very scalable for large deployments
- FAISS: library (not a full DB), popular for experiments
- ChromaDB: lightweight, good for local development

## 4. Embedding Models

### Databricks Options

- Databricks Foundation Model APIs
- Databricks-hosted embeddings (fully managed)

### Open Source Embedding Models

- `bge-large-en`: strong retrieval quality, open source
- `gte-large`: good balance of quality and speed
- `all-MiniLM-L6-v2`: small, fast, and low cost, but lower accuracy

### Commercial Embeddings

- `text-embedding-3-large`: very strong retrieval quality

Tradeoffs:

- API cost
- External dependency

## 5. Databricks-Specific RAG Flow

```text
Databricks Volume
  -> ai_parse_document()
  -> Chunking
  -> Embedding Generation
  -> Delta Table
  -> Mosaic AI Vector Search
  -> Retrieve Top-K Chunks
  -> Foundation Model Endpoint
  -> Answer
```

## Common Interview Questions

### Why Chunking?

Large documents cannot fit efficiently into embeddings or context windows.

### Why Embeddings?

To enable semantic search instead of keyword-only search.

### Why Vector Search?

To find similar meaning, not just exact words.

### Why Top-K?

To return only the most relevant chunks.

Typical values:

- K = 3
- K = 5
- K = 10

### Why RAG Instead of Fine-Tuning?

| RAG | Fine-Tuning |
| --- | --- |
| Knowledge updates are easy | Retraining is required |
| Lower cost | Higher cost |
| Uses enterprise docs directly | Changes model behavior |
| Lower maintenance risk | Higher maintenance overhead |

## 30-Second Databricks Summary

In Databricks, RAG typically starts with documents stored in Volumes. Documents are parsed using `ai_parse_document`, chunked, converted into embeddings, and stored in Delta tables. Mosaic AI Vector Search creates a vector index over these embeddings. When a user asks a question, the query is embedded, relevant chunks are retrieved using vector similarity search, and those chunks are passed to an LLM to generate a grounded response.

Key advantages include reduced hallucinations, access to private enterprise knowledge, and seamless integration with Unity Catalog, Delta Lake, and Mosaic AI.