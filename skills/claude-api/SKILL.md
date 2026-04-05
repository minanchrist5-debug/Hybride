# Building LLM-Powered Applications with Claude

This skill helps you build LLM-powered applications with Claude. Choose the right surface based on your needs, detect the project language, then read the relevant language-specific documentation.

## Trigger Conditions

**USE this skill when:**
- Code imports `anthropic`, `@anthropic-ai/sdk`, or `claude_agent_sdk`
- User asks to use Claude API, Anthropic SDKs, or Agent SDK
- User asks how to call Claude programmatically
- User wants to build a chatbot, agent, or LLM-powered feature using Claude

**DO NOT USE this skill when:**
- Code imports `openai` or another AI SDK (not Anthropic)
- User is asking about Claude Code (the CLI tool) itself — use the claude-code-guide skill instead
- User wants general information about AI/ML concepts without implementation

## Defaults

Unless the user requests otherwise:

For the Claude model version, please use Claude Opus 4.6, which you can access via the exact model string `claude-opus-4-6`. Please default to using adaptive thinking (`thinking: {type: "adaptive"}`) for anything remotely complicated. And finally, please default to streaming for any request that may involve long input, long output, or high `max_tokens` — it prevents hitting request timeouts. Use the SDK's `.get_final_message()` / `.finalMessage()` helper to get the complete response if you don't need to handle individual stream events

---

## Language Detection

Before reading code examples, determine which language the user is working in:

1. **Look at project files** to infer the language:

   - `*.py`, `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile` → **Python** — read from `python/`
   - `*.ts`, `*.tsx`, `package.json`, `tsconfig.json` → **TypeScript** — read from `typescript/`
   - `*.js`, `*.jsx` (no `.ts` files present) → **TypeScript** — JS uses the same SDK, read from `typescript/`
   - `*.java`, `pom.xml`, `build.gradle` → **Java** — read from `java/`
   - `*.kt`, `*.kts`, `build.gradle.kts` → **Java** — Kotlin uses the Java SDK, read from `java/`
   - `*.scala`, `build.sbt` → **Java** — Scala uses the Java SDK, read from `java/`
   - `*.go`, `go.mod` → **Go** — read from `go/`
   - `*.rb`, `Gemfile` → **Ruby** — read from `ruby/`
   - `*.cs`, `*.csproj` → **C#** — read from `csharp/`
   - `*.php`, `composer.json` → **PHP** — read from `php/`

2. **If multiple languages detected** (e.g., both Python and TypeScript files):

   - Check which language the user's current file or question relates to
   - If still ambiguous, ask: "I detected both Python and TypeScript files. Which language are you using for the Claude API integration?"

3. **If language can't be inferred** (empty project, no source files, or unsupported language):

   - Use AskUserQuestion with options: Python, TypeScript, Java, Go, Ruby, cURL/raw HTTP, C#, PHP
   - If AskUserQuestion is unavailable, default to Python examples and note: "Showing Python examples. Let me know if you need a different language."

4. **If unsupported language detected** (Rust, Swift, C++, Elixir, etc.):

   - Suggest cURL/raw HTTP examples from `curl/` and note that community SDKs may exist
   - Offer to show Python or TypeScript examples as reference implementations

5. **If user needs cURL/raw HTTP examples**, read from `curl/`.

### Language-Specific Feature Support

| Language   | Tool Runner | Agent SDK | Notes                                 |
| ---------- | ----------- | --------- | ------------------------------------- |
| Python     | Yes (beta)  | Yes       | Full support — `@beta_tool` decorator |
| TypeScript | Yes (beta)  | Yes       | Full support — `betaZodTool` + Zod    |
| Java       | Yes (beta)  | No        | Beta tool use with annotated classes  |
| Go         | Yes (beta)  | No        | `BetaToolRunner` in `toolrunner` pkg  |
| Ruby       | Yes (beta)  | No        | `BaseTool` + `tool_runner` in beta    |
| cURL       | N/A         | N/A       | Raw HTTP, no SDK features             |
| C#         | No          | No        | Official SDK                          |
| PHP        | Yes (beta)  | No        | `BetaRunnableTool` + `toolRunner()`   |

