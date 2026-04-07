---
tags:
  - ai-tools
  - coding-agent
  - software-development
  - cli
created: 2026-04-07
source: https://github.com/badlogic/pi-mono
---

# PI-Mono: AI Coding Agent for Software Development

## What is PI-Mono?

**PI-Mono** is an open-source monorepo by Mario Zechner containing tools for building AI agents and managing LLM deployments. The flagship product is **pi** -- a minimal, terminal-based coding agent CLI similar to [[Claude Code]] or Cursor, but built around aggressive extensibility over built-in features.

- **Repository**: [github.com/badlogic/pi-mono](https://github.com/badlogic/pi-mono)
- **Website**: [pi.dev](https://pi.dev)
- **License**: MIT
- **Language**: TypeScript (95.9%)

### Core Philosophy

> "Adapt pi to your workflows, not the other way around."

Pi deliberately keeps the core minimal and pushes features into extensions. No built-in MCP, no sub-agents, no plan mode, no permission popups -- all of these are achievable through its extension system.

---

## Architecture Overview

PI-Mono is organized as 7 packages in a monorepo:

| Package | npm Name | Purpose |
|---------|----------|---------|
| `packages/ai` | `@mariozechner/pi-ai` | Unified multi-provider LLM API |
| `packages/agent` | `@mariozechner/pi-agent-core` | Agent runtime with tool calling & state management |
| `packages/coding-agent` | `@mariozechner/pi-coding-agent` | Interactive coding agent CLI (main product) |
| `packages/mom` | `@mariozechner/pi-mom` | Slack bot delegating to the coding agent |
| `packages/tui` | `@mariozechner/pi-tui` | Terminal UI library |
| `packages/web-ui` | `@mariozechner/pi-web-ui` | Web components for AI chat |
| `packages/pods` | `@mariozechner/pi-pods` | CLI for managing vLLM on GPU pods |

### Layered Architecture

```
pi-coding-agent   (CLI + 4 default tools: read, write, edit, bash)
       |
  pi-agent-core   (stateful agent runtime, events, hooks, tools)
       |
     pi-ai         (unified LLM API, 30+ providers, streaming)
```

---

## Installation & Setup

### Install

```bash
npm install -g @mariozechner/pi-coding-agent
```

### Authentication

**Option 1: API Key (environment variable)**

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

**Option 2: Use existing subscription** (Claude Pro/Max, ChatGPT Plus, Gemini CLI, etc.)

```bash
pi
/login   # Select provider interactively
```

**Option 3: Auth file** at `~/.pi/agent/auth.json`

```json
{
  "anthropic": "sk-ant-...",
  "openai": "!op read 'OpenAI Key'",
  "google": "$GOOGLE_API_KEY"
}
```

Supports direct values, shell commands (prefix `!`), and env var references (prefix `$`).

**Credential resolution order**: CLI `--api-key` > `auth.json` > environment variable > `models.json` custom providers.

### Supported Providers (18+)

Anthropic, OpenAI, Azure OpenAI, Google Gemini, Google Vertex, Amazon Bedrock, Mistral, Groq, Cerebras, xAI, OpenRouter, Vercel AI Gateway, ZAI, OpenCode, Hugging Face, Kimi, MiniMax, and custom OpenAI-compatible endpoints.

---

## Basic Usage

### Interactive Mode

```bash
# Start with a prompt
pi "List all .ts files in src/"

# Continue last session
pi -c

# Resume from session browser
pi -r

# Specify model
pi --model openai/gpt-4o "Help me refactor"

# Model with thinking level
pi --model sonnet:high "Solve this complex problem"
```

### Non-Interactive / Piped Mode

```bash
# Print mode (no TUI)
pi -p "Summarize this codebase"

# Pipe input
cat README.md | pi -p "Summarize this text"

# JSON output
pi --mode json "List dependencies"

# Include file references
pi @code.ts @test.ts "Review these files"

# Read-only mode (no writes allowed)
pi --tools read,grep,find,ls -p "Review the code"
```

### Editor Shortcuts

| Shortcut | Action |
|----------|--------|
| `@` | Fuzzy-search project files |
| `Tab` | Path completion |
| `Shift+Enter` | Multi-line input |
| `Ctrl+V` | Paste images |
| `!command` | Run bash, send output to LLM |
| `!!command` | Run bash silently |
| `Ctrl+C` | Clear editor (2x to quit) |
| `Escape` | Cancel/abort (2x for `/tree`) |
| `Ctrl+L` | Model selector |
| `Ctrl+P` / `Shift+Ctrl+P` | Cycle models |
| `Shift+Tab` | Cycle thinking level |

### Message Queue (while agent runs)

| Key | Behavior |
|-----|----------|
| `Enter` | Queue **steering** message (interrupts after current tool calls) |
| `Alt+Enter` | Queue **follow-up** message (runs after agent finishes) |
| `Escape` | Abort and restore queued messages |

---

## Configuration

### Settings Files

- **Global**: `~/.pi/agent/settings.json`
- **Project**: `.pi/settings.json` (overrides global; nested objects merge)

### Key Configuration Options

**Model & Thinking:**

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 10240,
    "high": 32768
  }
}
```

**Compaction (context management):**

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

**Retry:**

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000,
    "maxDelayMs": 60000
  }
}
```

