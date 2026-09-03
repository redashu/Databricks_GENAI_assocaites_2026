# LLM Agent Evaluation

Agent Evaluation is now integrated into MLflow 3. The evaluation APIs are under `mlflow.genai`, and MLflow provides built-in judges, custom LLM judges, code-based scorers, evaluation datasets, tracing, and production monitoring.

## 1. Why do we need an LLM Judge?

Traditional ML is relatively easy to evaluate.

For example:

```text
Actual = 1
Expected = 1
→ Correct

Or:

Predicted price = $150
Actual price = $155
→ Calculate RMSE / MAE
```

But consider a GenAI application:

**User:** "Explain what RAG is."

**Agent:** "RAG combines retrieval with generation. It retrieves relevant external information and gives it to an LLM so the LLM can generate a grounded response."

**How do you programmatically determine:**

- Is this answer correct?
- Is it relevant?
- Is it well written?
- Is it grounded?
- Is it safe?

There isn't always a simple formula.

**That's where LLM-as-a-Judge comes in.**

## 2. What is an LLM Judge?

Think of it as:

```text
          Your Agent
              │
              ↓
          Answer
              │
              ↓
        ┌───────────┐
        │ LLM Judge │
        └─────┬─────┘
              │
              ↓
         Evaluation
```

The judge is another LLM whose job is not to answer the user's question, but to evaluate the answer produced by your application.

**Example:**

```text
User:
  "What is RAG?"

Agent:
  "RAG retrieves relevant documents and gives them
   to an LLM to generate a grounded answer."
        
      ↓
    
  Judge LLM

Question:
  What is RAG?

Answer:
  ...

Evaluate correctness from 1-5.

      ↓

Score = 5
Reason = "The answer accurately describes
          retrieval and generation."
```

Databricks defines LLM judges as a type of MLflow scorer that uses an LLM to assess application quality.

## 3. Judge vs Scorer — VERY important for the exam

These two terms can be confusing.

### Judge

The judge is the evaluation logic using an LLM.

```text
Judge
  ↓
"Is this answer relevant?"
```

### Scorer

A scorer is the broader MLflow mechanism that evaluates a trace and produces feedback.

```text
Trace
  ↓
Scorer
  ↓
Feedback / Score
```

A scorer may use:

- **LLM Judge** OR
- **Python code**

Databricks describes it nicely: **Judges are a type of scorer that use LLMs for evaluation.**

**So remember:**

```text
            SCORER
               │
      ┌────────┴────────┐
      ↓                 ↓
  LLM Judge         Python Code
```

## 4. What exactly does the Judge evaluate?

There isn't one universal "quality score." You define evaluation dimensions.

**For example:**

```text
                Agent
                  │
                  ↓
          ┌──────────────┐
          │    Judges    │
          └──────┬───────┘
                 │
     ┌───────────┼──────────────┐
     ↓           ↓              ↓
Correctness  Relevance      Safety
     ↓           ↓              ↓
    0-1         0-1            PASS
```

Databricks provides built-in judges for common dimensions including:

- Correctness
- Relevance
- Safety
- Groundedness
- Retrieval relevance
- and others.

## 5. Example: RAG evaluation

This is particularly important for your RAG studies.

**Suppose:**

**User:** "What is the company's refund policy?"

**Retriever finds:**

- Document A: Refunds are allowed within 30 days.
- Document B: Premium membership costs $100.
- Document C: Contact customer support for returns.

**Agent answers:** "Customers can request a refund within 30 days."

Now we can evaluate different things.

### Judge 1 — Retrieval relevance

Did the retrieved documents actually relate to the question?

```text
Retrieval Relevance = 0.95
```

### Judge 2 — Groundedness

Is the answer supported by the retrieved context?

```text
Groundedness = 1.0
```

### Judge 3 — Correctness

Is the answer factually correct?

```text
Correctness = 1.0
```

### Judge 4 — Relevance

Did the answer actually answer the user's question?

```text
Relevance = 1.0
```

**So you can diagnose where the problem occurred.**

## 6. A VERY useful mental model

For RAG:

```text
             USER QUESTION
                   │
                   ↓
              RETRIEVER
                   │
                   ↓
          Retrieved Context
                   │
                   ↓
                 LLM
                   │
                   ↓
              Final Answer
                   │
         ┌─────────┼─────────┐
         ↓         ↓         ↓
    Retrieval Grounded Correct
    Relevance  ness   ness
```

**Suppose:**

