# pi-ai — Product Spec

*Reverse-engineered by MacGyver · badlogic/pi-mono → packages/ai*

---

## Problem Statement

Here's the thing about building on top of large language models: every provider speaks a different dialect. OpenAI has one message format, Anthropic has another, Google does its own thing, and Bedrock is off in AWS-land with a completely different wire protocol. That's before you get to the fun stuff — thinking blocks that only exist on some providers, tool call IDs that are 450 characters long on one API and max 64 on another, error messages for "your context is too big" that come in twenty different phrasings.

If you're building an AI-powered application — say, a coding agent — you don't want to write and maintain a separate client for each of these. You want to say "send this conversation to this model, stream the response back, tell me what it costs" and have that work the same way whether it's Claude, GPT-5, Gemini, or a local Ollama server.

That's what pi-ai does. It's a unified LLM API — one streaming interface, one message format, one tool system, one cost tracker — that sits in front of 23+ providers and 800+ models. You pick a model, hand it a conversation, and get back a provider-neutral stream of events. The provider differences happen behind the curtain.

---

## Actors & Goals

**The Integrator** — the primary user. A developer embedding LLM capabilities into an application (most likely an AI coding agent, chatbot, or workflow automation). They want one programmatic interface that works with any model, full streaming with incremental events, tool calling with validation, thinking/reasoning support, cost tracking, and the ability to switch providers mid-conversation without losing context.

**The Operator** — the person who configures which providers are available, manages API keys and OAuth credentials, and controls caching behavior. They configure environment variables, run OAuth login flows, and query the model registry. Their day-to-day interaction is mostly "set it and forget it" until they need to add a new provider or rotate keys.

**The Test Author** — a developer who needs to write automated tests for LLM-powered features without making real API calls or spending real money. They want a mock provider that behaves exactly like the real thing — streaming, tool calls, thinking blocks, abort handling, usage estimation — but runs entirely in-process with scripted responses.

---

## Operator Value

**One dependency replaces a dozen SDKs.** Instead of importing the OpenAI SDK, the Anthropic SDK, the Google GenAI SDK, the AWS Bedrock SDK, and the Mistral SDK — and learning each one's quirks — you import pi-ai and get all of them through a single interface.

**Provider portability is real, not theoretical.** You can start a conversation with Claude, switch to GPT-5 for the next turn, then hand it to Gemini — and it just works. The library automatically transforms thinking blocks, normalizes tool call IDs, drops provider-specific opaque payloads, and injects synthetic tool results for orphaned calls. The conversation flows across providers without the integrator touching any of that.

**Cost tracking is baked in.** Every response comes back with a full token usage breakdown — input, output, cache read, cache write — and per-token cost calculations based on the model's known pricing. You always know what a request cost.

**Testing is free.** The faux provider simulates the entire streaming protocol — text deltas, thinking blocks, tool calls with incremental JSON, abort handling, prompt cache estimation — using scripted responses. No network, no API keys, no charges. You can write a full integration test suite against it.

**It works in browsers too.** The core streaming and completion functions run in browser environments (with some documented limitations around Bedrock and OAuth). Same API, same types, different runtime.

---

## Core Capabilities

### Unified Streaming Completions

The heart of the library. You call `stream(model, context, options)` and get back an async iterable of events — `start`, `text_delta`, `thinking_delta`, `toolcall_delta`, `done`, `error` — in a defined sequence. Or you call `complete()` and get back the final message directly. Either way, the interface is identical regardless of which provider is behind it.

There are two tiers: `stream`/`complete` expose provider-specific options (Anthropic thinking budget, OpenAI reasoning effort, Google thinking level), while `streamSimple`/`completeSimple` provide a simplified interface that normalizes thinking across all providers into five levels: `minimal`, `low`, `medium`, `high`, `xhigh`.

The stream exposes a `.result()` promise that resolves to the complete `AssistantMessage` after the stream ends — whether it ended successfully, with an error, or via abort. It resolves, never rejects. Errors are data, not exceptions.

### Multi-Provider Model Registry

800+ pre-configured models across 23 providers, auto-generated from external sources. Each model carries its API type, provider, base URL, reasoning capability, input modalities (text, image), per-token cost (input, output, cache read, cache write), context window size, and max output tokens.

`getModel('openai', 'gpt-4o-mini')` gives you a fully typed object with IDE auto-completion for both the provider name and model ID. `getProviders()` and `getModels(provider)` let you discover what's available. It's a proper registry, not a pile of constants.

