Me :&#x20;

you are the expert of prompt engineer, I am creating a application , which user LLM (claude or chatgpt) to generate user story from the piece of user requirement. will plan two use muptiple agent. how to use **Self-refine, critique the results or Maieutic prompting**

C:

As an expert in prompt engineering, I can guide you on using self-refine techniques, critique methods, and Maieutic prompting to create an application that generates user stories from user requirements using LLMs like Claude or ChatGPT. Here's a strategy incorporating multiple agents and these techniques:

1.  Initial Story Generation Agent:
    This agent takes the user requirement and generates an initial user story.

Prompt:
"Generate a user story based on the following requirement: \[Insert requirement here]. Format the user story using the standard 'As a \[user], I want \[goal] so that \[benefit]' structure."

1.  Self-Refine Agent:
    This agent reviews and improves the initial user story.

Prompt:
"Review and refine the following user story: \[Insert initial user story]. Consider the following aspects:

1.  Clarity: Is the story easy to understand?
2.  Completeness: Does it capture all aspects of the requirement?
3.  Actionability: Can developers easily derive tasks from this story?
4.  Value: Is the benefit clearly stated?

Provide an improved version of the user story."

1.  Critique Agent:
    This agent critically analyzes the refined user story.

Prompt:
"Critically analyze the following user story: \[Insert refined user story]. Evaluate it based on these criteria:

1.  INVEST principles (Independent, Negotiable, Valuable, Estimable, Small, Testable)
2.  Alignment with original requirement
3.  Potential edge cases or scenarios not covered

Provide a detailed critique highlighting strengths and areas for improvement."

1.  Maieutic Prompting Agent:
    This agent uses a series of questions to further refine the user story.

Prompt:
"Consider the following user story: \[Insert critiqued user story]. Answer these questions to improve it:

1.  Who exactly is the user in this story? Can we be more specific?
2.  What is the core action or goal? Is it clearly stated?
3.  Why is this story important? Is the benefit evident and compelling?
4.  Are there any assumptions we're making that should be clarified?
5.  How might this story impact other parts of the system?

Based on your answers, propose a final, improved version of the user story."

1.  Synthesis Agent:
    This agent combines insights from all previous agents to produce the final user story.

Prompt:
"Review the following:

