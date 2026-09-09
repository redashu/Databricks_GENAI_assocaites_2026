# 20-Lab Implementation Roadmap

## Track A — Foundation

| Lab | Topic | Industry outcome | Certification |
| --- | --- | --- | --- |
| 01 | Databricks GenAI environment | Understand platform | Design Applications |
| 02 | AI Playground + LLMs | Model selection | Design Applications |
| 03 | Prompt engineering | Reliable outputs | Design Applications |
| 04 | Structured outputs | Ticket/document extraction | Design Applications |
| 05 | LLM application in Python | First GenAI app | Application Development |

## Track B — RAG

| Lab | Topic | Industry outcome | Certification |
| --- | --- | --- | --- |
| 06 | Document ingestion | Enterprise KB | Data Preparation |
| 07 | Chunking + embeddings | Quality retrieval | Data Preparation |
| 08 | Databricks AI Search | Enterprise RAG | Application Development |
| 09 | RAG application | IT knowledge assistant | Application Development |
| 10 | RAG evaluation | Measure quality | Evaluation |

> The current AI Search implementation is particularly relevant because Databricks has renamed Vector Search to AI Search, and the current docs show AI Search indexes providing real-time similarity search over Delta tables.

## Track C — MLflow 3

| Lab | Topic | Outcome |
| --- | --- | --- |
| 11 | MLflow 3 tracing | Debug GenAI |
| 12 | LLM judges/scorers | Automated evaluation |
| 13 | Human feedback | Expert evaluation |
| 14 | Prompt/app versioning | Iterative improvement |

> This is important: we will use MLflow 3, not build around the older Agent Evaluation APIs. Databricks has explicitly integrated Agent Evaluation into `mlflow.genai`, including evaluation, human labeling, evaluation datasets, and enhanced tracing.

## Track D — Agents

| Lab | Topic | Outcome |
| --- | --- | --- |
| 15 | Tool-calling agent | IT support agent |
| 16 | Agent Framework | Production agent |
| 17 | LangGraph comparison | Framework understanding |
| 18 | Multi-agent | Enterprise supervisor |
| 19 | MCP | External/internal tools |
| 20 | Deployment + governance | Production architecture |

Current Databricks agent documentation supports custom agents, RAG applications, tool-calling agents, and multi-agent systems.

MCP is also now an explicit part of the current platform. Databricks' August 2026 documentation describes managed, external, and custom MCP servers, with Databricks recommending Apps or Model Serving for deployment.

---

## One application, progressively evolved

We won't treat these as 20 isolated notebooks. We will progressively evolve one application.

```text
LAB 01  Databricks foundation
   ↓
LAB 02  LLM
   ↓
LAB 03  Prompt
   ↓
LAB 04  Structured output
   ↓
LAB 05  Python application
   ↓
LAB 06-10  RAG
   ↓
LAB 11-14 MLflow 3
   ↓
LAB 15  Agent
   ↓
LAB 16  Agent Framework
   ↓
LAB 17  LangGraph
   ↓
LAB 18  Multi-agent
   ↓
LAB 19  MCP
   ↓
LAB 20  Production architecture
```

By the end, you won't just know:

> "I know RAG."

You will be able to explain:

> "Here is how I would design, implement, evaluate, govern, deploy, and monitor an enterprise GenAI application on Databricks."

That is much closer to what the certification is testing.

---

# LAB 01 — Databricks GenAI Foundation

We're ready to start.

## Objective

Before writing GenAI code, we will understand the Databricks architecture we are going to use:

```text
                  Databricks
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   Unity Catalog   Compute      AI/ML
        │                           │
        │              ┌────────────┼────────────┐
        │              ▼            ▼            ▼
        │         AI Playground  AI Search   MLflow 3
        │
        ▼
   Delta Tables
        │
        ▼
   Enterprise Data
```

We will specifically inspect:

- Workspace
- Catalog
- Schema
- Volume
- Notebook
- Serverless compute
- AI Playground
- Model serving
- MLflow
- AI Search

## Step 1 — Open your Free Edition

Use the official Databricks Free Edition environment.

### Databricks Free Edition

Do not create a paid cloud workspace for this course.

Free Edition is specifically intended for learning and experimentation, although it has fair-use and feature limitations.

## Step 2 — First exercise: explore the workspace

Once you're inside Databricks, do not create anything yet.

Look at the left navigation.

We want to identify whether you have access to:

- Workspace
- Catalog
- Compute
- SQL
- AI/ML
  - Playground
  - Experiments
  - Models
- Serving
- Apps

Your UI may differ slightly from screenshots and tutorials because Databricks changes the navigation and product naming frequently.

## Step 3 — AI Playground

Before coding, we'll use the Playground.

Databricks' current official tutorial uses AI Playground to:

- query LLMs
- compare models
- prototype tool-calling agents
- export agents to Apps/notebooks
- optionally prototype RAG

So our first experiment will be deliberately simple.

Ask:

> Explain Retrieval Augmented Generation to a senior data engineer who understands Spark and Databricks but is new to GenAI.

Then we'll compare another model.

The goal is not the answer. The goal is to observe:

```text
Prompt
   ↓
Model
   ↓
Response
```

and start developing the mental model:

An LLM application is an engineered system around a model, not simply a model call.

## Step 4 — Our first certification question

Think about this:

> A company wants an AI assistant that answers questions using internal company documents. The LLM itself does not contain those documents. What architectural capability should you introduce?

Do not answer from memory yet.

We'll reason through it when we implement RAG.

The answer will eventually become:

```text
Internal Data
     ↓
Retrieval
     ↓
Context
     ↓
LLM
     ↓
Answer
```

## Step 5 — Our project data

For the entire course, we'll create a small synthetic enterprise dataset.

We'll eventually have:

```text
enterprise_genai/
│
├── knowledge/
│   ├── IT_Policy.pdf
│   ├── VPN_Guide.pdf
│   ├── Password_Policy.pdf
│   ├── Incident_Runbook.pdf
│   └── Cloud_Operations.pdf
│
├── tickets/
│   └── tickets.csv
│
├── incidents/
│   └── incidents.csv
│
└── employees/
    └── employees.csv
```

Nothing confidential. Everything synthetic.

That means you can later publish the project as a portfolio or GitHub project without exposing company data.

Step 6 — The final application we're building

Our eventual application will look like:

                 User
                  │
                  ▼
           Enterprise AI
             Assistant
                  │
          ┌───────┴────────┐
          │                │
          ▼                ▼
      Knowledge          IT Ops
       Agent              Agent
          │                │
          ▼                ▼
      AI Search        Structured Data
                           │
                           ▼
                         Genie
          │                │
          └───────┬────────┘
                  ▼
                 LLM
                  │
                  ▼
             Guardrails
                  │
                  ▼
              Response
                  │
                  ▼
              MLflow 3
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      Trace     Eval      Feedback

Databricks' current agent architecture supports this general direction, including custom agents, retrieval, tools and multi-agent applications.

One important rule for our implementation

Whenever I give you code, we'll understand the code before moving forward.

For example, if we eventually write:

mlflow.genai.evaluate(...)

I won't just say:

"Run this."

We'll understand:

What is MLflow?
        ↓
What is GenAI evaluation?
        ↓
What is a scorer?
        ↓
What is an evaluator?
        ↓
What data goes into evaluation?
        ↓
What comes out?
        ↓
Why does Databricks use it?
        ↓
What could the certification ask?