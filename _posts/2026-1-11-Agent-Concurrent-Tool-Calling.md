---
layout: post
title: Adding Concurrent Tool Calling to Our Agent
---

In the [previous post](/2026/01/10/ReAct-Pattern.html), we built a basic ReAct agent that executes tool calls sequentially. But imagine the agent needs to check the weather in three cities, or fetch data from multiple APIs simultaneously. Running these sequentially wastes time - they're independent operations that could happen in parallel.

The model can return multiple tool calls at once. Instead of executing them one-by-one, we want to run them concurrently and collect all results before continuing. Since tool calls typically involve I/O (API calls, database queries, file operations) rather than heavy computation, we don't need true parallelism - we just need to avoid blocking while waiting for responses.

This is a perfect use case for Python's async/await.

## Python Concurrency Crash Course

Before diving into the code, let's understand Python's concurrency landscape. First, the constraint we're working with:

```
Python program
└── Python process
    └── Python interpreter
        └── One GIL (Global Interpreter Lock)
            └── Many kernel threads
```

**The GIL means only one thread can execute Python bytecode at a time per process.** This shapes our three concurrency approaches:

### 1. AsyncIO - Cooperative Multitasking

```
AsyncIO
└── 1 thread
    └── event loop
        └── many tasks (cooperative yielding)
```

**Best for:** I/O-bound work where libraries support async (HTTP requests with `httpx`, database queries with `asyncpg`, file I/O with `aiofiles`)

**How it works:** One thread runs an event loop. Tasks voluntarily yield control at `await` points. While one task waits for I/O, others can run.

### 2. Threading - Blocking I/O Concurrency

```
Thread Pool
└── N OS threads
    └── blocking I/O
    └── GIL released during I/O waits
```

**Best for:** Blocking I/O operations using synchronous libraries (`requests`, standard file I/O, `time.sleep()`)

**How it works:** Multiple OS threads. When a thread hits I/O and blocks, the GIL is released so other threads can run. NOT useful for CPU-bound work due to GIL.

### 3. Multiprocessing - True Parallelism

```
Multiprocessing
└── N processes
    └── N Python interpreters
        └── N GILs
            └── real CPU parallelism
```

**Best for:** CPU-bound work (heavy calculations, data processing, image manipulation)

**How it works:** Separate processes with separate interpreters and separate GILs. True parallel execution. Higher overhead due to inter-process communication.

## Understanding AsyncIO and Await

### The async/await Syntax

**`async def`** - Defines a function that can pause and resume
```python
async def fetch_data():
    # This is a coroutine function
    # Calling it returns a coroutine object, doesn't run it yet
    return "data"
```

**`await`** - The yield point where execution pauses
```python
result = await fetch_data()
# 1. Waits for fetch_data() to complete
# 2. Yields control to event loop while waiting
# 3. Other tasks can run during this time
# 4. When done, resumes and assigns result
```

**Key insight:** `await` does two things:
1. "I need this result before continuing"
2. "Let other tasks run while I wait"

### How the Event Loop Works

When you `await` something, this happens under the hood:

1. **Register callback** - Tell the event loop "notify me when this completes"
2. **Yield control** - Suspend current task, run other tasks
3. **Event triggers** - I/O completes, timer expires, thread finishes
4. **Resume task** - Event loop gives you the result and continues execution

For network I/O:
```python
response = await httpx.get('https://api.example.com/data')
```
- Event loop tells OS: "notify me when this socket has data"
- OS uses epoll/kqueue/IOCP to monitor socket
- Other tasks run while waiting
- When data arrives, OS notifies event loop
- Your task resumes with the response

### Async vs Blocking: A Critical Example

**Blocking (BAD in async code):**
```python
async def bad_sleeper():
    time.sleep(5)  # BLOCKS THE ENTIRE EVENT LOOP
    return "done"
```
The thread is frozen for 5 seconds. No other tasks can run. Everything stops.