1.  Original requirement: \[Insert original requirement]
2.  Initial user story: \[Insert initial user story]
3.  Refined user story: \[Insert refined user story]
4.  Critique: \[Insert critique]
5.  Maieutic improvements: \[Insert maieutic agent's output]

Synthesize all this information to create a final, optimized user story that best represents the original requirement while incorporating all improvements and addressing all critiques."

To implement this system:

1.  Create a pipeline where each agent's output feeds into the next.
2.  Use the Synthesis Agent's output as the final result.
3.  Optionally, add a human-in-the-loop step after the Synthesis Agent for final approval or adjustments.
4.  Consider implementing a feedback loop where the final story can be fed back into the system for further refinement if needed.

This approach leverages self-refinement through multiple iterations, critical analysis through the Critique Agent, and deeper exploration of the problem space through Maieutic prompting. The result should be a well-crafted, thoroughly considered user story that accurately reflects the original requirement.

Me:

please provide sample code for above agents

C:

Certainly! I'll provide sample Python code for implementing the agents described above. We'll use OpenAI's API for this example, but you can easily adapt it for Claude or other LLMs.

```python
import openai
import os

# Set up OpenAI API key
openai.api_key = os.getenv("OPENAI_API_KEY")

def generate_completion(prompt, max_tokens=150):
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=max_tokens
    )
    return response.choices[0].message.content.strip()

class InitialStoryGenerator:
    def generate(self, requirement):
        prompt = f"Generate a user story based on the following requirement: {requirement}. Format the user story using the standard 'As a [user], I want [goal] so that [benefit]' structure."
        return generate_completion(prompt)

class SelfRefineAgent:
    def refine(self, user_story):
        prompt = f"""Review and refine the following user story: {user_story}. Consider the following aspects:
        1. Clarity: Is the story easy to understand?
        2. Completeness: Does it capture all aspects of the requirement?
        3. Actionability: Can developers easily derive tasks from this story?
        4. Value: Is the benefit clearly stated?
        
        Provide an improved version of the user story."""
        return generate_completion(prompt)

class CritiqueAgent:
    def critique(self, user_story):
        prompt = f"""Critically analyze the following user story: {user_story}. Evaluate it based on these criteria:
        1. INVEST principles (Independent, Negotiable, Valuable, Estimable, Small, Testable)
        2. Alignment with original requirement
        3. Potential edge cases or scenarios not covered
        
        Provide a detailed critique highlighting strengths and areas for improvement."""
        return generate_completion(prompt, max_tokens=250)

class MaieuticPromptingAgent:
    def improve(self, user_story):
        prompt = f"""Consider the following user story: {user_story}. Answer these questions to improve it:
        1. Who exactly is the user in this story? Can we be more specific?
        2. What is the core action or goal? Is it clearly stated?
        3. Why is this story important? Is the benefit evident and compelling?
        4. Are there any assumptions we're making that should be clarified?
        5. How might this story impact other parts of the system?
        
        Based on your answers, propose a final, improved version of the user story."""
        return generate_completion(prompt, max_tokens=250)

class SynthesisAgent:
    def synthesize(self, original_requirement, initial_story, refined_story, critique, maieutic_improvements):
        prompt = f"""Review the following:
        1. Original requirement: {original_requirement}
        2. Initial user story: {initial_story}
        3. Refined user story: {refined_story}
        4. Critique: {critique}
        5. Maieutic improvements: {maieutic_improvements}
        
        Synthesize all this information to create a final, optimized user story that best represents the original requirement while incorporating all improvements and addressing all critiques."""
        return generate_completion(prompt, max_tokens=200)

def generate_user_story(requirement):
    initial_generator = InitialStoryGenerator()
    self_refine_agent = SelfRefineAgent()
    critique_agent = CritiqueAgent()
    maieutic_agent = MaieuticPromptingAgent()
    synthesis_agent = SynthesisAgent()

    initial_story = initial_generator.generate(requirement)
    refined_story = self_refine_agent.refine(initial_story)
    critique = critique_agent.critique(refined_story)
    maieutic_improvements = maieutic_agent.improve(refined_story)
    final_story = synthesis_agent.synthesize(requirement, initial_story, refined_story, critique, maieutic_improvements)

    return final_story

# Example usage
if __name__ == "__main__":
    requirement = "Create a feature that allows users to sort their todo list items by priority and due date."
    final_user_story = generate_user_story(requirement)
    print("Final User Story:")
    print(final_user_story)

```

This code implements the multi-agent approach we discussed for generating user stories. Here's a breakdown of the main components:

1.  We use OpenAI's API for generating completions. You can replace this with Claude's API or any other LLM API you prefer.

2.  Each agent is implemented as a separate class with a primary method (e.g., `generate`, `refine`, `critique`, etc.).

3.  The `generate_user_story` function orchestrates the entire process, calling each agent in sequence and passing the results to the next agent.

4.  The `generate_completion` function is a helper that sends prompts to the LLM and returns the response.

To use this code:

1.  Install the OpenAI Python library: `pip install openai`
2.  Set your OpenAI API key as an environment variable: `export OPENAI_API_KEY='your-api-key-here'`
3.  Run the script, providing a requirement as input.

This implementation provides a solid foundation for your user story generation application. You can further enhance it by:

1.  Adding error handling and retries for API calls.
2.  Implementing a caching mechanism to reduce API calls for similar requirements.
3.  Creating a user interface (web or CLI) for easier interaction.
4.  Adding a feedback loop where users can rate or edit the generated stories.
5.  Implementing parallel processing for the agents that can work independently.

Would you like me to explain any part of the code in more detail?

Me:

please review above context history, then think about any better approach

C:

Certainly. Let me review our conversation and consider if there might be any better approaches to generating user stories from requirements using LLMs and multiple agents.

Upon reflection, here are some improvements we could make to our approach:

1.  Iterative Refinement Loop:
    Instead of a linear pipeline, we could implement an iterative refinement loop where the story goes through multiple rounds of improvement until a certain quality threshold is met or a maximum number of iterations is reached.

2.  Consistency Check Agent:
    We could introduce a new agent that checks the consistency between the final user story and the original requirement, ensuring we haven't drifted too far from the initial intent.

3.  Prioritization Agent:
    Add an agent that assigns priority to the user story based on its perceived importance and complexity.

4.  Acceptance Criteria Generator:
    Introduce an agent that generates acceptance criteria for each user story, making it more actionable for the development team.

5.  Stakeholder Perspective Agent:
    Create an agent that evaluates the user story from different stakeholder perspectives (e.g., end-user, business, development team) to ensure it meets various needs.

6.  Context-Aware Agent:
    Implement an agent that considers the broader context of the project or product, ensuring the user story aligns with overall goals and existing features.

7.  Natural Language Processing (NLP) Enhancement:
    Incorporate NLP techniques to analyze the semantic similarity between the original requirement and the generated user story, ensuring key concepts are preserved.

8.  Multi-Model Approach:
    Instead of relying on a single LLM, we could use different models for different tasks and compare or ensemble their outputs for potentially better results.

9.  Human-in-the-Loop Integration:
    Design the system to allow for easy human intervention at key points, allowing for manual adjustments or approvals.

10. Metadata Tagging:
    Add an agent that tags the user story with relevant metadata (e.g., feature area, estimated complexity, related stories) to aid in project management.

Let's implement some of these improvements in our code:

```python
import openai
import os
from typing import List, Dict
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

openai.api_key = os.getenv("OPENAI_API_KEY")

def generate_completion(prompt: str, max_tokens: int = 150) -> str:
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=max_tokens
    )
    return response.choices[0].message.content.strip()

def get_embedding(text: str) -> List[float]:
    response = openai.Embedding.create(
        model="text-embedding-ada-002",
        input=text
    )
    return response['data'][0]['embedding']

class UserStoryAgent:
    def process(self, input_data: Dict) -> Dict:
        raise NotImplementedError

class InitialStoryGenerator(UserStoryAgent):
    def process(self, input_data: Dict) -> Dict:
        requirement = input_data['requirement']
        prompt = f"Generate a user story based on the following requirement: {requirement}. Format the user story using the standard 'As a [user], I want [goal] so that [benefit]' structure."
        return {'user_story': generate_completion(prompt)}

class StoryRefiner(UserStoryAgent):
    def process(self, input_data: Dict) -> Dict:
        user_story = input_data['user_story']
        prompt = f"Refine the following user story, improving clarity, completeness, actionability, and value: {user_story}"
        return {'user_story': generate_completion(prompt)}

class ConsistencyChecker(UserStoryAgent):
    def process(self, input_data: Dict) -> Dict:
        requirement = input_data['requirement']
        user_story = input_data['user_story']
        req_embedding = get_embedding(requirement)
        story_embedding = get_embedding(user_story)
        similarity = cosine_similarity([req_embedding], [story_embedding])[0][0]
        return {'consistency_score': similarity, 'user_story': user_story}

class PrioritizationAgent(UserStoryAgent):
    def process(self, input_data: Dict) -> Dict:
        user_story = input_data['user_story']
        prompt = f"Assign a priority (High, Medium, Low) to this user story and explain why: {user_story}"
        result = generate_completion(prompt)
        priority = result.split('\n')[0].split(':')[-1].strip()
        explanation = '\n'.join(result.split('\n')[1:])
        return {'priority': priority, 'priority_explanation': explanation, 'user_story': user_story}

class AcceptanceCriteriaGenerator(UserStoryAgent):
    def process(self, input_data: Dict) -> Dict:
        user_story = input_data['user_story']
        prompt = f"Generate 3-5 acceptance criteria for this user story: {user_story}"
        return {'acceptance_criteria': generate_completion(prompt), 'user_story': user_story}

class UserStoryPipeline:
    def __init__(self, agents: List[UserStoryAgent], max_iterations: int = 3, consistency_threshold: float = 0.8):
        self.agents = agents
        self.max_iterations = max_iterations
        self.consistency_threshold = consistency_threshold

    def run(self, requirement: str) -> Dict:
        data = {'requirement': requirement}
        for i in range(self.max_iterations):
            for agent in self.agents:
                data.update(agent.process(data))
            
            if 'consistency_score' in data and data['consistency_score'] >= self.consistency_threshold:
                break
        
        return data

# Example usage
if __name__ == "__main__":
    pipeline = UserStoryPipeline([
        InitialStoryGenerator(),
        StoryRefiner(),
        ConsistencyChecker(),
        PrioritizationAgent(),
        AcceptanceCriteriaGenerator()
    ])

    requirement = "Create a feature that allows users to sort their todo list items by priority and due date."
    result = pipeline.run(requirement)

    print("Final User Story:")
    print(result['user_story'])
    print("\nPriority:", result['priority'])
    print("Priority Explanation:", result['priority_explanation'])
    print("\nAcceptance Criteria:")
    print(result['acceptance_criteria'])
    print("\nConsistency Score:", result['consistency_score'])

```

This enhanced version incorporates several improvements:

1.  Iterative Refinement: The `UserStoryPipeline` class allows for multiple iterations of refinement until a consistency threshold is met or a maximum number of iterations is reached.

2.  Consistency Checker: We've added a `ConsistencyChecker` that uses embeddings to calculate the semantic similarity between the original requirement and the generated user story.

3.  Prioritization Agent: A `PrioritizationAgent` assigns priority to the user story and provides an explanation.

4.  Acceptance Criteria Generator: An `AcceptanceCriteriaGenerator` creates acceptance criteria for the user story.

5.  Modular Design: The code is now more modular, with each agent inheriting from a base `UserStoryAgent` class, making it easier to add or modify agents.

6.  Type Hinting: We've added type hints to improve code readability and catch potential type-related errors.

7.  Embedding-based Similarity: We use OpenAI's text embedding model to calculate semantic similarity, providing a more sophisticated consistency check.

This improved system offers several advantages:

*   More refined output: The iterative process allows for continuous improvement of the user story.
*   Better alignment with requirements: The consistency checker ensures the final story doesn't drift too far from the original requirement.
*   Enhanced usability: The addition of priority and acceptance criteria makes the output more immediately useful for development teams.
*   Flexibility: The modular design makes it easy to add new agents or modify the pipeline as needed.

To further improve this system, you could:

1.  Implement error handling and retries for API calls.
2.  Add logging to track the refinement process.
3.  Create a user interface for easier interaction with the system.
4.  Implement caching to reduce API calls for similar requirements.
5.  Add a human-in-the-loop option for final approval or adjustments.

Would you like me to elaborate on any part of this enhanced approach?

Me:&#x20;

elaborate more about human-in-the-loop

C:

Certainly. The human-in-the-loop approach is a crucial aspect of AI-assisted systems, especially for tasks like user story generation where context, nuance, and domain knowledge are important. Let's explore how we can integrate this concept into our user story generation system.

The goal of human-in-the-loop (HITL) in this context is to leverage the strengths of both AI and human expertise. The AI can quickly generate and refine user stories, while humans can provide critical judgment, domain-specific knowledge, and final approval.

Here are some ways we can implement HITL in our user story generation system:

1.  Review and Approval Step:
    After the AI generates the final user story, we can add a step where a human (likely a product manager or business analyst) reviews and approves the story before it's considered complete.

2.  Iterative Feedback:
    Allow humans to provide feedback on the generated user story, which the system can then use to further refine the story.

3.  Customization Points:
    Give humans the ability to customize certain aspects of the story, such as adjusting the priority or modifying the acceptance criteria.

4.  Exception Handling:
    For cases where the AI's output doesn't meet certain criteria (e.g., low consistency score), automatically flag it for human review.

5.  Training Data Generation:
    Use human-approved stories as training data to improve the AI's performance over time.

Let's implement some of these ideas by extending our previous code:

```python
import openai
import os
from typing import List, Dict
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

openai.api_key = os.getenv("OPENAI_API_KEY")

# ... [Previous code for generate_completion, get_embedding, and agent classes remains the same]

class HumanInTheLoopAgent:
    def get_human_input(self, prompt: str) -> str:
        print(prompt)
        return input().strip()

    def review_story(self, story: Dict) -> Dict:
        print("\nPlease review the following user story:")
        print(f"User Story: {story['user_story']}")
        print(f"Priority: {story['priority']}")
        print(f"Acceptance Criteria: {story['acceptance_criteria']}")
        
        approval = self.get_human_input("Do you approve this user story? (yes/no): ")
        
        if approval.lower() != 'yes':
            feedback = self.get_human_input("Please provide feedback for improvement: ")
            return {'approved': False, 'feedback': feedback}
        
        return {'approved': True}

    def customize_story(self, story: Dict) -> Dict:
        print("\nDo you want to customize any part of the user story?")
        customize = self.get_human_input("Enter 'u' for user story, 'p' for priority, 'a' for acceptance criteria, or 'n' for none: ")
        
        if customize.lower() == 'u':
            story['user_story'] = self.get_human_input("Enter the revised user story: ")
        elif customize.lower() == 'p':
            story['priority'] = self.get_human_input("Enter the new priority (High/Medium/Low): ")
        elif customize.lower() == 'a':
            story['acceptance_criteria'] = self.get_human_input("Enter the revised acceptance criteria: ")
        
        return story

class UserStoryPipeline:
    def __init__(self, agents: List[UserStoryAgent], hitl_agent: HumanInTheLoopAgent, max_iterations: int = 3, consistency_threshold: float = 0.8):
        self.agents = agents
        self.hitl_agent = hitl_agent
        self.max_iterations = max_iterations
        self.consistency_threshold = consistency_threshold

    def run(self, requirement: str) -> Dict:
        data = {'requirement': requirement}
        for i in range(self.max_iterations):
            for agent in self.agents:
                data.update(agent.process(data))
            
            if 'consistency_score' in data and data['consistency_score'] >= self.consistency_threshold:
                break
        
        # Human-in-the-loop review
        while True:
            review_result = self.hitl_agent.review_story(data)
            if review_result['approved']:
                break
            else:
                # Use feedback to refine the story
                data['user_story'] += f"\nFeedback: {review_result['feedback']}"
                data = self.agents[1].process(data)  # Assume agents[1] is StoryRefiner
        
        # Allow human customization
        data = self.hitl_agent.customize_story(data)
        
        return data

# Example usage
if __name__ == "__main__":
    hitl_agent = HumanInTheLoopAgent()
    pipeline = UserStoryPipeline([
        InitialStoryGenerator(),
        StoryRefiner(),
        ConsistencyChecker(),
        PrioritizationAgent(),
        AcceptanceCriteriaGenerator()
    ], hitl_agent)

    requirement = "Create a feature that allows users to sort their todo list items by priority and due date."
    result = pipeline.run(requirement)

    print("\nFinal Approved User Story:")
    print(result['user_story'])
    print("\nPriority:", result['priority'])
    print("\nAcceptance Criteria:")
    print(result['acceptance_criteria'])
    print("\nConsistency Score:", result['consistency_score'])

```

This enhanced version integrates human-in-the-loop functionality into our user story generation system. Here's a breakdown of the key additions:

1.  HumanInTheLoopAgent:
    *   This new class handles all interactions with the human reviewer.
    *   It includes methods for getting human input, reviewing the generated story, and customizing parts of the story.

2.  Review Process:
    *   After the AI generates a user story, it's presented to the human for review.
    *   The human can approve the story or provide feedback for improvement.
    *   If feedback is provided, the story goes through another round of refinement.

3.  Customization:
    *   After approval, the human has the option to customize specific parts of the user story (the story itself, priority, or acceptance criteria).

4.  Iterative Improvement:
    *   The pipeline now includes a loop that continues until the human approves the story.
    *   This ensures that the final output meets the human reviewer's standards.

5.  Flexibility:
    *   The system allows for different levels of human involvement. Reviewers can simply approve/reject, provide detailed feedback, or make direct edits.

Benefits of this HITL approach:

1.  Quality Assurance: Humans can catch nuances or context-specific details that the AI might miss.
2.  Continuous Improvement: Human feedback can be used to improve the AI's performance over time.
3.  Customization: Allows for easy adjustments to fit specific project needs or writing styles.
4.  Stakeholder Confidence: Having a human review step can increase stakeholder confidence in the generated user stories.
5.  Handling Edge Cases: Humans can intervene when the AI struggles with unusual or complex requirements.

To further enhance this system, you could consider:

1.  Implementing a GUI for easier human interaction.
2.  Adding a feedback collection mechanism to store human edits for future AI training.
3.  Incorporating role-based reviews (e.g., separate reviews by product manager, developer, and QA).
4.  Implementing an audit trail to track changes and approvals.
5.  Adding a feature to compare multiple AI-generated versions before human approval.

This human-in-the-loop approach strikes a balance between leveraging AI capabilities and ensuring human oversight, resulting in high-quality, context-appropriate user stories.
