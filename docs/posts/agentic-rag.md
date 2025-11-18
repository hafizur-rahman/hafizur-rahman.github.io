**Disclaimer: Copied from Udacity Agentic AI Nanodegree**

Refer to [RAG vs. Agentic RAG](https://mljourney.com/rag-vs-agentic-rag-a-comprehensive-comparison/)

## From Passive Lookup to Active Reasoning: Building Agentic RAG
Basic Retrieval-Augmented Generation (RAG) systems follow a simple pattern: accept a query, retrieve a few documents, and generate an answer. This works for straightforward questions but falls short in more complex scenarios. Agentic RAG enhances this by giving agents the ability to reflect, retry, and refine — transforming retrieval from a static tool into an active part of the agent’s reasoning loop.

### Beyond One-Shot Retrieval
Traditional RAG does not evaluate the usefulness of what it retrieves. It retrieves once and moves on. Agentic RAG introduces a feedback loop:

* The agent assesses whether the documents are relevant
* It identifies missing information
* It reformulates queries if needed
* It repeats retrieval and adjusts strategies before answering
This makes the process iterative and intelligent, not one-and-done.

### Case Study: Zillow Offers Failure
Search results on Zillow’s failed home-buying program, which ran from 2018 to 2021, reveal multiple contributing factors, including flawed algorithmic assumptions, unpredictable market behavior, and internal business pressures. (See resources below for more on these factors.)

A simple RAG agent might summarize that Zillow’s housing algorithm failed due to unpredictable markets. An agentic RAG agent, however, would:

* Notice gaps in the initial retrieval
* Reflect on what’s missing (e.g., algorithmic assumptions, feedback loops)
* Reformulate the query to focus on technical flaws and business strategy
* Retrieve better data and produce a layered explanation

The final answer becomes nuanced, grounded, and explanatory — not just a surface-level summary.

### Planning and Flexible Workflows
Agentic RAG agents don’t retrieve blindly. They plan:
* When to retrieve (or skip retrieval entirely)
* What to retrieve first (e.g., definitions, background, evidence)
* How to integrate retrieved context across multiple steps

Agents also inspect their results:
* Do they match query keywords?
* Do they answer “what” or “why” directly?
* Can each claim be backed by a specific source?
This self-monitoring guides whether to retry, pivot tools, or adapt strategy.

### Reflection and Retry Strategies
Agents can score retrieved passages or summarize what’s known vs. unknown before answering. These heuristics, often guided by prompt engineering or scoring models, help determine confidence and whether another retrieval round is necessary.

Frameworks may include metrics like faithfulness and relevancy, but even without them, the agent learns to reflect on its process — a major leap from static pipelines.

### Summary
Agentic RAG turns retrieval into part of the thinking loop:

* It reflects on what’s retrieved
* It adapts questions and strategy
* It decides whether to retry, switch tools, or proceed

This adaptive behavior allows agents to move beyond passive summarization into active problem-solving. The core pattern becomes: retrieve, reason, retry — a loop that defines more capable, intelligent systems.