**Non-blocking (GOOD):**
```python
async def good_sleeper():
    await asyncio.sleep(5)  # Yields control to event loop
    return "done"
```
Schedules a timer, immediately returns control. Other tasks run during the 5 seconds.

### The Threading Escape Hatch

What if you must use a blocking library in async code?

```python
async def fetch_with_requests():
    # requests.get() is blocking - would freeze event loop
    # Solution: run it in a thread pool
    result = await asyncio.to_thread(requests.get, 'https://api.example.com')
    return result
```

`asyncio.to_thread()` runs the blocking call in a separate thread, then awaits its completion without blocking the event loop.

## When to Use Which Approach
- **AsyncIO**: "Don't block; yield while waiting"
- **Threading**: "If you must block, block somewhere else"
- **Multiprocessing**: "Bypass the GIL entirely for CPU work"

## Running Async Code

Calling `async def` functions from sync code doesn't work the way you'd expect:

```python
# This doesn't run the function - it returns a coroutine object
result = some_async_function()  # <coroutine object>

# From sync/top-level code (entry point), use asyncio.run()
import asyncio
result = asyncio.run(some_async_function())

# From inside async code, use await
result = await some_async_function()
```

### Running Multiple Tasks Concurrently

**`asyncio.gather()`** - Run multiple coroutines concurrently and collect results:

```python
async def main():
    results = await asyncio.gather(
        fetch_weather('SF'),
        fetch_weather('NYC'),
        fetch_weather('Tokyo')
    )
    # results = [sf_weather, nyc_weather, tokyo_weather]
```

All three fetches run concurrently on the same thread, yielding to each other at await points.

## Example: Concurrent API Calls

Let's see the difference in practice:

**Sequential (slow):**
```python
async def sequential_fetch():
    result1 = await fetch_api('endpoint1')  # 2 seconds
    result2 = await fetch_api('endpoint2')  # 2 seconds
    result3 = await fetch_api('endpoint3')  # 2 seconds
    return [result1, result2, result3]
    # Total time: ~6 seconds
```

**Concurrent (fast):**
```python
async def concurrent_fetch():
    results = await asyncio.gather(
        fetch_api('endpoint1'),
        fetch_api('endpoint2'),
        fetch_api('endpoint3')
    )
    return results
    # Total time: ~2 seconds (limited by slowest call)
```

**Mixing async and blocking:**
```python
async def mixed_calls():
    results = await asyncio.gather(
        fetch_api_async('endpoint1'),              # async library
        asyncio.to_thread(fetch_api_blocking, 'endpoint2'),  # blocking library
        fetch_api_async('endpoint3')               # async library
    )
    return results
```

## Applying to Our Agent Code

Now let's upgrade our agent from the previous post to handle concurrent tool calls. The key changes:

1. Make the agent's `run()` method async
2. Add a `call_tool()` helper that handles both async and sync tools
3. Parse tool calls as a list (single call becomes a list of one)
4. Use `asyncio.gather()` to execute all tools concurrently
5. Collect results and feed them back to the model

Here's the full implementation:

```python
import os
import json
import asyncio
import inspect
from anthropic import Anthropic
import tool_funcs

class Agent:
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

            If you need to call multiple tools, return a list of the JSON dictionary calls.
            """

    async def call_tool(self, func_name, params):
        """Execute a single tool call, handling both async and sync functions."""
        if hasattr(tool_funcs, func_name):
            try:
                func = getattr(tool_funcs, func_name)

                # Check if function is async
                if inspect.iscoroutinefunction(func):
                    result = await func(**params)
                else:
                    # Run sync function in threadpool
                    result = await asyncio.to_thread(func, **params)

                print(f'     Tool result ({func_name}): {result}')
                return {
                    'tool': func_name,
                    'result': result
                }
            except Exception as e:
                error_msg = f'Error calling {func_name}: {e}'
                print(f'     {error_msg}')
                return {
                    'tool': func_name,
                    'error': str(e)
                }
        else:
            error_msg = f'Unknown tool: {func_name}'
            print(f'     {error_msg}')
            return {
                'tool': func_name,
                'error': error_msg
            }

    async def run(self):
        print(self.system_prompt)
        while True:
            user_message = input("User: ")
            self.history.append({
                "role": "user",
                "content": user_message,
            })

            while True:
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

                if model_response_message.content[0].text.startswith("```"):
                    try:
                        lines = model_response_message.content[0].text.splitlines()
                        if lines[0].startswith("```"):
                            lines = lines[1:]
                        if lines and lines[-1].startswith("```"):
                            lines = lines[:-1]
                        text = "\n".join(lines).strip()
                        decoded_text = json.loads(text)
                        print('     Agentic tool call: ', decoded_text)

                        # Handle both single dict and list of dicts
                        tool_calls = decoded_text if isinstance(decoded_text, list) else [decoded_text]

                        # Execute all tool calls in parallel
                        tasks = [
                            self.call_tool(call['tool'], call.get('inputs', {}))
                            for call in tool_calls
                        ]
                        results = await asyncio.gather(*tasks)

                        # Format results for the model
                        results_text = '\n'.join([
                            f"Tool '{r['tool']}' returned: {r.get('result', r.get('error'))}"
                            for r in results
                        ])

                        self.history.append({
                            "role": "user",
                            "content": results_text,
                        })
                        print(self.history)
                    except json.JSONDecodeError as e:
                        print("JSON decode error:", e)
                        return None
                else:
                    print('Agent: ', model_response_message.content[0].text)
                    break
```

### Key Changes Explained

**1. Async method signature:**
```python
async def run(self):  # Now async
```

**2. Smart tool execution:**
```python
if inspect.iscoroutinefunction(func):
    result = await func(**params)  # Native async tool
else:
    result = await asyncio.to_thread(func, **params)  # Blocking tool -> threadpool
```

This handles both async and sync tools automatically. If `tool_funcs.get_weather()` is async, we await it directly. If it's a blocking function (uses `requests` library), we run it in a thread pool.

**3. Concurrent execution:**
```python
tool_calls = decoded_text if isinstance(decoded_text, list) else [decoded_text]

tasks = [
    self.call_tool(call['tool'], call.get('inputs', {}))
    for call in tool_calls
]
results = await asyncio.gather(*tasks)
```

Whether the model returns one tool call or ten, we execute them all concurrently. `asyncio.gather()` waits for all to complete and preserves order.

**4. Entry point:**
```python
if __name__ == "__main__":
    agent = Agent()
    asyncio.run(agent.run())  # Launch async code from sync context
```

### Example Output

```
User: What's the weather in San Francisco, New York, and Tokyo?

[Agent reasoning...]
     Agentic tool call: [
       {'tool': 'get_weather', 'inputs': {'location': 'San Francisco'}},
       {'tool': 'get_weather', 'inputs': {'location': 'New York'}},
       {'tool': 'get_weather', 'inputs': {'location': 'Tokyo'}}
     ]

[All three API calls execute concurrently]
     Tool result (get_weather): {"temp": "68°F", "conditions": "Sunny"}
     Tool result (get_weather): {"temp": "45°F", "conditions": "Cloudy"}
     Tool result (get_weather): {"temp": "72°F", "conditions": "Clear"}

[Agent synthesizes results...]
Agent: Here's the weather:
- San Francisco: Sunny, 68°F
- New York: Cloudy, 45°F
- Tokyo: Clear, 72°F
```

Instead of 3 sequential API calls (3× delay), all three run concurrently (1× delay).

## Things Not Covered

- Exception handling strategies (fail-fast vs collect all)
- Timeout and cancellation
- Rate limiting and backoff
- Streaming tool results back to the model incrementally
- Tool call prioritization (critical vs optional)
- Retry logic with exponential backoff
- Circuit breakers for failing tools
- Structured logging and tracing for concurrent execution
- Async context managers for resource cleanup

These are topics for future posts on production-ready agent architectures.
