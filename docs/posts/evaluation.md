References
* [ML Journey Article](https://mljourney.com/how-can-llamaindex-help-to-evaluate-results/)
* [LangChain Article](https://docs.langchain.com/langsmith/evaluate-rag-tutorial)

![Alt text](https://mintcdn.com/langchain-5e9cc07a/Fr2lazPB4XVeEA7l/langsmith/images/rag-eval-overview.png?w=840&fit=max&auto=format&n=Fr2lazPB4XVeEA7l&q=85&s=8656d8f8af7ffa7f2684376cf2f70874)

Evaluators
One way to think about different types of RAG evaluators is as a tuple of what is being evaluated X what its being evaluated against:

| Category | Goal | Mode | Evaluator |
| ------------- |:-------------:| -------------:| -------------:|
| Correctness: Response vs reference answer | Measure “how similar/correct is the RAG chain answer, relative to a ground-truth answer” | Requires a ground truth (reference) answer supplied through a dataset | Use LLM-as-judge to assess answer correctness. |
| Relevance: Response vs input | Measure “how well does the generated response address the initial user input” | Does not require reference answer, because it will compare the answer to the input question | Use LLM-as-judge to assess answer relevance, helpfulness, etc.|
| Groundedness: Response vs retrieved docs | Measure “to what extent does the generated response agree with the retrieved context” | Does not require reference answer, because it will compare the answer to the retrieved context | Use LLM-as-judge to assess faithfulness, hallucinations, etc. |
| Retrieval relevance: Retrieved docs vs input | Measure “how relevant are my retrieved results for this query” | Does not require reference answer, because it will compare the question to the retrieved context | Use LLM-as-judge to assess relevance |
