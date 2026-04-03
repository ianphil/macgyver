# pi-agent-core — Product Spec

*Reverse-engineered by MacGyver · badlogic/pi-mono → packages/agent*

---

## Problem Statement

Having a unified LLM API (that's what pi-ai does) gets you halfway. You can send messages to any model and stream responses back. But there's a gap between "I can talk to an LLM" and "I have an agent that does things."

That gap is the loop. When a model says "I want to call the `read_file` tool with these arguments," something needs to find that tool, validate the arguments, execute it, take the result, feed it back to the model, and keep going until the model is done. And while that's happening, a user might type "actually, stop — do this instead." And the UI needs to show what's happening in real time. And the tool might stream progress updates. And tokens might expire mid-execution. And the model might call five tools at once.

pi-agent-core is the orchestration layer that handles all of that. It takes pi-ai's streaming primitives and wraps them in a stateful agent with tool execution, lifecycle events, message queuing, and abort handling. It's the runtime that turns "conversation with an LLM" into "agent that gets things done."

---

## Actors & Goals

**The App Developer** — the primary user. Someone building an AI-powered application (coding agent, chatbot, workflow tool) who needs a ready-made agent runtime. They want to define tools, hand the agent a prompt, subscribe to events for UI updates, and have the tool-call loop handled for them. They don't want to write the loop themselves.

**The UI** — whatever is rendering the agent's activity. It subscribes to events and gets a real-time feed: when the agent starts, when text streams in, when tools execute, when things finish. It needs enough granularity to show partial assistant responses, tool execution progress, and error states.

**The Tool Author** — someone defining tools the agent can use. They want a clean interface: schema for parameters, an execute function that gets validated args, and a way to stream progress. Errors should be thrown, not encoded. The framework handles the rest.

**The Operator** — configures the agent (model, thinking level, session ID, API keys) and controls its behavior (tool execution mode, before/after hooks, steering). They can swap models, adjust thinking, and block dangerous tool calls without touching tool code.

---

## Operator Value

**You don't write the agent loop.** The entire cycle — stream response, extract tool calls, validate arguments, execute tools (sequentially or in parallel), feed results back, repeat until done — is handled. You define tools and subscribe to events.

**Real-time UI is built in.** The event system emits 10 distinct event types covering the full lifecycle: agent start/end, turn start/end, message start/update/end, tool execution start/update/end. You get streaming text deltas, partial tool results, and structured completion data — everything a responsive UI needs.

**Mid-run control.** Steering messages let you interrupt the agent between turns while tools are still running. Follow-up messages queue work for after the agent would normally stop. Both have configurable drain modes (one-at-a-time or all-at-once).

**Tool safety hooks.** `beforeToolCall` runs after argument validation but before execution — you can block dangerous calls with a reason. `afterToolCall` lets you modify results, redact content, or flip error flags before they reach the model.

**It's a layer, not a framework.** The package has one dependency (pi-ai) and five source files. It doesn't dictate your message types, your persistence strategy, or your transport. Custom message types are added via TypeScript declaration merging. The stream function is swappable — point it at a proxy server and the same agent works in a browser.

---

## Core Capabilities

### The Agent Loop

The heart of the package. Two nested loops:

**Inner loop** — handles tool calls and steering. After each assistant response, if there are tool calls, execute them and go again. If steering messages arrived during tool execution, inject them before the next turn.

**Outer loop** — handles follow-ups. When the inner loop finishes (no more tool calls, no steering), check for follow-up messages. If any exist, inject them and restart the inner loop. If none, the agent stops.

The loop terminates on: no more tool calls + no steering + no follow-ups, or error, or abort.

### Tool Execution

Tools are `AgentTool` objects — they extend pi-ai's `Tool` (name, description, schema) with execution logic:

- **`label`**: Human-readable name for UI display
- **`prepareArguments`**: Optional shim that transforms raw LLM arguments before validation (for backward compatibility or format normalization)
- **`execute`**: The actual function. Receives validated params, an abort signal, and an `onUpdate` callback for streaming progress. Returns `{ content, details }`. Throws on failure — the framework catches it and reports to the model.

Two execution modes:
- **`parallel`** (default): Preflight all tool calls sequentially (validation + `beforeToolCall`), then execute allowed tools concurrently. Final results are emitted in the order the assistant requested them.
- **`sequential`**: Each tool call is fully prepared, executed, and finalized before the next one starts.

The preflight-then-execute split in parallel mode is deliberate: `beforeToolCall` might need to see context or prompt the user for confirmation, so it runs sequentially. But actual tool execution — reading files, running commands, making API calls — runs concurrently.

### Before/After Tool Hooks

**`beforeToolCall`** fires after arguments are validated, before execution. It receives the assistant message, tool call, validated args, and current context. Return `{ block: true, reason: "..." }` to prevent execution — the model gets an error result with your reason. This is your safety gate.

**`afterToolCall`** fires after execution, before final events are emitted. It can override the result's content, details, or error flag. Useful for redacting sensitive output, adding audit metadata, or flipping error flags. Field-level override — omitted fields keep original values, no deep merge.

Both hooks receive the abort signal and are responsible for honoring it.

### Event System

Ten event types covering three layers:

**Agent lifecycle**: `agent_start`, `agent_end` — brackets the entire run. `agent_end` carries all new messages produced during the run.

**Turn lifecycle**: `turn_start`, `turn_end` — brackets one assistant response + its tool executions. `turn_end` carries the assistant message and its tool results.

**Message lifecycle**: `message_start`, `message_update`, `message_end` — covers user messages, assistant messages, and tool results. `message_update` is assistant-only and carries the raw streaming event from pi-ai (text deltas, thinking deltas, tool call deltas).

**Tool lifecycle**: `tool_execution_start`, `tool_execution_update`, `tool_execution_end` — covers individual tool calls. `tool_execution_update` carries partial results from tools that stream progress.

Listeners are awaited in registration order. `agent_end` means no more loop events, but the run isn't settled until all `agent_end` listeners finish. This matters: `waitForIdle()` and the `prompt()` promise don't resolve until listener work completes. Your `agent_end` listener can flush state, save transcripts, or do cleanup as part of the run.

### AgentMessage and Custom Types

The agent doesn't work with raw LLM messages internally — it works with `AgentMessage`, which is a union of standard LLM messages (`user`, `assistant`, `toolResult`) plus any custom types you define via TypeScript declaration merging.

This means your transcript can include app-specific messages — notifications, artifacts, status updates — alongside standard LLM messages. The `convertToLlm` function strips them out before each LLM call, so the model only sees what it understands.

The two-phase transform pipeline:
1. **`transformContext`** (optional): Operates on `AgentMessage[]`. Prune old messages, inject external context, manage the context window.
2. **`convertToLlm`** (required): Converts `AgentMessage[]` → LLM `Message[]`. Filter out custom types, transform app messages to LLM format.

Both have a "must not throw" contract — return a safe fallback instead.

### Steering and Follow-Up Queues

Two message queues enable mid-run and post-run injection:

**Steering queue** (`agent.steer(message)`): Messages injected after the current turn's tool calls finish but before the next LLM call. The model sees them as new user input mid-conversation. Use for "stop doing that, do this instead."

**Follow-up queue** (`agent.followUp(message)`): Messages injected only after the agent would otherwise stop (no more tool calls, no steering). Use for "also do this when you're done."

Both queues have a drain mode:
- **`one-at-a-time`** (default): Drain one message per check. The agent processes it, then checks again.
- **`all`**: Drain all queued messages at once.

Queues can be cleared independently or together.

### The Agent Class (Stateful Wrapper)

`Agent` is the high-level API. It wraps the low-level loop with:

- **State management**: System prompt, model, thinking level, tools, messages, streaming state, pending tool calls, error messages. Tools and messages arrays are defensively copied on assignment.
- **Lifecycle**: `prompt()` starts a run, `continue()` resumes from existing context, `abort()` cancels, `waitForIdle()` waits for settlement, `reset()` clears everything.
- **Single-run enforcement**: Calling `prompt()` while a run is active throws. Use `steer()` or `followUp()` to queue messages during a run.
- **Event subscription**: `subscribe()` returns an unsubscribe function. Listeners receive both the event and the current abort signal.

### The Low-Level API

`agentLoop()` and `agentLoopContinue()` are the raw loop functions — they return an `EventStream<AgentEvent>` that you iterate directly. No state management, no queuing, no single-run enforcement. Use these when you want full control.

Important difference from `Agent`: the low-level streams are observational. They don't wait for your async event handling before proceeding to the next phase. If you need `message_end` processing to complete before tool preflight starts, use the `Agent` class.

### Proxy Stream Function

`streamProxy()` is a drop-in replacement for pi-ai's `streamSimple` that routes LLM calls through an HTTP server instead of calling providers directly. The server handles auth and proxies to providers.

The proxy protocol is bandwidth-optimized: it strips the `partial` field from delta events (which contains the full accumulated message) and the client reconstructs it locally. The client sends model, context, and options as JSON; the server returns SSE events.

Use this as the `streamFn` option to run the same agent in a browser, with the server managing API keys and provider access.

### Dynamic API Key Resolution

`getApiKey` is called before every LLM request with the provider name. It returns a fresh API key — critical for short-lived OAuth tokens (GitHub Copilot, Anthropic OAuth) that might expire during long tool execution phases. If your tool runs for 10 minutes, the token that was valid when the turn started might not be valid when the next LLM call happens.

---

## Observable Behaviors

**Prompting the agent:** Call `agent.prompt("fix the bug")`. Events fire: `agent_start` → `turn_start` → user `message_start/end` → assistant `message_start` → streaming `message_update`s → assistant `message_end` → if tool calls: `tool_execution_start/update/end` → tool result `message_start/end` → `turn_end` → loop continues if more tool calls. Ends with `agent_end`. Promise resolves after all `agent_end` listeners settle.

**Steering mid-run:** Call `agent.steer(message)` while the agent is executing tools. After the current turn's tool calls finish, the steering message is injected before the next LLM call. The model sees it as new context.

**Following up:** Call `agent.followUp(message)` at any time. After the agent would stop (no tool calls, no steering), the follow-up is injected and another turn runs.

**Aborting:** Call `agent.abort()`. The current stream terminates with `stopReason: "aborted"`. Tool calls in flight receive the abort signal. The agent emits `turn_end` → `agent_end` with the partial results.

**Continuing after error:** Call `agent.continue()`. The loop restarts from the current context (last message must be `user` or `toolResult`). Useful for retrying after transient errors without re-prompting.

**Tool validation failure:** If the LLM produces arguments that don't match the schema, the framework catches the validation error and returns it to the model as a tool result with `isError: true`. The model gets a chance to retry.

**Tool not found:** If the LLM calls a tool that isn't in the current tools list, an error result is returned: "Tool X not found."

**Tool blocked by beforeToolCall:** The framework emits `tool_execution_start`, runs the hook, gets `{ block: true, reason }`, emits an error tool result with the reason, and continues. The model sees the block reason and can adjust.

---

## Edge Cases & Failure Behavior

| What happens | What the system does |
|---|---|
| `prompt()` called while already running | Throws: "Agent is already processing a prompt" |
| `continue()` with no messages | Throws: "No messages to continue from" |
| `continue()` when last message is `assistant` | Drains steering/follow-up queues first; throws if none available |
| Tool throws during execution | Caught, returned as `isError: true` tool result with error message |
| `beforeToolCall` blocks execution | Error tool result with custom reason (or default "Tool execution was blocked") |
| `convertToLlm` encounters custom message type | Must filter/convert; the contract says "must not throw" |
| `transformContext` fails | Contract says "must not throw" — return safe fallback |
| LLM returns error/aborted stopReason | Loop terminates immediately, emits `turn_end` + `agent_end` |
| OAuth token expires during tool execution | `getApiKey` is called fresh before every LLM request |
| Multiple tool calls from one response (parallel mode) | Preflight sequentially, execute concurrently, emit results in source order |
| Steering arrives but no active run | Queued; drained when next turn completes |
| Both steering and follow-up queued | Steering is checked first (after each turn); follow-up only when agent would stop |
| Abort signal fired during tool execution | Signal passed to tool's execute function; tool is responsible for honoring it |
| Low-level `agentLoop` used instead of `Agent` | No barrier between message processing and tool preflight; async listeners may race |

---

## Non-Functional Constraints

- **Single dependency**: Only depends on `@mariozechner/pi-ai`. No HTTP frameworks, no state libraries, no UI dependencies.
- **Five source files**: `agent.ts`, `agent-loop.ts`, `proxy.ts`, `types.ts`, `index.ts`. That's the whole package.
- **Node.js ≥ 20.0.0** required.
- **Browser compatible**: Core agent works in browsers. Use `streamProxy` to route LLM calls through a server.
- **Stateful but not persistent**: `Agent` holds state in memory. Persistence is the app's responsibility — save `state.messages`, restore via `initialState`.
- **Transport-agnostic**: The `streamFn` is swappable. Direct provider calls, proxy server, WebSocket — whatever returns the right event stream.
- **No concurrency**: One active run per agent instance. Parallel tool execution happens within a single run.

---

## Non-Goals

**Not a tool library.** The package defines the `AgentTool` interface and the execution framework, but ships zero built-in tools. No file reading, no command execution, no web search. Tools are the app's responsibility.

**Not a persistence layer.** No database, no file storage, no session management. The transcript lives in `state.messages`; you serialize and store it however you want.

**Not a UI framework.** Events are emitted for UI consumption, but there's no rendering, no components, no terminal output. The package is headless.

**Not a multi-agent system.** One agent, one loop, one transcript. No agent-to-agent communication, no orchestration of multiple agents, no delegation.

**Not an LLM client.** It doesn't talk to providers directly — that's pi-ai's job. The agent calls `streamSimple` (or your custom `streamFn`) and consumes the event stream. It has no knowledge of Anthropic, OpenAI, or any specific provider.

**Not a prompt engineering framework.** No built-in system prompts, no chain-of-thought templates, no RAG integration. The system prompt is a string you provide.

---

## How This Connects to pi-ai

This is the other half of the story. pi-ai is the conversation and tool declaration layer — it tells the model what tools exist, streams responses, tracks costs, and handles cross-provider translation. pi-agent-core is the orchestration and execution layer — it runs the loop, executes tools, manages state, and emits events.

The seam between them is clean:

- **pi-ai's `Tool`** = `{ name, description, parameters }` — data only, no execution
- **pi-agent-core's `AgentTool`** = pi-ai's `Tool` + `{ label, execute, prepareArguments }` — adds execution

When the agent sends context to the LLM, `AgentTool[]` is passed directly as pi-ai's `Context.tools`. pi-ai only sees the schema fields. The `execute` function is invisible to it.

When the model returns tool calls, the agent finds the matching `AgentTool` by name, validates arguments using pi-ai's `validateToolArguments`, and calls `execute`. The result goes back as a `ToolResultMessage` (a pi-ai type) into the context.

pi-ai never executes anything. The agent never talks to providers. Each package does one thing well.

---

## What's Interesting Under the Hood

The parallel tool execution architecture is more nuanced than "run them all at once." It splits into two phases: sequential preflight (validation + `beforeToolCall`) and concurrent execution. This means your safety hook can see the full context and even prompt a user for confirmation — sequentially, predictably — and then actual execution fans out. Results are still emitted in the order the assistant requested them, not in completion order. That's important for deterministic UIs.

The `Agent` class treats `message_end` processing as a barrier before tool preflight begins. The low-level `agentLoop` does not — it's observational. This is documented clearly and it's the right call: the `Agent` class is for apps that need ordering guarantees (most apps), and the raw loop is for apps that need maximum throughput and can handle their own synchronization.

The custom message type system via declaration merging is elegant. You add `notification` or `artifact` or whatever to `CustomAgentMessages`, and suddenly `AgentMessage` includes your type — with full type safety. The `convertToLlm` function filters them out before the LLM call. Your transcript can carry app-specific data without polluting the model's view of the conversation.

The proxy stream function strips the `partial` field from delta events on the wire. That field contains the entire accumulated message at every delta — which means for a long response, you're sending the same text over and over, growing linearly. Stripping it and reconstructing client-side is a significant bandwidth optimization for browser-based agents.

The "must not throw" contracts on `convertToLlm` and `transformContext` are worth noting. These run inside the loop, and an uncaught throw would break the event sequence — no `agent_end`, no cleanup. The contract pushes error handling to the implementer, which is the right boundary for a runtime that doesn't know what your custom messages look like.

---

*Spec written by MacGyver — practical, hands-on, finds something interesting in every codebase.*
