---
layout: post
title: Intro to Agents - Building ReAct from Scratch
---

Most developers interact with AI agents through high-level APIs like Anthropic's native tool use. But what's actually happening under the hood? This post demystifies agent architectures by building a basic ReAct agent from scratch.

The goal isn't to build production-ready code - it's to understand the fundamental pattern that powers AI agents. We'll implement the core reasoning loop at the model level, similar to what happens beneath Anthropic's API layer.

## What is ReAct?

**ReAct** stands for **Reasoning + Acting**. It's a pattern where language models alternate between:

1. **Reasoning** - Thinking about what to do next
2. **Acting** - Executing a tool/action
3. **Observing** - Receiving the tool result
4. **Repeat** - Using the observation to reason about the next step

This creates a loop: **Thought → Action → Observation → Thought → Action...**

The key insight: by interleaving reasoning with actions, the model can dynamically adapt its plan based on actual results rather than trying to predict everything upfront.

### ReAct vs Alternatives

- **Chain of Thought (CoT)**: Pure reasoning with no external actions. The model thinks through a problem step-by-step but can't interact with the world.

- **Plan and Execute**: Creates a complete plan upfront, then executes it. Less flexible when unexpected results occur.

- **Reflexion**: ReAct + self-reflection. The agent critiques its own performance and learns from failures across multiple attempts.

- **Tree of Thoughts**: Explores multiple reasoning branches in parallel, evaluating different paths before committing to one.

ReAct strikes a balance - it's dynamic enough to adapt but simple enough to implement and reason about.

## Architecture Overview

Our implementation has three components:

1. **tool_funcs.py** - Actual tool implementations (e.g., `get_weather()`, `search_web()`)
2. **tool_descriptions.json** - Tool schemas describing parameters and usage
3. **agent.py** - The Agent class that orchestrates the ReAct loop

## Implementation

### The Agent Class

The `__init__` method loads tool definitions and constructs a system prompt that instructs the model to output tool calls as JSON. Instead of using Anthropic's native tool use format (which parses special tokens under their API), we're prompting the already-trained model to express tool calls in raw JSON format.

```python
def __init__(self):
    self.client = Anthropic(
        api_key=os.environ.get("ANTHROPIC_API_KEY"),
    )

    self.history = []

    # Load tool descriptions
    with open('tool_descriptions.json', 'r') as f:
        tools = json.load(f)

    tool_definitions = json.dumps(tools, indent=2)

    self.system_prompt = f"""
        You are an agent.

        You have access to some tools to help you answer the user's question.
        Here's a list of tool you can access:
        {tool_definitions}

        The output should be:
        "{{'tool': '<function name>', 'inputs': {{'<input name>': 'input_value'}}}}"

        Here is an example with get weather:
        "{{'tool': 'get_weather', 'inputs': {{'location': 'San Francisco'}}}}"

        When you need to call a tool, return JUST THE json dictionary with the tool call parameters filled in. no other text.
        """
```

**What's happening here:** Anthropic's API intercepts special tokens like `<function_calls>` that the model outputs and converts them into structured tool call objects. We're bypassing this by prompting the model to output JSON directly in its text response. The model was already trained to do tool calling - we're just asking it to use a different output format than what the API usually parses.

### The ReAct Loop

The `run()` method implements the core ReAct pattern. There are two nested loops:

1. **Outer loop** - Handles conversation turns (user input → agent response)
2. **Inner loop** - Handles the ReAct cycle (reasoning → tool calls → observations)