The 23 providers: OpenAI, Azure OpenAI, OpenAI Codex, Anthropic, Google, Google Vertex AI, Google Gemini CLI, Google Antigravity, Mistral, Groq, Cerebras, xAI, OpenRouter, Vercel AI Gateway, zAI, MiniMax, MiniMax CN, HuggingFace, GitHub Copilot, Amazon Bedrock, OpenCode Zen, OpenCode Go, Kimi For Coding — plus any OpenAI-compatible endpoint (Ollama, vLLM, LM Studio, etc.) via custom model definition.

### Custom Model Definition

You can define a model for any endpoint that speaks a supported protocol. Create a `Model` object with the API type, base URL, costs (even if zero), and compatibility settings, and hand it to `stream()`. This is how you point at a local Ollama instance, a LiteLLM proxy, or a custom inference server without touching the registry.

The compatibility layer is surprisingly thorough — it handles differences in role naming (`developer` vs `system`), max token field names, reasoning parameter formats (OpenAI, OpenRouter, z.ai, Qwen each do it differently), strict mode support, tool result requirements, and streaming usage reporting. Auto-detected from URL for known providers, overridable per model.

### Tool Calling with Schema Validation

Tools are defined with TypeBox schemas — name, description, parameters. During streaming, tool call arguments are incrementally parsed from partial JSON as they arrive, so you can show "writing to: /path/to/file" in the UI before the full arguments are complete. At minimum, you get `{}` — never undefined.

`validateToolCall()` validates completed arguments against the schema using AJV with type coercion. In environments with strict CSP (browser extensions), validation is gracefully skipped rather than crashing. There's also a `StringEnum` helper that produces `enum`-based schemas instead of `anyOf/const` patterns — because Google's API doesn't understand the latter.

### Thinking and Reasoning

Many models across providers support thinking/reasoning — Claude, GPT-5, Gemini 2.5/3, Grok, GPT-OSS. The library gives you unified control over all of them.

The simplified interface maps five reasoning levels (`minimal` through `xhigh`) to each provider's native mechanism — Anthropic's budget tokens, OpenAI's reasoning effort, Google's thinking level enum. Custom token budgets per level are configurable. The provider-specific interface gives you full control when you need it.

Thinking content streams through its own event channel (`thinking_start`, `thinking_delta`, `thinking_end`), separate from text. Encrypted/redacted thinking payloads — used by some providers for multi-turn continuity — are preserved for same-model replay and dropped for cross-model handoffs. That's a subtle but important detail: you can safely hand a conversation with thinking blocks from one provider to another without worrying about opaque encrypted blobs causing API errors.

### Cross-Provider Conversation Handoffs

This is where it gets interesting. You can switch models mid-conversation — Claude to GPT-5 to Gemini — and the library handles the translation automatically.

What happens behind the scenes: thinking blocks from other providers become plain text. Tool call IDs get normalized (OpenAI Responses generates 450+ char IDs; Anthropic requires max 64 chars matching `[a-zA-Z0-9_-]+`). Provider-specific signatures and opaque payloads get dropped. Errored or aborted assistant messages are stripped entirely — replaying partial turns causes API errors. Orphaned tool calls (assistant requested a tool but no result exists) get synthetic error results injected so the next provider doesn't choke.

User messages and tool results pass through unchanged. The conversation just works.

### Context Overflow Detection

Every provider has its own way of saying "your input is too big." Anthropic says "prompt is too long: X tokens > Y maximum." OpenAI says "exceeds the context window." Google says "input token count exceeds the maximum." Cerebras just returns a 400 with no body.

`isContextOverflow(message)` matches against 20+ known patterns across all supported providers, including generic fallbacks. It also handles silent overflow — providers like z.ai that accept the request anyway — by comparing reported input tokens against the context window when you pass it. And it explicitly excludes false positives from throttling/rate-limit errors that use similar phrasing ("too many tokens" from a throttled Bedrock request ≠ context overflow).

### OAuth Authentication

Five providers need OAuth instead of static API keys: Anthropic (Claude Pro/Max), GitHub Copilot, Google Gemini CLI, Google Antigravity, and OpenAI Codex. The library provides:

- Login flows that present URLs, collect user input, and return credentials
- Automatic token refresh via `getOAuthApiKey()` — call it, get a valid key, don't think about expiry
- A CLI tool: `npx @mariozechner/pi-ai login anthropic`
- A pluggable provider registry — you can register custom OAuth providers or reset to built-ins

