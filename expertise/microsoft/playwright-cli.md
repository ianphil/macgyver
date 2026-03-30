---

# Product Spec: microsoft/playwright-cli

## Problem Statement

Coding agents — the kind that write tests, debug web apps, and automate browsers while simultaneously reasoning about a large codebase — have a context window problem. The existing solution for browser automation in agentic workflows is MCP, but MCP's approach loads verbose accessibility trees and large tool schemas directly into the model's context. When your agent is already holding thousands of lines of source code, test output, and reasoning, burning tokens on a full DOM dump for every browser interaction is genuinely expensive.

Playwright-CLI solves this by flipping the model: instead of a persistent server that streams rich browser state into the model, it's a simple command-line tool. Short commands, structured text output, no bloat. The agent calls `playwright-cli click e15` and gets back a compact snapshot. The page structure stays in a file. Only what the agent needs lands in context.

This is browser automation designed specifically for the shape of a coding agent's workday.

---

## Actors & Goals

**The Coding Agent** (Claude Code, GitHub Copilot, etc.) is the primary user. It wants to:
- Navigate and interact with web pages without exceeding context limits
- Verify app behavior during test writing or debugging
- Generate real Playwright test code from actual browser interactions
- Run multiple isolated browser sessions in parallel for different tasks

**The Developer** uses it directly to:
- Explore and debug a running web app by hand
- Watch what their coding agent is doing in real time
- Generate a test file by recording manual interactions
- Recover from a stuck or zombie browser session

**The CI System** needs it to:
- Run browser automation as a step in a pipeline
- Clean up sessions and browser data reliably after each run

---

## Operator Value

Before this, giving a coding agent browser access meant either accepting MCP's context overhead or writing bespoke shell scripts around Playwright. Now there's a purpose-built interface that:

- Keeps browser state out of the model context by default
- Lets agents interact with pages using short, stable element refs instead of CSS selectors or DOM trees
- Turns every browser action into ready-to-paste Playwright TypeScript — so test generation is a byproduct of normal exploration
- Supports concurrent named browser sessions, so one agent can manage multiple contexts simultaneously
- Ships with installable "skills" — structured guidance documents that coding agents pick up automatically without any extra prompting

---

## Core Capabilities

**Browser session management.** Open, close, name, and isolate browser instances. Sessions are in-memory by default (cookies and storage vanish when the browser closes) or optionally persisted to disk across restarts. Multiple named sessions can run concurrently, fully isolated from each other.

**Page interaction.** Click, double-click, fill, type, drag, hover, select, check, uncheck, press keys, move the mouse, scroll, resize the window, accept or dismiss dialogs. The full range of human-like interactions you'd expect.

**Snapshot-based element addressing.** After every command, the tool returns a structured text snapshot of the current page state: the URL, the title, and a reference to a snapshot file. That file lists page elements with short stable IDs (e1, e2, e3…). Subsequent commands target those IDs directly. No DOM, no accessibility tree in the context.

**Multi-tab control.** Open new tabs, list open tabs, switch between them, and close individual ones — all within a session.

**Browser storage inspection and manipulation.** Read, write, and clear cookies, localStorage, and sessionStorage. Save and restore full storage state to a file, useful for loading a pre-authenticated session without going through login flows.

**Network interception and mocking.** Intercept requests by URL pattern and return custom status codes, bodies, or headers. Block specific origins entirely. List and remove active intercepts. For more complex scenarios (conditional responses, response modification, simulated latency), execute arbitrary Playwright code inline.

**Diagnostics and recording.** Capture screenshots (full page or specific element), save pages as PDFs, inspect console messages, review network traffic, record execution traces, and capture video of a session.

**Test code generation.** Every interaction command prints the equivalent Playwright TypeScript code alongside its output. Collecting these snippets gives you the skeleton of a test file.

**Debugging live tests.** When a Playwright test is paused at failure, the CLI can attach to that test's browser, allowing exploration and interaction while the test waits. Actions taken in this mode also generate Playwright code, which can be copied back into the test.

**Skills installation.** Running `playwright-cli install --skills` drops structured guidance files into the project. Coding agents discover these automatically and use them to understand available commands, workflows, and best practices — without the developer needing to prompt for it.

**Visual monitoring dashboard.** `playwright-cli show` opens a live window displaying all active browser sessions as screencasts. You can click into any session to take over remote control, navigate with a full address bar, and observe or intervene in what the agent is doing.

**Configuration.** A JSON config file controls browser choice, profile persistence, viewport, network allow/blocklists, action and navigation timeouts, output directory, output mode (stdout or files), console log level, and more. Config can also be set via environment variables.

---

## Observable Behaviors

**Opening a browser and navigating**
- *Trigger*: `playwright-cli open <url>`
- *Response*: A structured text block — page URL, page title, and a path to a snapshot file containing element refs
- *Effect*: A browser session is running in the background, addressable by subsequent commands
- *Failure*: If the URL is unreachable or navigation times out (default 60 seconds), the command exits with a non-zero code and an error message

