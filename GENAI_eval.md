# LLM Evaluation Frameworks

There are several established LLM evaluation frameworks, including frameworks specifically designed to evaluate LLM-as-a-Judge bias, reliability, and quality.

## Commonly Used Frameworks

### 1. MLflow Evaluate (Databricks)

Highly relevant for Databricks use cases.

Supports evaluation of:

- Correctness
- Groundedness
- Relevance
- Toxicity
- Retrieval quality
- Custom LLM judges

Example flow:

```text
Question
  -> Answer
  -> MLflow Evaluate
  -> Judge Model
  -> Score
```

### 2. Ragas

Popular for RAG applications.

Evaluates:

- Faithfulness
- Context precision
- Context recall
- Answer relevancy

Commonly used with:

- Vector search
- RAG pipelines
- LLM applications

### 3. DeepEval

Provides:

- Hallucination metrics
- Bias metrics
- Toxicity metrics
- Answer relevancy
- Custom judge evaluation

Designed specifically for LLM applications.

### 4. LangSmith

Provides:

- Tracing
- Human evaluation
- LLM-as-a-Judge workflows
- Dataset-based testing

Useful for agentic applications.

### 5. TruLens

Focuses heavily on:

- Groundedness
- Relevance
- Context quality
- RAG evaluation

## Evaluating Judge Bias

Researchers usually do not rely on a single framework alone.

They combine:

- Evaluation framework
- Human-labeled dataset
- Multiple judge models

to detect:

- Position bias
- Length bias
- Model favoritism
- Self-preference bias

## Databricks-Oriented Answer

If you are in a Databricks GenAI environment, a strong approach is:

Databricks provides MLflow Evaluate and Mosaic AI Evaluation capabilities for assessing correctness, groundedness, relevance, and safety. For broader industry evaluation, frameworks such as Ragas, DeepEval, TruLens, and LangSmith are commonly used. These frameworks help evaluate both application quality and the reliability of LLM judges.