```text
Retrieval relevance = 30%
Groundedness        = 90%
Correctness         = 40%
```

You shouldn't immediately blame the LLM. **The retrieval system may be the problem.**

## 7. Agent Evaluation

An agent is more complicated than a simple RAG pipeline.

**Imagine your BA/DA Supervisor:**

```text
User
  ↓
Supervisor
  ↓
  ┌─────────┐
  ↓         ↓
  BA        DA
  ↓         ↓
  MCP       MCP
  ↓         ↓
  Tool      Tool
  ↓         ↓
Answer    Answer
```

The final answer could be wrong because:

- Supervisor selected the wrong agent
- Correct agent selected the wrong tool
- Tool received wrong parameters
- Tool returned wrong data
- RAG retrieved wrong documents
- LLM misunderstood the data
- Final answer was poorly formatted

**This is why tracing becomes extremely important for agent evaluation.**

MLflow scorers can evaluate not only inputs/outputs but also the complete execution trace.

## 8. Evaluate your BA/DA Supervisor

Suppose you have:

```python
test_data = [
    {
        "inputs": {
            "query": "What is the average price in Mission?"
        },
        "expectations": {
            "expected_agent": "BA Agent"
        }
    },
    {
        "inputs": {
            "query": "Is $350/night considered luxury?"
        },
        "expectations": {
            "expected_agent": "DA Agent"
        }
    }
]
```

Your agent executes:

```python
result = await Runner.run(supervisor, query)
```

MLflow captures the trace.

You can then create a code-based scorer:

```python
from mlflow.genai.scorers import scorer

@scorer
def routing_accuracy(
    *,
    inputs,
    outputs,
    expectations,
    trace
):
    expected = expectations["expected_agent"]
    actual = trace["last_agent"]
    return actual == expected
```

**Conceptually:**

```text
Expected Agent
      │
      │
      ├─────────────┐
      │             │
      ↓             ↓
    BA Agent      DA Agent
                    ↑
                    │
               Actual Agent

          Compare them
                ↓
             PASS/FAIL
```

This is a code-based scorer, not an LLM judge.

Databricks specifically recommends code-based scorers for deterministic/programmatic metrics such as exact matching and business rules.

## 9. When should I use LLM Judge vs Python scorer?

This is probably the most important exam concept.

### Use Python/code scorer when the answer can be determined deterministically.

**Example:**

```text
Expected agent = BA
Actual agent   = BA
→ PASS

Or:

Expected status = "SUCCESS"
Actual status   = "SUCCESS"
→ PASS

Or:

Response contains required field "customer_id"
→ TRUE
```

Use:

```python
@scorer
def exact_match(...):
    ...
```

### Use LLM Judge when human-like interpretation is required.

**Example:** "Is the answer relevant to the user's question?"

There may be many valid answers. You can't easily write:

```python
if answer == expected_answer:
```

Instead:

```text
Answer
  ↓
LLM Judge
  ↓
Relevant?
  ↓
Yes
```

Databricks' built-in judges are intended for these common semantic quality dimensions.

## 10. Built-in LLM Judges

You don't have to build the judge yourself.

Databricks provides built-in judges such as:

- `RelevanceToQuery`
- `RetrievalRelevance`
- `Safety`
- `Correctness`
- `Groundedness`

The exact available judges continue to evolve, so for the current exam/product behavior, check the current MLflow 3 documentation.

**Conceptually:**

```python
from mlflow.genai.scorers import (
    Correctness,
    Safety,
)

scorers = [
    Correctness(),
    Safety()
]
```

Then:

```python
mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=my_agent,
    scorers=scorers
)
```

The evaluation harness runs the application against the evaluation dataset and applies the scorers to the resulting traces.

## 11. Custom LLM Judge ⭐

Suppose your company has a special rule:

> "For banking customer support, the agent must never recommend a financial product unless the customer explicitly asks for one."

That's not necessarily a standard built-in metric.

You can create a custom judge.

MLflow 3 provides `make_judge()` for this.

**Conceptually:**

```python
from mlflow.genai.judges import make_judge

financial_policy_judge = make_judge(
    name="financial_policy",
    instructions="""
    Evaluate whether the agent response follows this policy:

    The agent must not recommend financial products
    unless the user explicitly requests a recommendation.

    Return PASS if the policy is followed.
    Return FAIL otherwise.

    User query:
    {{ inputs }}

    Agent response:
    {{ outputs }}

    Expected behavior:
    {{ expectations }}
    """
)
```

**Now:**

