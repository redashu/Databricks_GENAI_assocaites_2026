# From Delta Tables to a RAG Table

We combined the three Delta tables into one RAG corpus. The key point is not just the transformation itself, but why we did it and what changed.

## What RAG needs

RAG generally follows this flow:

```text
User question
      ↓
Retrieve relevant knowledge
      ↓
Send that knowledge to the LLM
      ↓
Generate the answer
```

Databricks AI Search works from a Delta table and can create embeddings from a text column. The retrieved result is essentially a relevant document or chunk, along with metadata.

## Why the original tables were not yet a good RAG corpus

Our original tables were:

- `incidents`
- `support_tickets`
- `service_catalog`

These are excellent for analytics, but they are still operational data tables rather than semantic knowledge documents.

They look like this:

```text
incident_id | service | severity | status | description
```

That structure is useful for SQL queries, dashboards, and filtering, but it is not yet ideal for semantic retrieval.

## What `rag_documents` does

We transformed each row into a self-contained piece of knowledge.

For example, a row like this:

```text
INC001
payment-api
P1
Resolved
SRE
High
Service returning intermittent 5xx errors
```

becomes a text document such as:

```text
Incident ID: INC001
Service: payment-api
Severity: P1
Status: Resolved
Assigned Team: SRE
Incident Date: ...
Description: Service returning intermittent 5xx errors
Customer Impact: High
Resolution Minutes: ...
```

That full text becomes one retrievable document.

So the transformation is:

```text
1 CSV row
    ↓
1 RAG document
```

## Why do this?

Suppose a user asks:

> What happened with payment-api incidents?

The embedding model needs meaningful textual content to understand the semantic relationship between:

- "payment-api incidents"
- and "Service returning intermittent 5xx errors..."

AI Search can then create embeddings from the text column and retrieve the most relevant records. Databricks describes this pattern as using a Delta table containing text, computing embeddings from it, and using the resulting index for similarity search.

## Why combine all three sources?

We are deliberately creating one knowledge base:

```text
                 RAG Knowledge Base
                        │
             rag_documents
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
      Incidents      Tickets       Services
       1000 rows     1000 rows     1000 rows
          └─────────────┼─────────────┘
                        ↓
                  3000 documents
                        ↓
                    AI Search
```

Now a question can retrieve information from any of the three sources.

For example:

> What is the SLA and history for critical payment services?

The system may retrieve:

- a service catalog row for a critical payment service
- an incident record for a past payment-api outage
- a ticket record describing a related user problem

The LLM then receives those retrieved documents and generates the answer.

## Important concept: `rag_documents` is not yet the embedding index

At this stage, `rag_documents` does not contain embeddings yet.

The current structure is roughly:

```text
rag_documents
├── doc_id
├── doc_type
├── content
└── source_table
```

We are currently here:

```text
CSV
 ↓
Delta tables
 ↓
RAG document table        ← we are here
 ↓
Embedding
 ↓
AI Search index
 ↓
Retriever
 ↓
LLM
```

Databricks AI Search can take the `content` column and compute embeddings during index creation, so we do not necessarily need to manually create an embedding column ourselves.

## Small improvement to the design

I would keep the original source information in the RAG table:

```text
doc_id
doc_type
content
source_table
```

This is useful later because when a result is retrieved, we want to know:

- did it come from an incident?
- a ticket?
- or a service catalog entry?

That metadata can also be used for filtering in AI Search.

Conceptually:

```text
                 RAG Document
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Identity      Content     Metadata
       doc_id       actual text   doc_type
                                  source_table
```

This is the exact purpose of the `rag_documents` table.