**Model cycling (limit which models appear):**

```json
{
  "enabledModels": ["claude-*", "gpt-4o"]
}
```

### Context Files

Pi loads `AGENTS.md` (or `CLAUDE.md`) at startup from:

1. `~/.pi/agent/AGENTS.md` (global)
2. Parent directories (walking up from cwd)
3. Current directory

You can also:
- Replace the system prompt: `.pi/SYSTEM.md`
- Append to system prompt: `APPEND_SYSTEM.md`

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `PI_CODING_AGENT_DIR` | Override config directory |
| `PI_SKIP_VERSION_CHECK` | Skip version check |
| `PI_CACHE_RETENTION=long` | Extended prompt cache |
| `VISUAL` / `EDITOR` | External editor for `Ctrl+G` |

---

## Extensibility System

This is what makes pi unique. Everything is extensible through 5 mechanisms:

### 1. Extensions (TypeScript Modules)

Place in `~/.pi/agent/extensions/` or `.pi/extensions/`.

```typescript
// ~/.pi/agent/extensions/my-tool.ts
export default function (pi: ExtensionAPI) {
  // Register custom tools
  pi.registerTool({
    name: "deploy",
    description: "Deploy to staging",
    parameters: { env: { type: "string" } },
    execute: async ({ env }) => {
      // deployment logic
    }
  });

  // Register slash commands
  pi.registerCommand("stats", {
    description: "Show session stats",
    execute: async (ctx) => { /* ... */ }
  });

  // Hook into events
  pi.on("tool_call", async (event, ctx) => {
    // intercept, modify, or log tool calls
  });
}
```

The repo ships with **68 example extensions** including permission gates, path protection, sub-agents, plan mode, todo tracking, SSH execution, and even games (Doom overlay, Snake).

### 2. Skills (On-Demand Capabilities)

```markdown
<!-- ~/.pi/agent/skills/review/SKILL.md -->
---
name: code-review
description: Use when the user asks for a code review.
---

# Code Review Skill

## Steps
1. Read the files specified by the user
2. Check for bugs, security issues, and performance problems
3. Suggest improvements with code examples
4. Rate overall code quality (1-10)
```

Invoke with `/skill:code-review` or auto-loaded when the agent detects relevance.

### 3. Prompt Templates

```markdown
<!-- ~/.pi/agent/prompts/review.md -->
Review this code for bugs, security issues, and performance problems.
Focus on: {{focus}}
```

Type `/review` in the editor to expand.

### 4. Themes

Custom visual themes with hot-reloading. Built-in: `dark`, `light`.

### 5. Pi Packages (Shareable Bundles)

Bundle extensions, skills, prompts, and themes as npm or git packages:

```bash
# Install from npm
pi install npm:@foo/pi-tools

# Install from git
pi install git:github.com/user/repo

# Project-local install
pi install -l npm:@foo/pi-tools

# Manage
pi list
pi update
pi remove npm:@foo/pi-tools
pi config   # enable/disable resources
```

Create your own package:

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

---

## Useful Workflows for Software Development

### Workflow 1: Code Review Pipeline

```bash
# Read-only review of a PR branch
git diff main...feature-branch | pi -p "Review this diff for bugs and security issues"

# Deep review with file context
pi --tools read,grep,find,ls "Review src/auth/ for security vulnerabilities"
```

**Project skill** (`.pi/skills/review/SKILL.md`):

```markdown
---
name: pr-review
description: Structured pull request review
---
## Steps
1. Run `git diff main...HEAD` to see all changes
2. Read each changed file fully for context
3. Check for: bugs, security issues, performance, style
4. Output a structured review with severity ratings
```

### Workflow 2: TDD Development Loop

```bash
pi "Implement the UserService class using TDD:
1. First write failing tests in tests/user-service.test.ts
2. Then implement src/user-service.ts to make them pass
3. Refactor if needed
4. Run tests after each step"
```