**Interacting with an element**
- *Trigger*: `playwright-cli click e15` (or any interaction command with a ref, CSS selector, or role selector)
- *Response*: The Playwright TypeScript code that was executed, followed by an updated snapshot of the page state
- *Effect*: The page changes as a result of the interaction; the new snapshot reflects those changes
- *Failure*: If the element isn't found or the action times out (default 5 seconds), the command exits non-zero with a description of what failed

**Taking a snapshot**
- *Trigger*: `playwright-cli snapshot` (also happens automatically after every command)
- *Response*: Page URL, title, and a link to the snapshot file; the file itself lists all interactive elements with their refs, roles, labels, and current values
- *Effect*: A `.playwright-cli/page-<timestamp>.yml` file is created (or a named file if `--filename=` is passed)
- *Failure*: If no browser session is open, exits with an error

**Running multiple sessions**
- *Trigger*: `playwright-cli -s=project-a open ...` and `playwright-cli -s=project-b open ...`
- *Response*: Each session operates independently with its own cookies, storage, history, and tabs
- *Effect*: Both sessions remain alive concurrently; commands routed to each session only affect that session
- *Failure*: If the session name is unknown, a new session is created rather than failing

**Installing skills**
- *Trigger*: `playwright-cli install --skills`
- *Response*: Confirms skill files were written
- *Effect*: A `SKILL.md` and reference docs are written into the project; coding agents with skill-discovery support begin using them automatically

**Monitoring sessions**
- *Trigger*: `playwright-cli show`
- *Response*: A visual window opens showing all active sessions as live screencasts
- *Effect*: The developer can observe, click into, and take over any session interactively; pressing Escape releases control back

**Attaching to a paused test**
- *Trigger*: A Playwright test run started with `PWPAUSE=cli`, then `playwright-cli open --attach=<test-worker-id>`
- *Response*: The CLI connects to the test's browser, prints a snapshot, and accepts subsequent commands
- *Effect*: Actions taken are reflected in the live test browser; generated code can be copied back into the test
- *Failure*: If the test process has already resumed or exited, the attach fails with a connection error

---

## Edge Cases

- **Skills-less operation**: Agents that haven't run `install --skills` can still use the tool by running `playwright-cli --help` and reading the command list from there. The README explicitly documents this path.
- **Selector fallback**: When a ref from a stale snapshot is used, the tool will attempt to resolve it. If the element no longer exists, the command fails and a fresh snapshot is needed.
- **Persistent vs. ephemeral profiles**: By default, a session's profile exists only in memory. If you close the browser and reopen without `--persistent`, cookies and storage are gone. This is by design — ephemeral by default keeps sessions clean — but it can surprise agents that assume continuity.
- **Zombie processes**: Long-running agent workflows can leave browser processes behind. `playwright-cli kill-all` forcefully terminates all of them when `close-all` isn't enough.
- **Concurrent conflicting sessions**: Two commands targeting the same unnamed session at the same time are not explicitly serialized by the tool — coordination is the caller's responsibility.
- **File upload restrictions**: By default, the tool only allows uploading files from within the workspace. Unrestricted file access is opt-in.
- **CDP and remote connections**: The tool can connect to an existing browser via a Chrome DevTools Protocol endpoint, or to a remote Playwright server — enabling headless-on-server, headed-in-browser workflows.

---

## Non-Functional Constraints

- Requires Node.js 18 or newer
- Headless by default; headed mode is opt-in per session
- Default action timeout: 5 seconds; default navigation timeout: 60 seconds (both configurable)
- Tested on Ubuntu, macOS, and Windows in CI
- Published to npm as `@playwright/cli`; installable globally or invocable via `npx`
- Snapshot files are written to `.playwright-cli/` in the working directory by default
- Output can be directed to stdout (default) or files via configuration

---

## Non-Goals

- **Not a replacement for Playwright MCP** in workflows where persistent browser state, continuous introspection, or iterative reasoning over page structure is the point. The README is explicit: MCP is the right tool for exploratory automation, self-healing tests, and long-running autonomous loops where context cost is acceptable.
- **Not a standalone test runner.** It generates test code but doesn't run test suites itself — that's Playwright's job.
- **Not a visual regression tool.** Screenshots are an output artifact, not a comparison mechanism.
- **Not a proxy or security boundary.** Network allow/blocklists are documented as not constituting a security boundary; they don't intercept redirects.
- **Not designed for browser-less environments** or browsers other than Chromium, Firefox, WebKit, Chrome, and Edge.

---

## Suspected Implementation Leakage

- The README distinguishes CLI from MCP partly in terms of "tool schemas" and "accessibility trees" — these are MCP-specific protocol details. A cleaner product-level framing: *CLI delivers structured page summaries on demand; MCP maintains a live, queryable model of the page.*
- `run-code` accepts arbitrary Playwright TypeScript and evaluates it — this is genuinely a product feature (escape hatch for complex scenarios), but it also exposes the underlying library at the surface. Worth calling out as an intentional power-user affordance rather than a specification-level concern.
- Environment variables are prefixed `PLAYWRIGHT_MCP_` rather than `PLAYWRIGHT_CLI_` for several config options — suggesting some config infrastructure is shared with the MCP server under the hood. Doesn't affect behavior from the outside, but worth noting if someone is building tooling around those env vars.