```text
User Query
    +
Agent Response
    +
Company Policy
       ↓
  Custom LLM Judge
       ↓
   PASS / FAIL
```

Custom judges can access inputs, outputs, expectations, and even the complete trace.

## 12. Why expectations are important

This is another exam-worthy concept.

**Suppose:**

```text
inputs:
  "What is RAG?"

outputs:
  "RAG retrieves relevant information..."

expectations:
  "Answer should explain retrieval + generation."
```

The judge gets:

```text
                Judge
                  │
      ┌───────────┼───────────┐
      ↓           ↓           ↓
    Input       Output    Expectation
      │           │           │
      └───────────┼───────────┘
                  ↓
                Score
```

expectations can represent ground truth or expected behavior, not necessarily an exact expected string. MLflow supports human feedback/expectations as part of its evaluation data model.

## 13. Does the Judge have to use the same LLM as my Agent?

**No.**

This is an important conceptual point.

You could have:

```text
Agent
  ↓
GPT / Claude / Llama

and:

Judge
  ↓
Another LLM
```

The judge is evaluating the agent.

Databricks provides hosted judges, but code-based scorers can also invoke your own LLM when you need that control.

**So:**

```text
             Agent LLM
                 │
                 ↓
              Answer
                 │
                 ↓
             Judge LLM
                 │
                 ↓
              Score
```

**The two models have different jobs.**

## 14. Can an LLM Judge evaluate the agent's reasoning?

Be careful with the terminology.

You don't need to expose or evaluate hidden chain-of-thought.

Instead, evaluate the observable execution trace:

```text
User input
     ↓
Supervisor decision
     ↓
Tool calls
     ↓
Tool arguments
     ↓
Retrieved context
     ↓
Final output
```

MLflow traces can provide the execution information that scorers inspect.

For example, you can ask:

- "Did the agent use the appropriate tool?"
- "Was the retrieved context relevant?"

rather than trying to inspect private reasoning.

## 15. Evaluation dataset

Now put everything together. You create an evaluation dataset.

**For example:**

| Query | Expected behavior |
|-------|-------------------|
| Average price in Mission? | BA |
| Is $350 luxury? | DA |
| Compare Mission vs Noe Valley | BA |
| What is refund policy? | RAG |
| Give me someone's SSN | Reject |

This becomes your golden/evaluation dataset.

**Then:**

```text
            Evaluation Dataset
                    ↓
              Run Agent
                    ↓
                 Traces
                    ↓
          ┌─────────┼─────────┐
          ↓         ↓         ↓
       Judge 1   Judge 2   Code Scorer
       Correct   Safety    Routing
          ↓         ↓         ↓
         90%       99%       95%
```

MLflow's evaluation harness is specifically designed to automate this process instead of manually checking responses one by one.

## 16. Benchmarking different versions

This is where evaluation becomes useful for development.

**Suppose:**

```text
Agent V1
Prompt V1
Chunk size = 512

Results:
  Correctness = 82%
  Groundedness = 86%
  Routing = 91%
```

You change the prompt:

```text
Agent V2
Prompt V2
Chunk size = 512

Run the same evaluation dataset:
  Correctness = 90%
  Groundedness = 92%
  Routing = 96%
```

**Now you have evidence:** V2 > V1

MLflow supports evaluation runs specifically for comparing iterations and detecting regressions.

## 17. LLM Judge isn't automatically "truth"

This is a VERY important interview/exam concept.

An LLM judge is itself an LLM. Therefore:

Agent can make mistakes
        AND
Judge can make mistakes

## 17. LLM Judge isn't automatically "truth"

This is a VERY important interview/exam concept.

An LLM judge is itself an LLM. Therefore:

```text
Agent can make mistakes
        AND
Judge can make mistakes

For example:

Agent → 90% correct
Judge  → incorrectly says 100%
```

Therefore, you should validate judges against human expert judgment.

Databricks explicitly supports human feedback and recommends using human feedback to align automated evaluation with expert judgment.

**Think:**

```text
Human Experts
     ↓
Validate Judge
     ↓
Automated Judge
     ↓
Scale evaluation
```

## 18. Human evaluation + LLM evaluation

A mature enterprise system can look like:

```text
             Production Agent
                   │
                   ↓
                Traces
                   │
         ┌─────────┴─────────┐
         ↓                   ↓
    LLM Judges          Human Review
         │                   │
         ↓                   ↓
    Automated             Expert
     Feedback             Feedback
         │                   │
         └─────────┬─────────┘
                   ↓
            Evaluation Dataset
                   ↓
              Improve Agent
```

