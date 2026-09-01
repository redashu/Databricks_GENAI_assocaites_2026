# Chunking and RAG Options

## 1. What exactly is chunking?

Suppose you have a 100-page PDF:

```text
PDF
 ↓
Parsing
 ↓
Clean text
 ↓
CHUNKING
 ↓
Chunk 1
Chunk 2
Chunk 3
...
 ↓
Embedding
 ↓
Vector Search
```

**The important point is:** The chunk is usually the unit that gets embedded and retrieved.

So bad chunk boundaries can create bad embeddings and therefore bad retrieval. Databricks specifically identifies chunking, parsing, metadata, and embedding as important parts of RAG quality.

## 2. Fixed-size / token-based chunking

The simplest approach. You say: "Every 500 tokens becomes a chunk."

**Example:**

```text
Original:
The refund policy allows customers to request
a refund within 30 days of purchase. The product
must be returned in its original condition...
                 ↓
Chunk size = 50 tokens

Chunk 1:
"The refund policy allows customers to request
a refund within..."

Chunk 2:
"30 days of purchase. The product must be
returned..."
```

The boundary doesn't necessarily care about meaning.

### With overlap

Instead of:

```text
Chunk 1: 1 ─────── 500
Chunk 2:          501 ─────── 1000
```

you might use:

```text
Chunk 1: 1 ─────── 500
Chunk 2:       451 ─────── 950
Chunk 3:             901 ─────── 1400

chunk_size = 500
overlap    = 50
```

### Advantages

- Very simple
- Fast
- Predictable
- Easy to scale
- Easy to implement

### Problems

It can split:

- sentences
- paragraphs
- tables
- code
- arguments

in the middle.

Databricks recommends fixed-size chunking as a starting point but notes that arbitrary character/token boundaries often don't preserve semantic coherence.

### When to use it

Good for:

- quick POC
- highly uniform documents
- simple text
- very large corpora where simplicity matters
## 3. Character-based chunking

Instead of tokens:

```text
chunk_size = 1000 characters

Document
 ↓
1000 chars
 ↓
Chunk
 ↓
1000 chars
 ↓
Chunk
```

This is similar to fixed-token chunking, but the unit is characters.

### Problem

1000 characters don't represent a consistent amount of semantic information.

For example:

- `"aaaaaaaaaaaaaaaa..."`
- `"complex technical explanation..."`

have very different information densities.

So token-based sizing is generally more meaningful for LLM-oriented systems, although character splitters are widely used as practical implementations.

## 4. Sentence-based chunking

Now we become more intelligent. Instead of splitting every 500 characters, we say: **Never break in the middle of a sentence.**

**Example:**

```text
Sentence 1
Sentence 2
Sentence 3
Sentence 4
Sentence 5
Sentence 6

Create:
Chunk 1:
  Sentence 1
  Sentence 2
  Sentence 3

Chunk 2:
  Sentence 4
  Sentence 5
  Sentence 6
```

You can still impose a maximum:

```text
maximum = 500 tokens
```

but only split between sentences.

### Advantage

Much better semantic integrity than arbitrary character splitting.

### Disadvantage

- A single sentence may be 800 tokens and exceed your desired chunk size.
- Sentences don't necessarily correspond to topics.

## 5. Paragraph-based chunking

This is another simple but useful strategy.

**Example:**

```markdown
# Refund Policy

Customers can request refunds within 30 days.

Products must be returned in original condition.

Digital products are not eligible for refunds.

Refunds are processed within 5 business days.
```

Instead of arbitrary splitting, each paragraph becomes a potential chunk:

```text
Chunk 1:
  Customers can request refunds within 30 days.

Chunk 2:
  Products must be returned in original condition.

Chunk 3:
  Digital products are not eligible...
```

### Advantage

Paragraphs often represent a coherent thought.

### Problem

Paragraph sizes vary:

- Paragraph A = 30 tokens
- Paragraph B = 900 tokens

You may need a maximum size.

Databricks explicitly lists paragraph-based chunking as a strategy because paragraph boundaries often preserve semantic coherence.

## 6. Recursive chunking ⭐

This is one of the most useful strategies to understand.

Instead of blindly splitting at 500 tokens, you define a hierarchy of separators:

1. Section
2. Paragraph
3. Sentence
4. Word
5. Character

The algorithm basically says: **Try to split at the largest/nicest boundary first. If the resulting piece is still too large, go down to the next level.**

**Example:**

```text
Document
   ↓
Section
   ↓
Paragraph
   ↓
Sentence
   ↓
Word
```

Suppose your target is 500 tokens:

```text
Section
  ↓
1200 tokens ❌
  ↓
Paragraphs
  ↓
600 tokens ❌
  ↓
Sentences
  ↓
450 tokens ✅
```