```python
def run(self):
    print(self.system_prompt)
    while True:
        # Get user input
        user_message = input("User: ")
        self.history.append({
            "role": "user",
            "content": user_message,
        })

        # Inner ReAct loop
        while True:
            # REASONING: Model decides what to do
            model_response_message = self.client.messages.create(
                max_tokens=1024,
                messages=self.history,
                model="claude-sonnet-4-5-20250929",
                system=self.system_prompt,
            )
            self.history.append({
                "role": "assistant",
                "content": model_response_message.content,
            })

            # Check if model wants to take an ACTION
            if model_response_message.content[0].text.startswith("```"):
                try:
                    # Parse the tool call JSON
                    lines = model_response_message.content[0].text.splitlines()
                    if lines[0].startswith("```"):
                        lines = lines[1:]
                    if lines and lines[-1].startswith("```"):
                        lines = lines[:-1]
                    text = "\n".join(lines).strip()
                    decoded_text = json.loads(text)
                    print('     Agentic tool call: ', decoded_text)
                    func_name = decoded_text['tool']
                    params = decoded_text.get('inputs', {})

                    # Execute the tool
                    if hasattr(tool_funcs, func_name):
                        try:
                            func = getattr(tool_funcs, func_name)
                            result = func(**params)
                            print(f'     Tool result: {result}')

                            # OBSERVATION: Feed result back to model
                            self.history.append({
                                "role": "user",
                                "content": 'The tool call returned: ' + result,
                            })
                            print(self.history)
                        except Exception as e:
                            print(f'Error calling {func_name}: {e}')
                    else:
                        print(f'Unknown tool: {func_name}')
                except json.JSONDecodeError as e:
                    print("JSON decode error:", e)
                    return None
            else:
                # Model returned final answer, exit inner loop
                print('Agent: ', model_response_message.content[0].text)
                break
```

### Example Output

Here's what the ReAct loop looks like in action when a user asks about the weather:

```
User: What's the weather in San Francisco and should I bring an umbrella?

[Agent thinking...]
     Agentic tool call: {'tool': 'get_weather', 'inputs': {'location': 'San Francisco'}}
     Tool result: {"temperature": "68°F", "conditions": "Sunny", "precipitation": "0%"}

[Agent thinking with weather data...]
Agent: The weather in San Francisco is currently sunny and 68°F with no precipitation expected. You won't need an umbrella today!
```

The ReAct loop breakdown:
1. **Reasoning**: Model decides it needs weather data to answer
2. **Acting**: Outputs JSON tool call for `get_weather`
3. **Observing**: Receives weather results back in conversation history
4. **Reasoning**: Model now has enough info, outputs final answer (exits loop)

### Understanding the ReAct Loop

The inner `while True` loop is where ReAct happens:

1. **Reasoning**: The model receives the conversation history and decides whether to call a tool or answer directly
2. **Acting**: If the model outputs JSON (wrapped in code blocks), we parse it and execute the specified tool
3. **Observing**: Tool results are appended to history as a user message
4. **Repeat**: Loop continues, model sees the observation and reasons about next steps

The loop exits when the model outputs plain text instead of a tool call - this signals it has enough information to answer the user.

**Important note:** Claude models were trained to understand tool calling during their pre-training and fine-tuning. They can adapt to different output formats through prompting. Anthropic's API handles the parsing layer (converting `<function_calls>` tokens to structured objects) - we're implementing our own parsing for raw JSON output here for educational purposes.

## Gaps to Production

This implementation is intentionally minimal. Production agents need:

**Critical missing pieces:**
- **Max iterations limit** - Prevent infinite tool calling loops
- **Error recovery** - Handle malformed JSON, invalid tool calls, tool execution failures
- **Token management** - Conversation history grows unbounded
- **Concurrency** - Running multiple tool calls in parallel
- **Observability** - Logging, tracing, debugging agentic behavior

**Interesting extensions** (for future posts):
- Context window management strategies
- Multi-agent orchestration
- Tool selection optimization
- Handling tool call errors gracefully
- Agent memory and state management

## Conclusion

The ReAct pattern is surprisingly simple: give a model tools, let it reason about when to use them, show it the results, and repeat. The complexity comes from making this reliable, efficient, and observable at scale.

By building this from scratch, you can see what APIs like Anthropic's tool use are abstracting away - prompt engineering for tool output format, parsing model responses, executing tools, and feeding results back into context.

Understanding these fundamentals helps you debug agent behavior, optimize performance, and build more sophisticated agentic systems. 