---

## Which Surface Should I Use?

> **Start simple.** Default to the simplest tier that meets your needs. Single API calls and workflows handle most use cases — only reach for agents when the task genuinely requires open-ended, model-driven exploration.

| Use Case                                        | Tier            | Recommended Surface       | Why                                     |
| ----------------------------------------------- | --------------- | ------------------------- | --------------------------------------- |
| Classification, summarization, extraction, Q&A  | Single LLM call | **Claude API**            | One request, one response               |
| Batch processing or embeddings                  | Single LLM call | **Claude API**            | Specialized endpoints                   |
| Multi-step pipelines with code-controlled logic | Workflow        | **Claude API + tool use** | You orchestrate the loop                |
| Custom agent with your own tools                | Agent           | **Claude API + tool use** | Maximum flexibility                     |
| AI agent with file/web/terminal access          | Agent           | **Agent SDK**             | Built-in tools, safety, and MCP support |
| Agentic coding assistant                        | Agent           | **Agent SDK**             | Designed for this use case              |
| Want built-in permissions and guardrails        | Agent           | **Agent SDK**             | Safety features included                |

> **Note:** The Agent SDK is for when you want built-in file/web/terminal tools, permissions, and MCP out of the box. If you want to build an agent with your own tools, Claude API is the right choice — use the tool runner for automatic loop handling, or the manual loop for fine-grained control (approval gates, custom logging, conditional execution).

### Decision Tree

```
What does your application need?

1. Single LLM call (classification, summarization, extraction, Q&A)
   └── Claude API — one request, one response

2. Does Claude need to read/write files, browse the web, or run shell commands
   as part of its work? (Not: does your app read a file and hand it to Claude —
   does Claude itself need to discover and access files/web/shell?)
   └── Yes → Agent SDK — built-in tools, don't reimplement them
       Examples: "scan a codebase for bugs", "summarize every file in a directory",
                 "find bugs using subagents", "research a topic via web search"

3. Workflow (multi-step, code-orchestrated, with your own tools)
   └── Claude API with tool use — you control the loop

4. Open-ended agent (model decides its own trajectory, your own tools)
   └── Claude API agentic loop (maximum flexibility)
```

### Should I Build an Agent?

Before choosing the agent tier, check all four criteria:

- **Complexity** — Is the task multi-step and hard to fully specify in advance?
- **Value** — Does the outcome justify higher cost and latency?
- **Viability** — Is Claude capable at this task type?
- **Cost of error** — Can errors be caught and recovered from?

If the answer is "no" to any of these, stay at a simpler tier.

---

## Architecture

Everything goes through `POST /v1/messages`. Tools and output constraints are features of this single endpoint — not separate APIs.

**User-defined tools** — You define tools (via decorators, Zod schemas, or raw JSON), and the SDK's tool runner handles calling the API, executing your functions, and looping until Claude is done.

**Server-side tools** — Anthropic-hosted tools that run on Anthropic's infrastructure. Code execution is fully server-side. Web search and web fetch retrieve live content.

**Structured outputs** — Constrains the Messages API response format (`output_config.format`). Use `client.messages.parse()` which validates responses against your schema automatically.

**Supporting endpoints** — Batches, Files, Token Counting, and Models APIs feed into or support Messages API requests.

---

## Current Models

| Model             | Model ID            | Context        | Input $/1M | Output $/1M |
| ----------------- | ------------------- | -------------- | ---------- | ----------- |
| Claude Opus 4.6   | `claude-opus-4-6`   | 200K (1M beta) | $5.00      | $25.00      |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 200K (1M beta) | $3.00      | $15.00      |
| Claude Haiku 4.5  | `claude-haiku-4-5`  | 200K           | $1.00      | $5.00       |

**ALWAYS use `claude-opus-4-6` unless the user explicitly names a different model.**

---

## Thinking & Effort