So you get: **Chunk 1 = 450 tokens** rather than arbitrarily cutting the 1200-token section.

### Why it's powerful

It combines:

- Structure awareness
- Size control

Databricks and current RAG guidance identify recursive splitting as a practical strategy for heterogeneous documents.

### When to use it

This is my default starting point for general documents.

## 7. Structure-aware / format-specific chunking ⭐

This is extremely important for real-world RAG.

Different documents have different structures. **Don't treat the whole thing as plain text.** Use the document's inherent structure.

### Markdown

```markdown
# Introduction

...

## Installation

...

## Configuration

...

## Troubleshooting

...
```

Use:

```text
Heading
 ↓
Section
 ↓
Subsection
```

**Chunk Example:**

```text
Document: Databricks Guide
Section: Vector Search
Subsection: Creating an Index
...
```

### HTML

Use structural tags as boundaries:

- `<h1>`, `<h2>`, `<h3>`
- `<section>`
- `<p>`

### PDF

Use:

```text
Page
 ↓
Title
 ↓
Heading
 ↓
Paragraph
 ↓
Table
```

### Code

Use:

```text
Class
 ↓
Method
 ↓
Function
 ↓
Code block
```

Rather than splitting every 500 characters.

Databricks explicitly recommends preserving document structure such as headings, lists and tables during data preparation.

## 8. Semantic chunking ⭐⭐⭐

Now we get to the more advanced strategy.

Instead of asking "Where is the next paragraph?" we ask: **"Where does the topic/meaning change?"**

**Consider:**

```text
Sentence 1:
Python is commonly used for data engineering.

Sentence 2:
PySpark provides distributed DataFrame processing.

Sentence 3:
Spark SQL can execute queries over DataFrames.

Sentence 4:
The company's refund policy allows returns within 30 days.

Sentence 5:
Customers must provide proof of purchase.
```

There is a semantic shift:

```text
        Data Engineering
S1 ─── S2 ─── S3
              │
              │ topic shift
              ↓
        Refund Policy
        S4 ─── S5
```

So semantic chunking creates:

```text
Chunk 1:
  S1 + S2 + S3

Chunk 2:
  S4 + S5
```

### How does it detect the topic change?

Typically:

```text
Sentences
   ↓
Embeddings
   ↓
Compare neighboring embeddings
   ↓
Similarity / distance
   ↓
Large semantic change?
   ↓
Create boundary
```

Databricks describes semantic chunking as grouping sentences by similarity and using embeddings to find natural boundaries.

### Important connection

You asked: "In semantic search, are we not using embeddings?"

**Don't confuse:**

- **Semantic SEARCH**: Embedding query + chunks → Find relevant chunks
- **Semantic CHUNKING**: Embedding sentences/segments → Find topic boundaries

Both can use embeddings, but for different purposes.
## 9. LLM-based / agentic chunking

This is a more advanced approach.

Instead of deterministic rules (500 tokens) or embeddings (semantic similarity), you ask an LLM:

> "Read this document and identify meaningful sections that should be independently retrievable."

**Example:**

```text
LLM

Document:
100-page legal contract
        ↓
Output:
"Section 1: Payment Terms"
"Section 2: Liability"
"Section 3: Termination"
"Section 4: Confidentiality"
...
```

The LLM itself decides boundaries.

### Advantages

- Potentially very good semantic understanding

### Disadvantages

- More expensive
- More latency
- Less deterministic
- Harder to scale
- Requires careful evaluation

I would not make this your default.

## 10. Sliding-window chunking

This is closely related to fixed-size chunking with overlap.

**Example:**

```text
Window 1:
Token 1 → 500

Window 2:
Token 401 → 900

Window 3:
Token 801 → 1300

Here:
window = 500
step   = 400
overlap = 100
```

**The idea:** Every chunk has surrounding context from neighboring text.

**Useful when:** Information frequently crosses boundaries.

### Downside

More overlap means:

- More chunks
- More embeddings
- More storage
- More retrieval candidates

So don't blindly increase overlap.

## 11. Parent-child / hierarchical chunking ⭐⭐⭐

This is one of the most useful production techniques.

Instead of saying: "The chunk I retrieve must be the chunk I send to the LLM," we separate:

```text
Retrieval unit ≠ Generation context
```

**Example:**

```text
Parent
2000 tokens
       │
 ┌─────┼─────┐
 ↓     ↓     ↓
Child Child Child
 500   500   500
```

You embed/search the children:

```text
Query
 ↓
Search 500-token children
 ↓
Best child
```

But return the parent:

```text
Parent = 2000 tokens
```

So:

```text
Search small → Retrieve precise match → Return larger context
```

Databricks explicitly documents this small-to-big / parent-child retrieval pattern.

### Why is this powerful?

**Suppose:**

