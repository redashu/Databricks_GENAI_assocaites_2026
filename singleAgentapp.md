# Single-Agent App (Databricks)

## Use Case

### HR Policy Assistant

Employees ask questions such as:

- What is the maternity leave policy?
- How many casual leaves do I get?
- What is the reimbursement policy?

The agent answers using company documents.

## End-to-End Architecture

```text
HR PDFs
  -> Databricks Volume
  -> ai_parse_document()
  -> Chunking
  -> Embeddings
  -> Delta Table
  -> Mosaic AI Vector Search
  -> Retrieval Agent
  -> Foundation Model
  -> Serving Endpoint
  -> Chat UI
```

## Step-by-Step Build

### Step 1: Upload Documents

Create a volume:

```sql
CREATE VOLUME hr_docs;
```

Upload:

- `leave_policy.pdf`
- `travel_policy.pdf`
- `insurance_policy.pdf`

Stored in:

- `/catalog/schema/volumes/hr_docs`

### Step 2: Parse Documents

```python
docs_df = spark.read.format("binaryFile").load(volume_path)

from pyspark.sql.functions import expr

parsed_df = docs_df.withColumn(
    "parsed_content",
    expr("""
      ai_parse_document(
          content,
          map("version", "2.0")
      )
    """)
)
```

Output:

- Document
- Extracted text

### Step 3: Chunk Documents

Example: a 100-page PDF becomes multiple chunks.

Typical settings:

- Chunk size: 500 tokens
- Overlap: 50 tokens

Store chunks:

```python
chunk_df.write.saveAsTable("hr_chunks")
```

### Step 4: Generate Embeddings

Use a Databricks embedding model.

Conceptually:

```python
chunk_df.withColumn(
   "embedding",
   ai_embedding(...)
)
```

Example output:

| chunk_id | text | embedding |
| --- | --- | --- |
| 1 | policy text | vector |
| 2 | policy text | vector |

Store embeddings:

```python
saveAsTable("hr_embeddings")
```

### Step 5: Create Vector Search Index

Create an index from the Delta table:

```text
hr_embeddings -> Vector Index
```

Databricks service:

- Mosaic AI Vector Search

Index contains:

- Chunk text
- Embedding
- Metadata

### Step 6: Test Retrieval

Sample question:

- What is maternity leave policy?

Process:

```text
Question -> Embedding -> Vector Search -> Top 5 Chunks
```

Result:

- Relevant HR policy chunks

### Step 7: Create Agent

Agent consists of:

- LLM
- Retriever
- Instructions

Prompt example:

```text
You are an HR assistant.
Answer only from retrieved documents.
If information is unavailable, say you don't know.
```

### Step 8: Add Tools (Optional)

Example tool:

```python
get_leave_balance(emp_id)
```

The agent can combine:

- Retrieval from policy documents
- Tool calls for personalized data

### Step 9: Log Agent

Use MLflow:

```python
mlflow.log_model(...)
```

Logs include:

- Agent
- Prompt
- Retriever
- Configuration

Stored in:

- MLflow artifacts

### Step 10: Register in Unity Catalog

```text
MLflow Model -> UC Model
```

Benefits:

- Versioning
- Governance
- Permissions
- Lineage

Example model name:

- `catalog.schema.hr_agent`

### Step 11: Deploy Serving Endpoint

Create a model serving endpoint.

Example:

- `hr-agent-endpoint`

Result:

- REST API becomes available

### Step 12: Test Endpoint

Request:

```json
{
  "messages": [
    {
      "role": "user",
      "content": "What is maternity leave policy?"
    }
  ]
}
```

Response:

```json
{
  "answer": "Employees are entitled to..."
}
```

### Step 13: Add Evaluation

Evaluate:

- Correctness
- Groundedness
- Relevance
- Safety

Using:

- MLflow Evaluate
- Mosaic AI Evaluation

### Step 14: Production Monitoring

Monitor:

- Latency
- Token usage
- Cost
- Quality
- Feedback

## Agent Bricks Version (Easier Path)

Instead of coding all steps manually:

```text
Create Agent
  -> Select Documents
  -> Select Vector Index
  -> Select LLM
  -> Add Instructions
  -> Deploy
```

Agent Bricks automatically sets up:

- Retriever
- Prompt
- Agent
- Serving endpoint
- Evaluation setup