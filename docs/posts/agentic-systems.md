## Constructing agentic systems using Python
Building dependable systems usually involves several major components, with a focus on crafting the core logic.

* **LLM Models** - The Engine: Central to any agent is its access to Large Language Models. These are the engines providing the core intelligence. A flexible system design might allow for different LLMs to be connected or swapped, making sure of adaptability as new models emerge.
* **Agent Logic & Prompting** - Custom Implementation: An important layer is the 'agent logic' itself. This is where Python code would define how an agent behaves. This includes crafting effective prompts to communicate with LLMs and implementing the distinct capabilities envisioned for different types of agents.
* **Workflow Orchestration** - Connecting Agents: With individual agents defined, a system needs to manage how they work together. 'Workflow orchestration' logic, also implemented in Python, would handle the sequence of operations, the flow of information between agents, and the overall execution of multi-agent processes. While various tools can provide these pieces, understanding how to conceptually structure these components in Python offers deep insight into their functionality.

### Agentic Workflow Patterns
* Prompt chaining
* Routing
* Parallelization
* Evaluator-Optimizer
* Orchestrator-Workers