Parent: "Employee Health Insurance" contains:

- Child 1 → Eligibility
- Child 2 → Premium
- Child 3 → Deductible
- Child 4 → Claims

**User asks:** "What is the deductible?"

**Search:** Child 3

**Provide:** Parent section to the LLM

This gives:

- Precision during retrieval
- Context during generation
## 12. Sentence-window chunking

This is a variation of the previous idea.

You index a small unit, often a sentence:

```text
Sentence 17
```

But when retrieved, you provide surrounding sentences:

```text
Sentence 14
Sentence 15
Sentence 16
Sentence 17 ← matched
Sentence 18
Sentence 19
Sentence 20
```

So:

- **Index:** small
- **Retrieve:** small
- **Generate:** larger surrounding window

This is especially useful when the exact answer is contained in one sentence but surrounding sentences are needed to understand it.

## 13. Document-summary / contextual chunking

This one is slightly different.

You don't necessarily change the physical chunk boundaries. Instead, you add context to each chunk.

**Original problem:**

```text
Chunk:
The deductible is $500.

Problem: $500 for what?
```

**Solution:**

```text
Document: Employee Health Insurance Policy
Section: Annual Deductible
Chunk: The deductible is $500.

Or:

Document summary: Company employee health insurance policy.
Section: Annual Deductible.
Content: The deductible is $500.
```

Then embed:

```text
Company employee health insurance policy.
Annual Deductible.
The deductible is $500.
```

Databricks specifically recommends enriching chunks with document summaries, section headers, entities, topics and categories because this can improve retrieval and reranking.

## 14. Table-aware chunking ⭐

Tables are a special problem.

**Suppose the PDF contains:**

| Product | Price | Region |
|---------|-------|--------|
| A       | $100  | US     |
| B       | $200  | India  |
| C       | $150  | UK     |

**Bad chunking might produce:**

```text
Product Price Region
A 100
B 200
C 150
```

or worse lose the table structure entirely.

**Instead, preserve the table as a logical unit or convert it into structured text:**

```text
Product A:
Price = $100
Region = US

Product B:
Price = $200
Region = India

Product C:
Price = $150
Region = UK
```

This is not simply "another splitter"; it's document-aware parsing + chunking.

Databricks recommends high-quality parsing and preserving tables/structure because poor parsing directly affects retrieval quality.

## 15. Code-aware chunking

For code repositories:

**Don't do:**

```text
500 tokens
500 tokens
500 tokens
```

because you could get:

```python
def calculate_price(...):
    ...
```

split away from its body.

**Instead, use the structure:**

```text
Repository
 ↓
File
 ↓
Class
 ↓
Method
 ↓
Function
```

**Example chunk:**

```text
File: payment.py
Class: PaymentProcessor
Method: calculate_refund()

def calculate_refund(...):
    ...
```

This preserves the logical unit.

## 16. Hierarchical / tree-based chunking

This goes beyond simple parent-child.

**Imagine:**

```text
Document
│
├── Chapter 1
│   ├── Section 1.1
│   ├── Section 1.2
│   └── Section 1.3
│
├── Chapter 2
│   ├── Section 2.1
│   └── Section 2.2
│
└── Chapter 3
```

You can build a hierarchy:

```text
Document
   ↓
Chapter
   ↓
Section
   ↓
Paragraph
   ↓
Sentence
```

Retrieval can operate at different levels depending on the question.

**For example:**

> "What is the company's termination policy?"

Retrieve:

```text
Chapter 8
 ↓
Section 8.3
 ↓
Relevant paragraph
```

Databricks' guidance mentions hierarchical/tree-based approaches as advanced strategies, alongside parent-child retrieval.

## 17. Cluster/topic-based chunking

Another approach is:

```text
Document
 ↓
Sentences
 ↓
Embeddings
 ↓
Clustering
 ↓
Topic groups
```

**For example:**

- Cluster 1 → Authentication
- Cluster 2 → Authorization
- Cluster 3 → Billing
- Cluster 4 → Troubleshooting

This is conceptually related to semantic chunking but can operate over larger groups and use clustering/topic modeling rather than only neighboring sentence similarity.

It's useful for large, loosely structured documents, but it is more complex.

## 18. Query-aware / dynamic chunking

This is an advanced idea:

**Instead of having one permanent chunking strategy, you adapt retrieval context based on the query.**

**For example:**

**Query A:** "What is the refund amount?"

Use: small precise chunks

**Query B:** "Explain the entire refund policy."

Use: larger sections / parent chunks

```text
User Query
    ↓
Understand query
    ↓
Choose retrieval granularity
    ↓
Retrieve
```

This is powerful but significantly more complex than standard RAG.

## 19. A very important distinction: chunking vs metadata

Suppose you have a chunk:

