---
layout: post
title: Agent Types
---

![Agent types](assets/img/agent-types.jpg)

Not all agents are created equal. They exist on a spectrum of complexity and autonomy, largely defined by how they interact with their LLM core.

* **Direct Prompting**: The simplest form, sending a prompt directly to the LLM.
* **Augmented Prompting**: Adds a system message to define a persona.
* **Dynamic Context Augmentation**: Adapts during an interaction by using memory and tools to update context.
* **Autonomous Agent**: The most advanced type, leveraging the LLM for complex planning and execution.

### Import
```
from openai import OpenAI
import numpy as np
import pandas as pd
import re
import csv
import uuid
from datetime import datetime
```

### DirectPromptAgent
```
class DirectPromptAgent:
    """Directly sends the user prompt to the LLM and returns the response text."""

    def __init__(self, openai_api_key: str):
        self.openai_api_key = openai_api_key
        self.api_base = "https://openai.vocareum.com/v1"

    def respond(self, prompt: str) -> str:
        client = OpenAI(base_url = self.api_base, api_key=self.openai_api_key)
        response = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "user", "content": prompt}
            ],
            temperature=0
        )
        return response.choices[0].message.content
```
### AugmentedPromptAgent
```
class AugmentedPromptAgent:
    def __init__(self, openai_api_key: str, persona: str, base_url: str):
        self.persona = persona
        self.openai_api_key = openai_api_key
        self.api_base = base_url

    def respond(self, input_text: str) -> str:
        client = OpenAI(base_url = self.api_base, api_key=self.openai_api_key)
        response = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": f"You are {self.persona}. Forget all previous context."},
                {"role": "user", "content": input_text}
            ],
            temperature=0
        )
        response_content = response.choices[0].message.content
        
        # Add explanatory comments about knowledge source and persona impact
        comments = f"\n\n[Comment] This answer used the LLM's general world knowledge (no external knowledge)."
        comments += f"\n[Comment] The persona '{self.persona}' was enforced in the system prompt to shape the response style and tone."
        
        return response_content + comments

class KnowledgeAugmentedPromptAgent:
    def __init__(self, openai_api_key: str, persona: str, knowledge: str, base_url: str):
        self.persona = persona
        self.knowledge = knowledge
        self.openai_api_key = openai_api_key
        self.api_base = base_url

    def respond(self, input_text: str) -> str:
        client = OpenAI(base_url = self.api_base, api_key=self.openai_api_key)
        response = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {
                    "role": "system",
                    "content": (
                        f"You are {self.persona} knowledge-based assistant. Forget all previous context. "
                        f"Use only the following knowledge to answer, do not use your own knowledge: {self.knowledge} "
                        f"Answer the prompt based on this knowledge, not your own."
                    ),
                },
                {"role": "user", "content": input_text},
            ],
            temperature=0,
        )
        return response.choices[0].message.content

# RAGKnowledgePromptAgent class definition
class RAGKnowledgePromptAgent:
    """
    An agent that uses Retrieval-Augmented Generation (RAG) to find knowledge from a large corpus
    and leverages embeddings to respond to prompts based solely on retrieved information.
    """

    def __init__(self, openai_api_key, persona, base_url, chunk_size=2000, chunk_overlap=100):
        """
        Initializes the RAGKnowledgePromptAgent with API credentials and configuration settings.

        Parameters:
        openai_api_key (str): API key for accessing OpenAI.
        persona (str): Persona description for the agent.
        base_url (str): Base URL for the OpenAI-compatible API endpoint.
        chunk_size (int): The size of text chunks for embedding. Defaults to 2000.
        chunk_overlap (int): Overlap between consecutive chunks. Defaults to 100.
        """
        self.persona = persona
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.openai_api_key = openai_api_key
        self.api_base = base_url
        self.unique_filename = f"{datetime.now().strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:8]}.csv"

    def get_embedding(self, text):
        """
        Fetches the embedding vector for given text using OpenAI's embedding API.

        Parameters:
        text (str): Text to embed.

        Returns:
        list: The embedding vector.
        """
        client = OpenAI(base_url = self.api_base, api_key=self.openai_api_key)
        response = client.embeddings.create(
            model="text-embedding-3-large",
            input=text,
            encoding_format="float"
        )
        return response.data[0].embedding

    def calculate_similarity(self, vector_one, vector_two):
        """
        Calculates cosine similarity between two vectors.

        Parameters:
        vector_one (list): First embedding vector.
        vector_two (list): Second embedding vector.

        Returns:
        float: Cosine similarity between vectors.
        """
        vec1, vec2 = np.array(vector_one), np.array(vector_two)
        return np.dot(vec1, vec2) / (np.linalg.norm(vec1) * np.linalg.norm(vec2))

    def chunk_text(self, text):
        """
        Splits text into manageable chunks, attempting natural breaks.

        Parameters:
        text (str): Text to split into chunks.

        Returns:
        list: List of dictionaries containing chunk metadata.
        """
        separator = "\n"
        text = re.sub(r'\s+', ' ', text).strip()

        if len(text) <= self.chunk_size:
            return [{"chunk_id": 0, "text": text, "chunk_size": len(text)}]

        chunks, start, chunk_id = [], 0, 0

        while start < len(text):
            end = min(start + self.chunk_size, len(text))
            if separator in text[start:end]:
                end = start + text[start:end].rindex(separator) + len(separator)

            chunks.append({
                "chunk_id": chunk_id,
                "text": text[start:end],
                "chunk_size": end - start,
                "start_char": start,
                "end_char": end
            })

            # Advance start ensuring forward progress; avoid infinite loop at end of text
            next_start = end - self.chunk_overlap
            if next_start <= start:
                next_start = end
            start = next_start
            chunk_id += 1

        with open(f"chunks-{self.unique_filename}", 'w', newline='', encoding='utf-8') as csvfile:
            writer = csv.DictWriter(csvfile, fieldnames=["text", "chunk_size"])
            writer.writeheader()
            for chunk in chunks:
                writer.writerow({k: chunk[k] for k in ["text", "chunk_size"]})

        return chunks

    def calculate_embeddings(self):
        """
        Calculates embeddings for each chunk and stores them in a CSV file.

        Returns:
        DataFrame: DataFrame containing text chunks and their embeddings.
        """
        df = pd.read_csv(f"chunks-{self.unique_filename}", encoding='utf-8')
        df['embeddings'] = df['text'].apply(self.get_embedding)
        df.to_csv(f"embeddings-{self.unique_filename}", encoding='utf-8', index=False)
        return df

    def find_prompt_in_knowledge(self, prompt):
        """
        Finds and responds to a prompt based on similarity with embedded knowledge.

        Parameters:
        prompt (str): User input prompt.

        Returns:
        str: Response derived from the most similar chunk in knowledge.
        """
        prompt_embedding = self.get_embedding(prompt)
        df = pd.read_csv(f"embeddings-{self.unique_filename}", encoding='utf-8')
        df['embeddings'] = df['embeddings'].apply(lambda x: np.array(eval(x)))
        df['similarity'] = df['embeddings'].apply(lambda emb: self.calculate_similarity(prompt_embedding, emb))

        best_chunk = df.loc[df['similarity'].idxmax(), 'text']

        client = OpenAI(base_url = self.api_base, api_key=self.openai_api_key)
        response = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": f"You are {self.persona}, a knowledge-based assistant. Forget previous context."},
                {"role": "user", "content": f"Answer based only on this information: {best_chunk}. Prompt: {prompt}"}
            ],
            temperature=0
        )

        return response.choices[0].message.content

class EvaluationAgent:

    def __init__(
        self,
        openai_api_key: str,
        base_url: str,
        persona: str,
        evaluation_criteria: str,
        worker_agent,
        max_interactions: int,
    ):
        self.openai_api_key = openai_api_key
        self.base_url = base_url
        self.persona = persona
        self.evaluation_criteria = evaluation_criteria
        self.worker_agent = worker_agent
        self.max_interactions = max_interactions

    def evaluate(self, initial_prompt: str):
        client = OpenAI(base_url = self.base_url, api_key=self.openai_api_key)
        prompt_to_evaluate = initial_prompt
        last_worker_response = ""
        last_evaluation = ""

        for i in range(self.max_interactions):
            print(f"\n--- Interaction {i+1} ---")

            print(" Step 1: Worker agent generates a response to the prompt")
            print(f"Prompt:\n{prompt_to_evaluate}")
            response_from_worker = self.worker_agent.respond(prompt_to_evaluate)
            last_worker_response = response_from_worker
            print(f"Worker Agent Response:\n{response_from_worker}")

            print(" Step 2: Evaluator agent judges the response")
            eval_prompt = (
                f"Does the following answer: {response_from_worker}\n"
                f"Meet this criteria: {self.evaluation_criteria}\n"
                f"Respond Yes or No, and the reason why it does or doesn't meet the criteria."
            )
            response = client.chat.completions.create(
                model="gpt-3.5-turbo",
                messages=[
                    {"role": "system", "content": f"You are {self.persona}. Forget all previous context."},
                    {"role": "user", "content": eval_prompt},
                ],
                temperature=0,
            )
            evaluation = response.choices[0].message.content.strip()
            last_evaluation = evaluation
            print(f"Evaluator Agent Evaluation:\n{evaluation}")

            print(" Step 3: Check if evaluation is positive")
            if evaluation.lower().startswith("yes"):
                print("✅ Final solution accepted.")
                break
            else:
                print(" Step 4: Generate instructions to correct the response")
                instruction_prompt = (
                    f"Provide concise instructions to fix an answer based on these reasons why it is incorrect: {evaluation}"
                )
                response = client.chat.completions.create(
                    model="gpt-3.5-turbo",
                    messages=[
                        {"role": "system", "content": f"You are {self.persona}. Forget all previous context."},
                        {"role": "user", "content": instruction_prompt},
                    ],
                    temperature=0,
                )
                instructions = response.choices[0].message.content.strip()
                print(f"Instructions to fix:\n{instructions}")

                print(" Step 5: Send feedback to worker agent for refinement")
                prompt_to_evaluate = (
                    f"The original prompt was: {initial_prompt}\n"
                    f"The response to that prompt was: {response_from_worker}\n"
                    f"It has been evaluated as incorrect.\n"
                    f"Make only these corrections, do not alter content validity: {instructions}"
                )

        return {
            "final_response": last_worker_response,
            "evaluation": last_evaluation,
            "iterations": i + 1,
        }

class RoutingAgent:

    def __init__(self, openai_api_key: str, agents,base_url: str):
        self.openai_api_key = openai_api_key
        self.agents = agents
        self.base_url = base_url

    def get_embedding(self, text: str):
        client = OpenAI(base_url = self.base_url, api_key=self.openai_api_key)
        response = client.embeddings.create(
            model="text-embedding-3-large",
            input=text,
            encoding_format="float",
        )
        embedding = response.data[0].embedding
        return embedding

    def route(self, user_input: str):
        input_emb = self.get_embedding(user_input)
        best_agent = None
        best_score = -1.0

        for agent in self.agents:
            description = agent.get("description", "")
            if not description:
                continue
            agent_emb = self.get_embedding(description)
            if agent_emb is None:
                continue

            similarity = float(
                np.dot(input_emb, agent_emb)
                / (np.linalg.norm(input_emb) * np.linalg.norm(agent_emb))
            )
            print(f"[Router] Similarity with {agent.get('name', 'unknown')}: {similarity:.3f}")

            if similarity > best_score:
                best_score = similarity
                best_agent = agent

        if best_agent is None:
            return "Sorry, no suitable agent could be selected."

        print(f"[Router] Best agent: {best_agent['name']} (score={best_score:.3f})")
        return best_agent["func"](user_input)

class ActionPlanningAgent:

    def __init__(self, openai_api_key: str, knowledge: str, base_url: str):
        self.openai_api_key = openai_api_key
        self.knowledge = knowledge
        self.api_base = base_url

    def extract_steps_from_prompt(self, prompt: str):
        client = OpenAI(base_url = self.api_base, api_key=self.openai_api_key)
        response = client.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {
                    "role": "system",
                    "content": (
                        "You are an action planning agent. Using your knowledge, you extract from the user prompt the steps "
                        "requested to complete the action the user is asking for. You return the steps as a list. Only "
                        "return the steps in your knowledge. Forget any previous context. This is your knowledge: "
                        f"{self.knowledge}"
                    ),
                },
                {"role": "user", "content": prompt},
            ],
            temperature=0,
        )
        response_text = response.choices[0].message.content

        # Normalize and clean steps
        raw_lines = [line.strip() for line in response_text.split("\n")]
        cleaned = []
        for line in raw_lines:
            if not line:
                continue
            # Remove common list markers
            line = re.sub(r"^[-*]\s+", "", line)
            line = re.sub(r"^\d+\.\s+", "", line)
            cleaned.append(line)

        return cleaned
```