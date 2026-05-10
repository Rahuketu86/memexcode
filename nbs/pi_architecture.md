# `pi-coding-agent` Comprehensive Architecture Guide

## 1. Core Philosophy & Entry Points

`pi` is a minimal, terminal-based coding agent harness written in TypeScript. Its design is **extension-first** and **session-based**, treating every interaction as a persistent, branchable conversation tree stored as JSONL. The agent is built around three layers:

1. **Agent Core** — the `AgentSession` class drives the prompt→toolcall→response loop, manages context, compaction, and model selection
2. **Plugin/Extension Layer** — extensions hook into every lifecycle phase (commands, toolcalls, UI, events)
3. **TUI Layer** — a `blessed`-based terminal UI component system with focus management, widgets, and themes

The main entry point is `pi_actor.ts`, which bootstraps the session, loads extensions, sets up the event loop, and dispatches user input through the appropriate handler (TUI, RPC, or programmatic SDK).

---

## 2. Session & Context Management

### 2.1 Session Format (JSONL)

Sessions are stored as **JSONL trees** (v3). Each line is a JSON object with `type`, `id` (8-char hex), `parentId`, and `timestamp`. The tree structure enables in-place branching without duplicating files.

**Entry Types:**

| Type | Purpose |
|------|---------|
| `session` (header) | Metadata only — version, UUID, CWD, parent session path. Not part of the tree. |
| `message` | Wraps an `AgentMessage` (user, assistant, toolResult, bashExecution, custom, branchSummary, compactionSummary) |
| `compaction` | A `CompactionEntry` with summary, `firstKeptEntryId`, and token stats |
| `branch_summary` | A `BranchSummaryEntry` from `/tree` navigation |
| `model_change` | User switched model mid-session |
| `thinking_level_change` | User changed reasoning depth |
| `custom` | Extension state persistence — NOT in LLM context |
| `custom_message` | Extension-injected message — IS in LLM context |
| `label` | Bookmark/marker on any entry |
| `session_info` | Session display name (set via `/name`) |

### 2.2 Message Types (`AgentMessage` Union)

```
AgentMessage =
  UserMessage           — role "user", content: string | (TextContent | ImageContent)[]
  AssistantMessage      — role "assistant", with provider/model/usage/stopReason
  ToolResultMessage     — role "toolResult", toolCallId, toolName, content, isError
  BashExecutionMessage  — role "bashExecution", direct command execution (not LLM tool calls)
  CustomMessage         — role "custom", extension-generated message in context
  BranchSummaryMessage  — role "branchSummary", context from abandoned branches
  CompactionSummaryMessage — role "compactionSummary", context window reduction record
```

Each message has `TextContent`, `ImageContent`, `ThinkingContent`, and `ToolCall` blocks. ToolCalls carry a unique `id`, `name`, and `arguments` dict.

### 2.3 Context Building

`buildSessionContext()` walks from the current leaf to root:

1. Collects all entries on the active path
2. If a `CompactionEntry` is on the path: emits the summary first, then messages from `firstKeptEntryId` to the compaction point, then post-compaction messages
3. Converts `BranchSummaryEntry` and `CustomMessageEntry` to proper message formats
4. Returns the full message list + current model/thinking level for the LLM

This means the LLM never sees entries *before* `firstKeptEntryId` that were summarized away.

---

## 3. Compaction & Branch Summarization

Two mechanisms manage the context window:

### 3.1 Auto-Compaction

Triggers when `contextTokens > contextWindow - reserveTokens` (default 16,384 reserved).

**Algorithm:**

1. Walk backwards accumulating tokens until `keepRecentTokens` (default 20k) is reached
2. Extract the span of messages to summarize
3. Call the LLM with a structured prompt to produce a markdown summary
4. Append a `CompactionEntry` with summary, `firstKeptEntryId`, and `tokensBefore`
5. The summary + messages from `firstKeptEntryId` onward become the new context

**Split Turns:** When a single turn exceeds the recent budget, pi generates two summaries (history + turn prefix) and merges them. Valid cut points are user messages, assistant messages, bashExecution, and custom messages — never tool results.

### 3.2 Branch Summarization

Triggered by `/tree` navigation. Walks from the old leaf to the common ancestor with the new branch, collects entries, generates a summary, and appends a `BranchSummaryEntry` at the navigation point. This preserves abandoned-branch context.

### 3.3 Summary Format

Both use a structured markdown format with sections:

- **Goal** — what the user is trying to accomplish
- **Constraints & Preferences** — requirements mentioned by user
- **Progress** — Done / In Progress / Blocked (with checklists)
- **Key Decisions** — decisions with rationale
- **Next Steps** — numbered action items
- **Critical Context** — data needed to continue
- **File Tracking** — `<read-files>` and `<modified-files>` blocks