MLflow stores human feedback as assessments attached to traces, and that feedback can help create evaluation data and improve automated evaluation.

## 19. What about benchmarking against other frameworks?

**Yes.**

MLflow 3 can integrate third-party scorers from external evaluation frameworks. This lets you combine specialized metrics with MLflow's built-in judges and code-based scorers. Databricks currently documents integrations with third-party evaluation frameworks.

**Conceptually:**

```text
               MLflow Evaluation
                     │
      ┌──────────────┼──────────────┐
      ↓              ↓              ↓
Databricks Judges  Code Scorers  3rd-party
                                  Scorers
```

This is useful when you need specialized metrics such as:

- BLEU / ROUGE
- jailbreak detection
- agent plan quality
- framework-specific metrics

rather than only Databricks' built-in dimensions.

## 20. One small end-to-end demo

Here's the simplified pattern to remember for an exam:

```python
import mlflow
from mlflow.genai.scorers import Correctness, Safety, RelevanceToQuery

eval_data = [
    {
        "inputs": {
            "query": "What is RAG?"
        },
        "expectations": {
            "expected_response": (
                "RAG retrieves relevant external information "
                "and provides it to an LLM for generation."
            )
        }
    },
    {
        "inputs": {
            "query": "Explain embeddings."
        },
        "expectations": {
            "expected_response": (
                "Embeddings represent information as numerical vectors."
            )
        }
    }
]

def predict_fn(query):
    # Call your actual Databricks agent here
    return my_agent(query)

results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=predict_fn,
    scorers=[
        Correctness(),
        Safety(),
        RelevanceToQuery()
    ]
)
```

**Conceptually:**

```text
            eval_data
                │
                ↓
           predict_fn
                │
                ↓
              Agent
                │
                ↓
              Trace
                │
      ┌─────────┼─────────┐
      ↓         ↓         ↓
 Correctness  Safety   Relevance
    Judge      Judge      Judge
      ↓         ↓         ↓
     0.92      1.0       0.95
```

The exact scorer names/API can evolve with MLflow 3 releases, so for an exam, focus first on the concept and roles, then the current API syntax. Databricks' current documentation shows `mlflow.genai.evaluate()` as the evaluation harness.

## 21. Exam-focused cheat sheet ⭐

If you're preparing for a Databricks GenAI exam, memorize these key concepts:

| Term | Definition |
|------|------------|
| **LLM-as-a-Judge** | An LLM evaluates the output/behavior of another GenAI application against defined criteria. |
| **Built-in Judge** | Predefined Databricks/MLflow judge for common dimensions such as correctness, relevance, safety, groundedness and retrieval relevance. |
| **Custom LLM Judge** | Use `make_judge()` when your organization has its own natural-language evaluation criteria. |
| **Code-based Scorer** | Use Python when evaluation can be deterministic/programmatic — exact match, routing accuracy, required fields, latency, etc. |
| **Scorer** | General MLflow mechanism that evaluates a trace and produces feedback. A scorer can use an LLM judge or deterministic code. |
| **Evaluation Dataset** | Representative test cases used to systematically evaluate the agent and compare versions. |
| **Expectations** | Ground truth/desired behavior used when the evaluation requires a reference. |
| **Trace** | Execution record containing the application's interaction and intermediate operations, which enables agent-level evaluation. |
| **Human Feedback** | Expert/user assessment used to validate and improve automated evaluation. |
| **Production Monitoring** | The same scorers can be applied to production traces to continuously monitor quality. |

⭐ The one diagram I'd memorize
                         USER
                           │
                           ▼
                    GENAI AGENT
                           │
                           ▼
                        TRACE
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
        LLM JUDGE     CODE SCORER    HUMAN FEEDBACK
             │             │             │
       ┌─────┼─────┐       │             │
       ↓     ↓     ↓       ↓             ↓
   Correct  Safe  Relevant Exact       Expert
   Ground   RAG   Quality  Match       Review
       │     │     │       │             │
       └─────┴─────┴───────┴─────────────┘
                           │
                           ▼
                    MLflow Evaluation
                           │
                           ▼
                 Compare Agent Versions
                           │
                           ▼
                  Production Monitoring

The exam trick: If the question says "evaluate semantic quality such as relevance/correctness/groundedness" → think LLM Judge. If it says "exact match, deterministic business rule, format validation, routing decision" → think code-based scorer. If it says "custom natural-language business criteria" → think custom LLM judge / make_judge()