```text
The deductible is $500.
```

You can improve it **without changing chunk boundaries** by adding metadata:

```text
Metadata:
document = employee_insurance.pdf
section = deductible
page = 12
topic = health insurance
```

Or enrich the actual embedding text:

```text
Employee Health Insurance
Section: Annual Deductible
The deductible is $500.
```

Databricks specifically recommends semantic metadata such as document summaries, section headers, entities and categories as part of retrieval-quality optimization.

**So don't think:** "Every RAG problem must be solved by changing chunk size."

Sometimes metadata/context enrichment is the better solution.

## 20. How all these strategies relate

Here's the hierarchy to understand:

```text
                    CHUNKING
                       │
       ┌───────────────┼────────────────┐
       │               │                │
    LENGTH          STRUCTURE        SEMANTIC
       │               │                │
       ├─ Fixed        ├─ Paragraph     ├─ Semantic
       ├─ Sliding      ├─ Markdown      ├─ Topic
       └─ Sentence     ├─ HTML          └─ LLM
                       ├─ Code
                       └─ Table

                       +
                       
                RETRIEVAL STRATEGY
                       │
                ┌──────┴──────┐
                ↓             ↓
             Parent        Sentence
             Child          Window
```
## 21. Which strategy should you actually use?

This is where people often make the mistake of asking: **"Which chunking strategy is the best?"**

**There isn't one.**

Databricks explicitly says the optimal chunk size and method depend on the use case and document characteristics, and recommends experimentation/evaluation.

### Recommended starting strategies by data type

| Your data | Starting strategy |
|-----------|-------------------|
| Simple text | Fixed/recursive |
| Normal documents | Recursive ⭐ |
| Markdown docs | Header/structure-aware |
| HTML | HTML structure-aware |
| Technical manuals | Recursive + metadata |
| PDFs | Parse → structure-aware/recursive |
| Long legal documents | Parent-child |
| Long articles | Semantic / recursive |
| FAQs | One Q&A = one chunk |
| Tables | Table-aware |
| Source code | Code-aware |
| Very high-value knowledge base | Semantic + parent-child |
| Complex heterogeneous corpus | Hybrid strategy |

Databricks' current retrieval-quality guide suggests experimenting with roughly 256, 512 and 1024-token chunk sizes as starting points, while emphasizing that smaller chunks improve localization and larger chunks preserve more context.

## 22. What I would implement in a Databricks RAG project

For a typical enterprise PDF/document RAG system, I wouldn't start with fancy semantic chunking.

I'd do:

```text
                PDF
                 ↓
          ai_parse_document
                 ↓
        Clean structured text
                 ↓
       Detect document structure
                 ↓
        ┌────────┴────────┐
        ↓                 ↓
    Headers           Tables
        ↓                 ↓
    Sections         Table chunks
        └────────┬────────┘
                 ↓
       Recursive chunking
          ~256–1024 tokens
                 ↓
       Add semantic metadata
                 ↓
       Parent-child mapping
                 ↓
             Embedding
                 ↓
        Databricks Vector Search
                 ↓
          Retrieve children
                 ↓
          Return parent/context
                 ↓
             Reranking
                 ↓
                LLM
```

That combines several techniques instead of treating them as mutually exclusive.

Databricks' own current guidance highlights semantic chunking, parent-child retrieval, document parsing, metadata enrichment and reranking as ways to improve retrieval quality.

## 23. The most important thing: evaluate, don't guess

Suppose you have multiple strategies to test:

```text
Strategy A → 256 tokens
Strategy B → 512 tokens
Strategy C → 1024 tokens
Strategy D → semantic
Strategy E → parent-child
```

Don't decide based on theory alone. **Create an evaluation set:**

```text
100 real user questions
        ↓
Run each strategy
        ↓
Measure
 ├── Retrieval recall
 ├── Context relevance
 ├── Answer correctness
 ├── Groundedness
 ├── Latency
 └── Cost
```

Then choose the winner.

Databricks explicitly recommends experimenting with chunk size and retrieval configuration and using evaluation to optimize RAG quality.

## Cheat Sheet: Quick Reference

| Strategy | Description |
|----------|-------------|
| **FIXED** | Split by size. |
| **RECURSIVE** | Split by the best available boundary, then smaller boundaries. |
| **STRUCTURE-AWARE** | Respect the document's natural structure. |
| **SEMANTIC** | Split when the meaning/topic changes. |
| **PARENT-CHILD** | Search small, return large. |
| **SENTENCE-WINDOW** | Search one sentence, give surrounding sentences. |
| **LLM-BASED** | Let an LLM decide meaningful boundaries. |
| **TABLE/CODE-AWARE** | Use the structure of the data type. |
| **METADATA ENRICHMENT** | Add document/section context to improve retrieval. |