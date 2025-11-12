---
layout: post
title: Agentic AI with Agno
---
### Agentic AI with Agno

I recently came to know about Google ADK (Agent Development Kit). Even though most of the frameworks aim for targetting any model, popular models are throroughly tested. Someone like me who want to try locally deployed model through Ollama or similar tools, the examples needs lots of tweaking as models behave differently. And debugging agentic code and LLM is not that interesting task specially you don't have NVIDIA GPU.

Today I came to know about another agentic framework named Agno. So far, it seems to work for open weight models properly!

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```

### Code
```
from agno.team import Team
from agno.agent import Agent
from agno.models.ollama import Ollama
from agno.tools.duckduckgo import DuckDuckGoTools

model=Ollama(id="aliafshar/gemma3-it-qat-tools:27b")

researcher = Agent(
    name="Researcher",
    role="Act as a Research Agent using search tool and present findings.",
    instructions=[
        "You are a specialized research agent."
        "Your only job is to find 5-10 articles on the given topic and present the findings with citations."
    ],
    tools=[DuckDuckGoTools()],
    markdown=True,
)

print("✅ Researcher agent created.")

writer = Agent(
    name="Writer",
    role="Expert at writing clear, engaging content.",
    instructions=[
        "Read the provided research findings.",
        "Create a concise summary as a bulleted list with 3-5 key points."
    ],
    markdown=True,
)

print("✅ Writer agent created.")

team = Team(
    name="Content Team",
    model=model,
    instructions=[
        "You are a team of researchers and writers that work together to create high-quality content.",
    ],
    members=[ researcher, writer],
    show_members_responses=True,
    debug_mode=True,
)

print("✅ Team created.")

team.print_response("What are the latest advancements in quantum computing and what do they mean for AI?")
```

### Output
```
```