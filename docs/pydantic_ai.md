## PydanticAI Integration with G4F Client

This README provides an overview of how to integrate PydanticAI with the G4F client to create an agent that interacts with a language model. With this setup, you'll be able to apply patches to use PydanticAI with G4F, enable debugging, and run simple agent-based interactions synchronously. However, please note that tool calls within AI requests are currently **not fully supported** in this environment.

## Requirements

Before starting, make sure you have the following Python dependencies installed:

- `g4f`: A client that interfaces with various LLMs.
- `pydantic_ai`: A module that provides integration with Pydantic-based models.

### Installation

To install these dependencies, you can use `pip`:

```bash
pip install g4f pydantic_ai
```

## Step-by-Step Setup

### 1. Patch PydanticAI to Use G4F Models

In order to use PydanticAI with G4F models, you need to apply the necessary patch to the client. This can be done by importing `patch_infer_model` from `g4f.integration.pydantic_ai`. The `api_key` parameter is optional, so if you have one, you can provide it. If not, the system will proceed without it.

```python
from g4f.integration.pydantic_ai import patch_infer_model

patch_infer_model(api_key="your_api_key_here")  # Optional
```

If you don't have an API key, simply omit the `api_key` argument.

### 2. Enable Debug Logging

For troubleshooting and monitoring purposes, you may want to enable debug logging. This can be achieved by setting `g4f.debug.logging` to `True`.

```python
import g4f.debug

g4f.debug.logging = True
```

This will log detailed information about the internal processes and interactions.

### 3. Create a Simple Agent

Now you are ready to create a simple agent that can interact with the LLM. The agent is initialized with a model, and you can also define a system prompt. Here's an example where a basic agent is created with the model `g4f:Gemini:Gemini` and a simple system prompt:

```python
from pydantic_ai import Agent

# Define the agent
agent = Agent(
    'g4f:Gemini:Gemini', # g4f:provider:model_name or g4f:model_name
    system_prompt='Be concise, reply with one sentence.',
)
```

### 4. Run the Agent Synchronously

Once the agent is set up, you can run it synchronously to interact with the LLM. The `run_sync` method sends a query to the LLM and returns the result.

```python
# Run the agent synchronously with a user query
result = agent.run_sync('Where does "hello world" come from?')

# Output the response
print(result.data)
```

In this example, the agent will send the system prompt along with the user query (`"Where does 'hello world' come from?"`) to the LLM. The LLM will process the request and return a concise answer.

### Example Output

```bash
The phrase "hello world" is commonly used in programming tutorials to demonstrate basic syntax and the concept of outputting text to the screen.
```

## Tool Calls and Limitations

**Important**: Tool calls (such as applying external functions or calling APIs within the AI request itself) are **currently not fully supported**. If your system relies on invoking specific external tools or functions during the conversation with the model, you will need to implement this functionality outside the agent's context or handle it before or after the agent's request.

For example, you can process your query or interact with external systems before passing the data to the agent.

---

### Simple Example without `patch_infer_model`

```python
from pydantic_ai import Agent
from g4f.integration.pydantic_ai import AIModel

agent = Agent(
    AIModel("gpt-4o"),
)

result = agent.run_sync('Are you gpt-4o?')
print(result.data)
```

This example shows how to initialize an agent with a specific model (`gpt-4o`) and run it synchronously.

---

### Full Example with Tool Calls:

```python
from pydantic import BaseModel
from pydantic_ai import Agent
from pydantic_ai.models import ModelSettings
from g4f.integration.pydantic_ai import AIModel
from g4f.Provider import PollinationsAI


class MyModel(BaseModel):
    city: str
    country: str

agent = Agent(AIModel(
    "gpt-4o", # Specify the provider and model
    PollinationsAI # Use a supported provider to handle tool-based response formatting
), result_type=MyModel, model_settings=ModelSettings(temperature=0))

if __name__ == '__main__':
    result = agent.run_sync('The windy city in the US of A.')
    print(result.data)
    print(result.usage())
```

This example demonstrates the use of a custom Pydantic model (`MyModel`) to capture structured data (city and country) from the response and running the agent with specific model settings.

---

### Support for Models/Providers without Tool Call Support

For models/providers that do not fully support tool calls or lack a direct API for structured output, the `ToolSupportProvider` can be used to bridge the gap. This provider ensures that the agent properly formats the response, even when the model itself doesn't have built-in support for structured outputs. It does so by leveraging a tool list and creating a response format when only one tool is used.

