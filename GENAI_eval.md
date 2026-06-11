# LLM Evaluation Frameworks

There are several established LLM evaluation frameworks, including frameworks specifically designed to evaluate LLM-as-a-Judge bias, reliability, and quality.

## Some of the most commonly used are:

### 1. MLflow Evaluate (Databricks)

Very relevant for Databricks.

Supports evaluation of:

Correctness
Groundedness
Relevance
Toxicity
Retrieval quality
Custom LLM judges

Example:

Question
   ↓
Answer
   ↓
MLflow Evaluate
   ↓
Judge Model
   ↓
Score
2. Ragas

Popular for RAG applications.

Evaluates:

Faithfulness
Context Precision
Context Recall
Answer Relevancy

Very commonly used with:

Vector Search
+
RAG
+
LLM
3. DeepEval

Provides:

Hallucination metrics
Bias metrics
Toxicity metrics
Answer relevancy
Custom judge evaluation

Designed specifically for LLM applications.

4. LangSmith

Provides:

Tracing
Human evaluation
LLM-as-a-Judge
Dataset-based testing

Useful for agentic applications.

5. TruLens

Focuses heavily on:

Groundedness
Relevance
Context quality
RAG evaluation
Specifically for Judge Bias

Researchers usually don't rely on a single framework alone.

They combine:

Evaluation Framework
        +
Human-Labeled Dataset
        +
Multiple Judge Models

to detect:

Position bias
Length bias
Model favoritism
Self-preference bias
Databricks-Oriented Answer

If you're in a Databricks GenAI session, a good option is:

Yes. Databricks provides MLflow Evaluate and Mosaic AI Evaluation capabilities for assessing correctness, groundedness, relevance, and safety. For broader industry evaluation, frameworks such as Ragas, DeepEval, TruLens, and LangSmith are commonly used. These frameworks help evaluate both application quality and the reliability of LLM judges themselves