**Opus 4.6 — Adaptive thinking (recommended):** Use `thinking: {type: "adaptive"}`. Claude dynamically decides when and how much to think. No `budget_tokens` needed — `budget_tokens` is deprecated on Opus 4.6 and Sonnet 4.6.

**Effort parameter:** Controls thinking depth via `output_config: {effort: "low"|"medium"|"high"|"max"}`. Default is `high`. `max` is Opus 4.6 only.

---

## Quick Examples by Language

### Python

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=16000,
    thinking={"type": "adaptive"},
    messages=[{"role": "user", "content": "Explain quantum entanglement"}]
)

for block in response.content:
    if block.type == "text":
        print(block.text)
```

### TypeScript

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 16000,
  thinking: { type: "adaptive" },
  messages: [{ role: "user", content: "Explain quantum entanglement" }],
});

for (const block of response.content) {
  if (block.type === "text") console.log(block.text);
}
```

### Streaming (Python)

```python
with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=64000,
    messages=[{"role": "user", "content": "Write a story"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final = stream.get_final_message()
```

### Tool Use — Tool Runner (Python)

```python
from anthropic import beta_tool

@beta_tool
def get_weather(location: str) -> str:
    """Get current weather for a location.

    Args:
        location: City and state, e.g., San Francisco, CA.
    """
    return f"72°F and sunny in {location}"

runner = client.beta.messages.tool_runner(
    model="claude-opus-4-6",
    max_tokens=16000,
    tools=[get_weather],
    messages=[{"role": "user", "content": "What's the weather in Paris?"}],
)

for message in runner:
    print(message)
```

### Agent SDK (Python)

```python
import anyio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def main():
    async for message in query(
        prompt="Explain this codebase",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Glob", "Grep"])
    ):
        if isinstance(message, ResultMessage):
            print(message.result)

anyio.run(main)
```

---

## Common Pitfalls

- **Adaptive thinking on Opus 4.6/Sonnet 4.6:** Use `thinking: {type: "adaptive"}` — do NOT use `budget_tokens` (deprecated).
- **`max_tokens` defaults:** Default to `~16000` for non-streaming, `~64000` for streaming.
- **128K output tokens:** Opus 4.6 supports up to 128K `max_tokens`, but requires streaming.
- **Structured outputs:** Use `output_config: {format: {...}}` — `output_format` is deprecated.
- **Tool call JSON parsing:** Always parse tool inputs with `json.loads()` / `JSON.parse()`.
- **SDK types:** Use `Anthropic.MessageParam`, `Anthropic.Tool`, etc. — don't redefine equivalent interfaces.
- **Don't truncate inputs:** If content is too long, notify the user and discuss options rather than silently truncating.

---

## Installation

### Python
```bash
pip install anthropic          # Claude API
pip install claude-agent-sdk   # Agent SDK
```

### TypeScript
```bash
npm install @anthropic-ai/sdk                    # Claude API
npm install @anthropic-ai/claude-agent-sdk       # Agent SDK
```

### Java (Maven)
```xml
<dependency>
    <groupId>com.anthropic</groupId>
    <artifactId>anthropic-java</artifactId>
    <version>2.17.0</version>
</dependency>
```

### Go
```bash
go get github.com/anthropics/anthropic-sdk-go
```

### Ruby
```bash
gem install anthropic
```

### PHP
```bash
composer require "anthropic-ai/sdk"
```

### C#
```bash
dotnet add package Anthropic
```

---

## Live Documentation

For the latest API docs, use WebFetch with these URLs:

- Models: `https://platform.claude.com/docs/en/about-claude/models/overview.md`
- Tool Use: `https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview.md`
- Streaming: `https://platform.claude.com/docs/en/build-with-claude/streaming.md`
- Prompt Caching: `https://platform.claude.com/docs/en/build-with-claude/prompt-caching.md`
- Structured Outputs: `https://platform.claude.com/docs/en/build-with-claude/structured-outputs.md`
- Agent SDK: `https://platform.claude.com/docs/en/agent-sdk.md`