### Example for Models/Providers without Tool Support (Single Tool Usage)

```python
from pydantic import BaseModel
from pydantic_ai import Agent
from pydantic_ai.models import ModelSettings
from g4f.integration.pydantic_ai import AIModel
from g4f.providers.tool_support import ToolSupportProvider

from g4f import debug
debug.logging = True

# Define a custom model for structured output (e.g., city and country)
class MyModel(BaseModel):
    city: str
    country: str

# Create the agent for a model with tool support (using one tool)
agent = Agent(AIModel(
    "OpenaiChat:gpt-4o", # Specify the provider and model
    ToolSupportProvider # Use ToolSupportProvider to handle tool-based response formatting
), result_type=MyModel, model_settings=ModelSettings(temperature=0))

if __name__ == '__main__':
    # Run the agent with a query to extract information (e.g., city and country)
    result = agent.run_sync('European city with the bear.')
    print(result.data)  # Structured output of city and country
    print(result.usage()) # Usage statistics
```

### Explanation:

- **`ToolSupportProvider` as a Bridge:** The `ToolSupportProvider` acts as a bridge between the agent and the model, ensuring that the response is formatted into a structured output, even if the model doesn't have an API that directly supports such formatting.
  
  - For instance, if the model generates raw text or unstructured data, the `ToolSupportProvider` will convert this into the expected format (like `MyModel`), allowing the agent to process it as structured data.
  
- **Model Initialization:** We initialize the agent with the `PollinationsAI:openai` model, which may not have a built-in API for returning structured outputs. Instead, it relies on the `ToolSupportProvider` to format the output.

- **Custom Result Model:** We define a custom Pydantic model (`MyModel`) to capture the expected output in a structured way (e.g., `city` and `country` fields). This helps ensure that even when the model doesn't support structured data, the agent can interpret and format it.

- **Debug Logging:** The `g4f.debug.logging` is enabled to provide detailed logs for troubleshooting and monitoring the agent's execution.

### Example Output:

```bash
city='Berlin'
country='Germany'
usage={'prompt_tokens': 15, 'completion_tokens': 50}
```

### Key Points:

- **`ToolSupportProvider` Role:** The `ToolSupportProvider` ensures that the agent formats the raw or unstructured response from the model into a structured format, even if the model itself lacks built-in support for structured data.
  
- **Single Tool Usage:** The `ToolSupportProvider` is particularly useful when only one tool is used by the model, and it needs to format or transform the model's output into a structured response without additional tools.

### Notes:

- This approach is ideal for models that return unstructured text or data that needs to be transformed into a structured format (e.g., Pydantic models).
- The `ToolSupportProvider` bridges the gap between the model's output and the expected structured format, enabling seamless integration into workflows that require structured responses.

---

## LangChain Integration Example

For users working with LangChain, here is an example demonstrating how to integrate G4F models into a LangChain environment:

```python
from g4f.integration.langchain import ChatAI
import g4f.debug

# Enable debugging logs
g4f.debug.logging = True

llm = ChatAI(
    model="llama3-70b-8192",
    provider="Groq",
    api_key=""  # Optionally add your API key here
)

messages = [
    {"role": "user", "content": "2 🦜 2"},
    {"role": "assistant", "content": "4 🦜"},
    {"role": "user", "content": "2 🦜 3"},
    {"role": "assistant", "content": "5 🦜"},
    {"role": "user", "content": "3 🦜 4"},
]

response = llm.invoke(messages)
assert(response.content == "7 🦜")
```

This example shows how to use LangChain's `ChatAI` integration to create a conversational agent with a G4F model. The interaction takes place with the given messages and the agent processes them step-by-step to return the expected output.

---

## Conclusion

By following these steps, you have successfully integrated G4F models into PydanticAI, created an agent, and enabled debugging. This allows you to conduct conversations with the language model, pass system prompts, and retrieve responses synchronously.

### Notes:
- The `api_key` parameter when calling `patch_infer_model` is optional. If you don’t provide it, the system will still work without an API key.
- Modify the agent’s `system_prompt` to suit the nature of the conversation you wish to have.
- **Tool calls within AI requests are not fully supported** at the moment. Use the agent's basic functionality for generating responses and handle external calls separately.

For further customization and advanced use cases, refer to the G4F and PydanticAI documentation.

---

[Return to Documentation](README.md)