Credential storage is the caller's responsibility. The CLI saves to a local `auth.json` as a convenience, but it's not a managed credential store.

### Environment-Based API Key Resolution

`getEnvApiKey(provider)` checks well-known environment variables for 20+ providers. Straightforward for most (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`), but thoughtful for complex cases: AWS Bedrock checks six different credential sources (IAM keys, profiles, ECS task roles, IRSA, bearer tokens), Google Vertex checks both API keys and Application Default Credentials with project/location validation, and Anthropic prefers `ANTHROPIC_OAUTH_TOKEN` over `ANTHROPIC_API_KEY` when both exist.

### Prompt Cache Control

A `cacheRetention` option with three values: `none`, `short` (default), `long`. On Anthropic, `long` extends cache TTL from 5 minutes to 1 hour. On OpenAI, from in-memory to 24 hours. A `sessionId` option enables session-aware caching for providers that support it — OpenAI Codex uses it for prompt caching and request routing.

### Abort and Cancellation

Pass an `AbortSignal` in options, and the stream terminates cleanly with `stopReason: "aborted"`, preserving whatever partial content and usage data was received. Aborted messages can be added to the conversation context and continued — "please continue where you left off" works.

### Faux Provider for Testing

`registerFauxProvider()` creates an in-memory mock that simulates the entire streaming protocol. Responses are queued — static messages or dynamic factory functions that receive context, options, call state, and model. Text, thinking, and tool call blocks are split into randomized chunks and delivered as delta events, with optional real-time pacing.

It even simulates prompt caching: when `sessionId` is active, it tracks the common prefix between consecutive prompts and reports cache reads/writes accordingly. Usage is estimated at ~1 token per 4 characters. Abort signals are respected at chunk boundaries.

Each registration gets a unique source ID, so `unregister()` only removes that specific mock — no collateral damage to other registrations or the built-in providers.

### API Provider Registry

The provider registry is pluggable and lazy-loaded. Built-in providers are registered on module load but their implementations aren't actually imported until first use — keeping startup fast. The registry enforces API type safety: if a model says it uses `anthropic-messages`, it goes to the Anthropic handler, and a mismatch throws immediately.

Custom providers can be registered, unregistered by source ID, or the whole registry can be cleared and reset to defaults.

### Everything Else

- **Image input**: User messages and tool results can include base64 images. Models declare their input modalities so you can check before sending.
- **Payload inspection**: `onPayload` callback lets you inspect or replace the provider request before it's sent. Great for debugging.
- **Unicode safety**: Unpaired surrogate characters are silently stripped before sending — prevents JSON serialization failures that providers would otherwise return.
- **Streaming JSON parsing**: `parseStreamingJson` safely handles truncated JSON mid-stream, always returning a valid object.
- **Cost calculation**: `calculateCost(model, usage)` computes per-request costs from model pricing and token counts.
- **Context serialization**: The entire `Context` (system prompt, messages, tools) is plain JSON — `JSON.stringify` and `JSON.parse` are the persistence API.
- **Model comparison**: `modelsAreEqual()` checks both ID and provider. `supportsXhigh()` detects GPT-5.x and Opus 4.6 models.

---

## Observable Behaviors

**Streaming a completion:** Call `stream(model, context)`. Get an async iterable of events in order: `start` → content events (`text_*`, `thinking_*`, `toolcall_*`) → `done` or `error`. The `.result()` promise resolves to the final `AssistantMessage`. No persistent effect — the caller adds the message to their context to continue.

**One-shot completion:** Call `complete(model, context)`. Get a promise resolving to the final `AssistantMessage`. On failure, it resolves (not rejects) with `stopReason: "error"` and `errorMessage`.

**Model lookup:** `getModel('anthropic', 'claude-sonnet-4-20250514')` returns a typed `Model` object. Returns `undefined` for unknown provider/model combos.

**Tool validation:** `validateToolCall(tools, toolCall)` returns validated (coerced) arguments or throws with a formatted error listing all violations with property paths and received values.

**OAuth login:** `loginProvider(callbacks)` or `npx @mariozechner/pi-ai login [provider]` → presents URL, collects input, returns credentials. CLI saves to `auth.json`.

**Abort:** Signal via `AbortController` → stream emits `error` event with `reason: "aborted"` and the partial message with whatever was received.

---

## Edge Cases & Failure Behavior

| What happens | What the system does |
|---|---|
| No API provider registered for the model's API | Throws synchronously: "No API provider registered for api: X" |
| API key missing | Provider-specific error in `AssistantMessage` with `stopReason: "error"` |
| Rate limit / throttling error | Delivered as `errorMessage` — explicitly NOT classified as context overflow |
| Server requests retry delay > `maxRetryDelayMs` | Fails immediately with error containing the requested delay |
| Faux provider queue empty | Error message: "No more faux responses queued" |
| Tool args fail schema validation | `validateToolCall` throws with formatted multi-line error |
| Unpaired Unicode surrogates in content | Silently removed before sending |
| Incomplete JSON during tool-call streaming | `parseStreamingJson` returns best-effort partial or `{}` |
| Cross-provider replay of encrypted thinking | Dropped silently (only valid for same model) |
| Replaying errored/aborted assistant messages | Dropped entirely to prevent API errors from incomplete turns |
| Orphaned tool calls (no result in history) | Synthetic error results injected automatically |
| Bedrock in browser | Fails at runtime — Bedrock requires Node.js |
| Strict CSP environment | AJV validation disabled gracefully; args pass through unvalidated |
| Expired OAuth token | `getOAuthApiKey` refreshes automatically; throws if refresh fails |

---

## Non-Functional Constraints

- **Node.js ≥ 20.0.0** for server environments.
- **Browser compatible** for core functions. Bedrock, OAuth, and env-var resolution are Node-only.
- **Lazy-loaded providers** — implementations imported on first use, not at startup.
- **Completely stateless** — no stored conversations, credentials, or config. Everything is the caller's responsibility.
- **JSON-serializable everything** — contexts, messages, tools, model objects all survive `JSON.stringify`/`JSON.parse`.
- **Cost data is a snapshot** — pricing comes from generated model data and may drift from current provider pricing.

---

## Non-Goals

**Not an agent loop.** It gives you the streaming and tool-call primitives, but it doesn't execute tools, run multi-turn loops, or make decisions. That's the job of higher-level packages (like the coding-agent that sits alongside it in the same monorepo).

**Not a conversation store.** You get the context object back; you decide where to put it.

**Not a credential manager.** OAuth credentials come back to you; you store them however you like.

**Not a retry system.** No client-side rate limiting, queuing, or automatic retry. It surfaces retry delays for you to handle.

**Not a model selector.** You pick the model explicitly. No smart routing, no fallback chains.

**No prompt engineering.** No templates, no chain-of-thought scaffolding, no RAG helpers.

**No provider health monitoring.** No uptime checks, latency tracking, or failover logic.

---

## What's Interesting Under the Hood

The cross-provider handoff system is the most technically impressive part. It's not just "convert the messages" — it's a multi-pass transformer that normalizes tool call IDs across incompatible formats, strips opaque encrypted thinking payloads that would cause errors on a different provider, drops errored turns that would confuse replay, and injects synthetic tool results for orphaned calls. The fact that you can bounce a conversation between Claude, GPT-5, and Gemini — with tool calls, thinking blocks, and aborted turns in the history — and it just works? That's a lot of careful edge-case handling.

The faux provider is remarkably thorough for a test mock. It doesn't just return canned responses — it simulates the full streaming protocol with chunked deltas, thinking blocks, tool call argument streaming, abort handling, and prompt cache estimation. You can set `tokensPerSecond` to pace delivery in real time. The usage estimates even track common prefixes for cache simulation. Most test mocks give you a happy path; this one gives you the whole protocol.

The lazy loading architecture is worth noting. Each provider is a separate module that only gets imported when you actually call a model that uses it. The startup cost is near-zero regardless of how many providers are registered. If you only use Anthropic, you never pay the cost of importing the OpenAI, Google, or Bedrock SDKs.

The way env-var resolution handles complex auth is thoughtful. For AWS Bedrock alone, it checks IAM keys, named profiles, bearer tokens, ECS task role credentials (both relative and full URI), and web identity tokens. For Google Vertex, it checks API keys, then falls back to Application Default Credentials with project and location validation. For Anthropic, OAuth tokens take precedence over API keys. These aren't just `process.env.FOO` lookups — there's real credential resolution logic.

---

*Spec written by MacGyver — practical, hands-on, finds something interesting in every codebase.*