Tool results are truncated to 2,000 chars during serialization to keep summarization within token budgets.

### 3.4 Extension Hooks

- `session_before_compact` — cancel or provide a custom summary
- `session_before_tree` — cancel navigation or provide custom branch summary

---

## 4. Extension System

### 4.1 Lifecycle & Registration

Extensions are TypeScript modules that export a factory function:

```typescript
import { definePiExtension } from "@earendil-works/pi-coding-agent";

export default definePiExtension({
  name: "my-extension",
  version: "1.0.0",
  author: "...",
  async setup(pi, ctx) {
    // pi — the AgentSession instance
    // ctx — ExtensionContext with file helpers, UI, settings access
  }
});
```

### 4.2 Event Hooks (Observer Pattern)

Extensions subscribe to lifecycle events via `pi.on(eventName, handler)`.

| Event | Phase |
|-------|-------|
| `agent_start` | Before processing a prompt |
| `agent_end` | After all messages generated |
| `turn_start` / `turn_end` | Per assistant turn |
| `message_start` / `message_update` / `message_end` | Streaming assistant message |
| `tool_execution_start` / `tool_execution_update` / `tool_execution_end` | Tool execution lifecycle |
| `queue_update` | Steering/follow-up queue changed |
| `compaction_start` / `compaction_end` | Compaction lifecycle |
| `auto_retry_start` / `auto_retry_end` | Retry after transient error |
| `extension_error` | Extension threw an error |
| `session_before_compact` | Before compaction (cancelable) |
| `session_before_tree` | Before `/tree` navigation (cancelable) |
| `session_before_switch` | Before session switch (cancelable) |

### 4.3 Command Registration (Adapter Pattern)

Extensions register slash commands via `pi.registerCommand()`:

```typescript
pi.registerCommand({
  name: "mycommand",
  description: "Do something cool",
  callback: async (args: string[], ctx: CommandContext) => {
    // Can send messages to LLM, interact with tools, etc.
    // Manages its own LLM interaction via ctx.sendMessage()
  }
});
```

Three command sources: extension commands (programmatic), prompt templates (`.md` files), and skills (`SKILL.md` folders).

### 4.4 Extension Context (`ctx`)

- `ctx.cwd` — working directory
- `ctx.settings` / `ctx.setSetting()` — read/write settings (with global/project merging)
- `ctx.fs` — file operations (read, write, glob, tree)
- `ctx.ui` — UI methods (select, confirm, input, editor, notify, setStatus, setWidget)
- `ctx.sendMessage()` / `ctx.abort()` — control the agent lifecycle

---

## 5. TUI (Terminal UI) System

### 5.1 Component Architecture

Built on `blessed`, the TUI has a hierarchical component system:

- **Component Interface** — each component has `render()`, `focus()`, `blur()`, `resize()`, and event handlers
- **Widgets** — Text, Screen, Form, List, Table, Log, ProgressBar, RadioButtons, Dialog, Prompt, Editor
- **Focus/IME Support** — proper keyboard navigation and input method editor (IME) compatibility
- **Themes** — JSON-driven color/styling applied to all components

### 5.2 Extension TUI Interaction

Extensions can:

- **Present dialogs:** `ctx.ui.select()`, `ctx.ui.confirm()`, `ctx.ui.input()`, `ctx.ui.editor()`
- **Display info:** `ctx.ui.notify()`, `ctx.ui.setStatus()`, `ctx.ui.setWidget()`
- **Manipulate the editor:** `ctx.ui.set_editor_text()`
- **Get editor state:** `ctx.ui.getEditorText()`, `ctx.ui.getToolsExpanded()`
- **Set terminal title:** `ctx.ui.setTitle()`

In RPC mode, dialogs translate to a request/response sub-protocol, while fire-and-forget methods are emitted as events.

---

## 6. RPC Mode (Headless Protocol)

RPC mode enables embedding `pi` as a subprocess communicating via **JSONL over stdin/stdout**.

### 6.1 Commands

| Command | Description |
|---------|-------------|
| `prompt` | Send user message (with optional images) |
| `steer` | Queue steering message during streaming |
| `follow_up` | Queue message for after agent finishes |
| `abort` | Abort current operation |
| `bash` | Execute shell command (result added on next prompt) |
| `abort_bash` | Kill running command |
| `set_model` / `cycle_model` | Change the active model |
| `set_thinking_level` / `cycle_thinking_level` | Adjust reasoning depth |
| `compact` / `set_auto_compaction` | Manual compaction control |
| `new_session` / `switch_session` | Session management |
| `fork` / `clone` | Branch operations |
| `get_state` / `get_messages` / `get_session_stats` | Query session state |
| `get_commands` | List available slash commands |
| `export_html` | Export session as HTML |

