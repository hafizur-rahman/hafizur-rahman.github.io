## Constructing agentic systems using Python
Building dependable systems usually involves several major components, with a focus on crafting the core logic.

* **LLM Models - The Engine** : Central to any agent is its access to Large Language Models. These are the engines providing the core intelligence. A flexible system design might allow for different LLMs to be connected or swapped, making sure of adaptability as new models emerge.
* **Agent Logic & Prompting** - Custom Implementation: An important layer is the 'agent logic' itself. This is where Python code would define how an agent behaves. This includes crafting effective prompts to communicate with LLMs and implementing the distinct capabilities envisioned for different types of agents.
* **Workflow Orchestration** - Connecting Agents: With individual agents defined, a system needs to manage how they work together. 'Workflow orchestration' logic, also implemented in Python, would handle the sequence of operations, the flow of information between agents, and the overall execution of multi-agent processes. While various tools can provide these pieces, understanding how to conceptually structure these components in Python offers deep insight into their functionality.

### Agentic Workflow Patterns

#### Main Articles
* [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents)
* [Design Patterns in Agentic AI Workflow](https://pub.towardsai.net/5-design-patterns-in-agentic-ai-workflow-c972c83f77e4)
* [Agentic Workflow & Patterns](https://javaaidev.com/docs/agentic-patterns/intro)

#### Detailed Articles
* Prompt chaining
    * [Datacamp Article](https://www.datacamp.com/tutorial/prompt-chaining-llm)
    * [IBM Article](https://www.ibm.com/think/topics/prompt-chaining)
    * [GeekForGeeks Article](https://www.geeksforgeeks.org/artificial-intelligence/prompt-chaining/)
* Routing
    * [Patronous AI Article](https://www.patronus.ai/ai-agent-development/ai-agent-routing)
* Parallelization
* Evaluator-Optimizer
* Orchestrator-Workers