### Workflow 3: Codebase Onboarding

```bash
# Quick summary
pi -p "Give me a high-level overview of this codebase architecture"

# Deep dive with session continuation
pi "Walk me through the main data flow in this application"
# ... later
pi -c  # continue the session
```

### Workflow 4: Multi-Model Problem Solving

```bash
# Start with fast model for exploration
pi --model haiku "Find all API endpoints in this project"

# Switch to powerful model for complex refactoring
pi --model sonnet:high "Refactor the authentication module to use JWT"

# Use Ctrl+L or Ctrl+P to switch models mid-session
```

### Workflow 5: Automated Refactoring with Safety

Create a permission-gate extension (`.pi/extensions/safe-refactor.ts`):

```typescript
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    if (event.tool === "write" || event.tool === "edit") {
      const confirmed = await ctx.confirm(
        `About to modify: ${event.args.path}\nProceed?`
      );
      if (!confirmed) return ctx.abort();
    }
  });
}
```

Then run:

```bash
pi "Rename all instances of 'userId' to 'user_id' across the project following snake_case convention"
```

### Workflow 6: Documentation Generation

```bash
pi -p @src/api/ "Generate API documentation in Markdown for all exported functions"
```

### Workflow 7: CI/CD Integration with RPC Mode

```bash
# In a CI pipeline script
echo '{"type":"prompt","content":"Run all tests and report failures"}' | \
  pi --mode rpc --tools read,bash 2>/dev/null | \
  jq 'select(.type=="text") | .content'
```

### Workflow 8: Multi-Agent via tmux

```bash
# Terminal 1: Frontend agent
tmux new-session -d -s frontend "cd frontend && pi 'Build the login page component'"

# Terminal 2: Backend agent
tmux new-session -d -s backend "cd backend && pi 'Create the /auth/login endpoint'"

# Terminal 3: Monitor both
tmux split-window -h
```

### Workflow 9: Local Model Development

Add to `~/.pi/agent/models.json`:

```json
{
  "providers": {
    "local-ollama": {
      "type": "openai-compatible",
      "baseUrl": "http://localhost:11434/v1",
      "models": {
        "codellama": { "contextWindow": 16384 },
        "deepseek-coder": { "contextWindow": 32768 }
      }
    }
  }
}
```

Then use:

```bash
pi --model local-ollama/deepseek-coder "Implement this function"
```

### Workflow 10: SDK Embedding in Custom Tools

```typescript
import {
  AuthStorage,
  createAgentSession,
  ModelRegistry,
  SessionManager
} from "@mariozechner/pi-coding-agent";

const authStorage = AuthStorage.create();
const modelRegistry = ModelRegistry.create(authStorage);

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage,
  modelRegistry,
});

// Use programmatically
const result = await session.prompt("What files are in the current directory?");
console.log(result);
```

---

## Session Management

Sessions are stored as JSONL files with a tree structure at `~/.pi/agent/sessions/`.

| Command | Action |
|---------|--------|
| `/tree` | Navigate session history, search, filter, bookmark |
| `/fork` | Create new session from any branch point |
| `pi -c` | Continue last session |
| `pi -r` | Resume from session browser |

**Compaction**: Automatically summarizes older messages when approaching context limits.

---

## Comparison with Other Tools

| Feature | pi | Claude Code | Cursor |
|---------|-----|-------------|--------|
| Open source | Yes (MIT) | No | No |
| Extension system | Full TypeScript API | Hooks + MCP | Limited |
| Multi-provider | 18+ providers | Anthropic only | Multiple |
| Local models | Yes (via models.json) | No | No |
| Session branching | Tree structure + /fork | Linear | Linear |
| Core philosophy | Minimal + extensible | Feature-rich | IDE-integrated |
| SDK for embedding | Yes | Yes | No |

---

## Quick Start Cheatsheet

```bash
# Install
npm install -g @mariozechner/pi-coding-agent

# Set API key
export ANTHROPIC_API_KEY=sk-ant-...

# Start coding
pi "Help me build a REST API"

# Key commands inside pi
/login          # Authenticate
/tree           # Browse session history
/fork           # Branch from any point
/skill:name     # Invoke a skill
/prompt-name    # Expand a prompt template
Ctrl+L          # Switch model
Shift+Tab       # Change thinking level
@filename       # Reference a file
!command        # Run shell command
```

---

## References

- [GitHub Repository](https://github.com/badlogic/pi-mono)
- [Website](https://pi.dev)
- [npm: @mariozechner/pi-coding-agent](https://www.npmjs.com/package/@mariozechner/pi-coding-agent)