### 6.2 Streaming Events

Events streamed on stdout as JSON lines:

- `agent_start`, `agent_end` (with all generated messages)
- `turn_start`, `turn_end` (per assistant turn + tool results)
- `message_start`, `message_update`, `message_end` (streaming deltas)
- `tool_execution_start`, `tool_execution_update`, `tool_execution_end` (tool lifecycle)
- `compaction_start`, `compaction_end` (with summary details)
- `auto_retry_start`, `auto_retry_end` (retry tracking)
- `queue_update` (pending messages)
- `extension_error` (extension failures)

### 6.3 Extension UI Sub-protocol (RPC)

Extension dialogs become bidirectional JSON:

- **Request (stdout):** `{"type": "extension_ui_request", "id": "uuid", "method": "select", "options": [...]}`
- **Response (stdin):** `{"type": "extension_ui_response", "id": "uuid", "value": "Allow"}`

Fire-and-forget methods (notify, setStatus, setWidget, etc.) emit requests but expect no response.

---

## 7. Model & Provider System

### 7.1 Provider Configuration

Providers are defined in `~/.pi/agent/models.json`. Built-in providers: anthropic, openai, google, together, groq, openrouter, etc. Custom providers (Ollama, LM Studio, vLLM, proxies) are added by declaring a new provider entry:

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [{ "id": "llama3.1:8b" }]
    }
  }
}
```

**Supported APIs:** `openai-completions`, `openai-responses`, `anthropic-messages`, `google-generative-ai`

### 7.2 Model Configuration

Per-model fields: `id`, `name`, `reasoning`, `input` (text/image), `contextWindow`, `maxTokens`, `cost`, `thinkingLevelMap` (per-level provider value mapping or `null` to hide).

**Compatibility overrides** (`compat`):

- `supportsDeveloperRole` — use `developer` vs `system` role
- `supportsReasoningEffort` — reasoning parameter support
- `supportsUsageInStreaming` — streaming token usage
- `thinkingFormat` — `reasoning_effort`, `deepseek`, `zai`, `qwen`
- `cacheControlFormat` — Anthropic-style caching on OpenAI-compatible providers
- `openRouterRouting` / `vercelGatewayRouting` — provider routing preferences

### 7.3 Value Resolution for Secrets

`apiKey` and `headers` support three formats:

- **Environment variable:** `"MY_API_KEY"` — reads from env
- **Shell command:** `"!op read 'op://vault/item/credential'"` — executes at request time
- **Literal value:** `"sk-..."` — used directly

### 7.4 Model Cycling & Discovery

Models are discovered from built-in lists + custom `models.json`. `--models` flag and `enabledModels` setting filter via glob patterns (`claude-*`, `gpt-4o`). `Ctrl+P` cycles through enabled models. `--list-models` shows all available.

---

## 8. Settings Architecture

Two-level hierarchy with **shallow merge** (nested objects merge, arrays replace):

| Level | Path | Scope |
|-------|------|-------|
| Global | `~/.pi/agent/settings.json` | All projects |
| Project | `<cwd>/.pi/settings.json` | Current directory only |

### Key Settings

| Category | Notable Settings |
|----------|-----------------|
| Model | `defaultProvider`, `defaultModel`, `defaultThinkingLevel`, `hideThinkingBlock`, `thinkingBudgets` |
| UI | `theme`, `quietStartup`, `doubleEscapeAction`, `treeFilterMode`, `editorPaddingX` |
| Compaction | `compaction.enabled`, `compaction.reserveTokens`, `compaction.keepRecentTokens` |
| Branch | `branchSummary.reserveTokens`, `branchSummary.skipPrompt` |
| Retry | `retry.enabled`, `retry.maxRetries`, `retry.baseDelayMs`, `retry.provider.timeoutMs` |
| Steering | `steeringMode`, `followUpMode`, `transport` (sse/websocket/auto) |
| Terminal | `terminal.showImages`, `terminal.imageWidthCells`, `images.autoResize` |
| Shell | `shellPath`, `shellCommandPrefix`, `npmCommand` |
| Resources | `packages`, `extensions`, `skills`, `prompts`, `themes`, `enableSkillCommands` |

### Resource Discovery

Arrays support globs and exclusions (`!pattern` to exclude, `+path` to force-include, `-path` to force-exclude). Resources loaded from:

- Local paths (files or directories)
- npm packages (`pi install npm:@scope/pkg`)
- git packages (`pi install git:github.com/user/repo@v1`)

---

## 9. Package System

### 9.1 Package Types

`pi-coding-agent` packages bundle extensions, skills, prompt templates, and themes for distribution via npm or git.

**Manifest format** (`package.json`):

```json
{
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

**Conventional directories** — if no `pi` manifest, auto-discovers from `extensions/`, `skills/`, `prompts/`, `themes/`.

### 9.2 Installation

```bash
pi install npm:@foo/bar@1.0.0     — pinned, skips `pi update`
pi install git:github.com/user/repo@v1   — pinned by ref
pi install /local/path             — direct path
pi install ./relative/path         — relative to settings file
```

Global installs use `~/.pi/agent/settings.json`. Project installs (`-l` flag) use `.pi/settings.json` and store git packages under `.pi/git/`, npm under `.pi/npm/`.

### 9.3 Peer Dependencies

Core packages listed in `peerDependencies` with `"*"` range and **not bundled**: `@earendil-works/pi-ai`, `@earendil-works/pi-agent-core`, `@earendil-works/pi-coding-agent`, `@earendil-works/pi-tui`, `typebox`. Other packages must be bundled and referenced via `node_modules/` paths.

---

## 10. Skills System

Skills follow the [Agent Skills standard](https://github.com/anthropics/anthropic-quickstarts/blob/main/tool-use/agent-skills/AGENTS.md). A skill is a directory containing a `SKILL.md` file.

**Discovery paths:**

- `~/.pi/agent/skills/` — global (user)
- `<cwd>/.pi/agent/skills/` — project
- Paths from `settings.json` → `skills` array
- From packages via `pi` manifest `skills` key

Skills expose `/skill:<name>` commands that inject instructions into the conversation context. When the model encounters the skill content, it understands the new capabilities.

**Security:** Skills run with full system access. Extensions execute arbitrary code and skills can instruct the model to run executables. Source should always be reviewed before installing third-party packages.

---

## 11. Prompt Templates

Prompt templates are Markdown files that expand into full system prompts when invoked via `/template-name`. They support:

- **Frontmatter configuration** — metadata like name, description
- **Variable substitution** — references to file contents, shell output, etc.
- **Context injection** — can read files, run commands, and embed results

Templates are discovered from:

- `~/.pi/agent/prompts/` — global
- `<cwd>/.pi/agent/prompts/` — project
- `settings.json` → `prompts` array
- Packages via `pi` manifest

---

## 12. Themes

Themes are JSON files defining TUI colors and styles. They follow the `blessed` theming convention with entries for every UI element.

**Loading:** from `~/.pi/agent/themes/`, project `.pi/themes/`, `settings.json` → `themes` array, or packages. Activated via `theme` setting or `/theme` command. Built-in themes: `dark`, `light`.

---

## 13. Architecture Patterns Used

| Pattern | Where It Appears |
|---------|-----------------|
| **Strategy** | Model providers (`openai-completions`, `anthropic-messages`, etc.) are swappable strategies for LLM communication |
| **Factory** | `AgentSession` is created via `SessionManager.create()` / `SessionManager.open()` / `SessionManager.continueRecent()` — factory methods for different session states |
| **State Machine** | Agent lifecycle: idle → processing → streaming → idle, with transition guards (compaction, retry) |
| **Adapter** | RPC mode adapts the full agent to a JSONL protocol; extension `ctx` adapts filesystem/UI operations |
| **Observer** | `pi.on(event, handler)` — extensions subscribe to lifecycle events for cross-cutting concerns |
| **Template Method** | Compaction and branch summarization follow a fixed algorithm (find cut point → extract → LLM summarize → append entry) with extension hooks to override the LLM call |
| **Composite** | The session entry tree (parent-child relationships, path walking) |
| **Decorator** | Skills/prompt templates decorate the system prompt at runtime |

---

## 14. Directory Structure (Conceptual)

```
~/.pi/agent/
├── settings.json          # Global settings
├── models.json            # Custom providers & models
├── extensions/            # Local extensions (.ts)
├── skills/                # Local skills (SKILL.md dirs)
├── prompts/               # Local prompt templates (.md)
├── themes/                # Local themes (.json)
├── sessions/              # Session JSONL files
│   └── --path--/timestamp_uuid.jsonl
└── git/                   # Cloned git packages
```

```
<project>/.pi/
├── settings.json          # Project settings (override global)
├── extensions/
├── skills/
├── prompts/
├── themes/
├── npm/                   # Project-local npm packages
└── git/                   # Project-local git packages
```

---

## 15. SDK Embedding

For programmatic embedding in Node.js/TypeScript:

```typescript
import { AgentSession } from "@earendil-works/pi-coding-agent";

const session = await AgentSession.create({ cwd });
session.on("message_update", (event) => { ... });
session.on("tool_execution_end", (event) => { ... });
await session.processPrompt("Hello!");
```

The SDK provides `AgentSession` as the primary API, with events for streaming, tool execution tracking, and compaction monitoring. For a Python client, the RPC mode provides the complete command/event protocol over stdin/stdout (with a reference Python client in the docs).
