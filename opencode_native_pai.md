# OpenCode Native PAI: System Design

## 1. Executive Summary

This document outlines the architectural design for porting the Personal AI Infrastructure (PAI) to **OpenCode** as its underlying platform.

While the current PAI implementation relies on Claude Code, the core philosophy of PAI is platform-independence. OpenCode, with its open-source nature, client/server architecture, and robust extensibility (Agents, Commands, Tools, Plugins), offers a compelling target for a "native" PAI implementation that aligns with the goal of user-owned, deterministic, and modular AI infrastructure.

## 2. Platform Capability Analysis

The following table analyzes how OpenCode's native capabilities map to PAI's functional goals.

| PAI Functional Goal | Claude Code Implementation | OpenCode Platform Capability | Comparison / Strategy |
| :--- | :--- | :--- | :--- |
| **Modular Capabilities** | `skills/` directory with `SKILL.md` routing. | **Agents (`.opencode/agent/`)** & **Commands (`.opencode/command/`)**. | **Stronger Native Support.** OpenCode natively globs and registers agents and commands from subdirectories, removing the need for a manual routing layer like PAI's `Task` tool. |
| **Specialized Personalities** | `.claude/agents/*.md` & `SKILL.md` system prompts. | **Agent Configurations (`.opencode/agent/*.md`)**. | **Direct Mapping.** OpenCode agents support frontmatter for model selection, temperature, and permissions, offering more granular control than Claude Code's prompt-only approach. |
| **Event-Driven Automation** | `.claude/hooks/` (Bash/TS scripts). | **Experimental Hooks (`experimental.hook`)** & **Plugins (`.opencode/plugin/`)**. | **Different but Powerful.** OpenCode supports specific hooks (`file_edited`, `session_completed`). For deeper lifecycle management (e.g., Session Start), `plugin` scripts can be used to inject logic into the runtime. |
| **System Context & Identity** | `CORE/CONSTITUTION.md` injected via SessionStart. | **Instructions (`instructions: []`)** in `opencode.jsonc`. | **Native Feature.** OpenCode allows defining global instruction files in config, which perfectly serves the "Constitution" role. |
| **Dynamic Configuration** | `process.env.DA` and `.env` files. | **Variable Substitution (`{env:DA}`)**. | **Direct Mapping.** OpenCode natively supports environment variable substitution in its config files. |
| **Observability** | Custom hook (`capture-all-events.ts`) writing to JSONL. | **OpenTelemetry** & **Plugins**. | **More Standardized.** OpenCode has experimental OpenTelemetry support. For PAI's specific dashboard, a custom plugin can intercept the event bus (if exposed) or use the `session_completed` hook to process logs. |

## 3. Architecture Design

The OpenCode Native PAI architecture flattens the distinction between "platform" and "infrastructure" by leveraging OpenCode's configuration loader as the primary orchestrator.

### 3.1 Directory Structure

We utilize OpenCode's standard configuration paths (`~/.opencode`) but organize it to respect PAI's "Skill" containerization principle.

```
~/.opencode/
├── opencode.jsonc             # Main Configuration (The "Kernel")
├── .env                       # Secrets (API Keys)
├── agent/                     # Agents (Personalities)
│   ├── CORE/                  # Core System Agents
│   │   ├── Supervisor.md      # Main orchestration agent
│   │   └── Critic.md          # Output validator
│   ├── Research/              # Research Skill Agents
│   │   ├── Researcher.md
│   │   └── Searcher.md
│   └── Coding/
│       └── Builder.md
├── command/                   # Workflows (Native Patterns)
│   ├── CORE/
│   │   └── update-pai.md
│   └── Research/
│       ├── deep-dive.md
│       └── quick-search.md
├── tool/                      # Tools (TypeScript/Python)
│   ├── memory-tool.ts
│   └── browser-tool.ts
├── plugin/                    # Deep Integrations (Hooks)
│   ├── pai-observer.ts        # PAI Observability Plugin
│   └── pai-voice.ts           # Voice Server Integration
└── instructions/              # Global Context
    ├── CONSTITUTION.md        # Core Operating Principles
    └── MEMORY.md              # Long-term memory context
```

### 3.2 Core Components

#### Identity & Context (The "Soul")
Instead of a complex hook injecting context, we use OpenCode's native `instructions` directive in `opencode.jsonc`:

```jsonc
// opencode.jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "username": "{env:DA}", // Generic Identity
  "instructions": [
    "instructions/CONSTITUTION.md", // PAI Core Principles
    "instructions/MEMORY.md"        // User Context
  ],
  "agent": {
    // Core Agents
    "supervisor": { "mode": "primary" },
    // Skill Agents (Auto-discovered from agent/**/*.md)
    "researcher": { "mode": "subagent" }
  }
}
```

#### Skills as "Agent+Command Bundles"
In OpenCode PAI, a "Skill" is a logical grouping of **Agents** (who does it) and **Commands** (how to do it).
- **Agents:** Defined in `agent/<SkillName>/<AgentName>.md`. They have specific models, tools, and permissions.
- **Commands:** Defined in `command/<SkillName>/<WorkflowName>.md`. These represent PAI's "Workflows" or "Native Patterns". They are prompt-driven tasks that can be invoked via CLI or by other agents.

#### Tools
PAI tools (Bash/TS scripts) migrate to `.opencode/tool/`. OpenCode requires tools to be TypeScript/JavaScript functions or MCP servers. Existing Bash scripts will be wrapped in a generic `BashTool` or converted to TypeScript for better integration.

### 3.3 Automation & Observability

#### Event Bus (Hooks)
We replace PAI's `hooks/` directory with a robust **Plugin** system.
- **Voice Integration:** A plugin `plugin/pai-voice.ts` listens for the `session_completed` event (via `experimental.hook` configuration) to trigger the TTS server.
- **Observability:** A plugin `plugin/pai-observer.ts` hooks into the OpenCode event bus (if available via `plugin` API) or uses the `experimental.openTelemetry` output to feed the PAI Dashboard.

#### History System
OpenCode saves session history natively. PAI's requirement for "Learning" and "Summary" extraction will be handled by a **Background Agent** triggered on `session_completed`.
```jsonc
// opencode.jsonc
"experimental": {
  "hook": {
    "session_completed": [
       { "command": ["bun", "run", "tool/extract-learnings.ts"] }
    ]
  }
}
```

## 4. Implementation Details

### 4.1 Migration Strategy (PAI to OpenCode)

1.  **Skills Flattening:**
    - PAI: `skills/Research/agents/researcher.md`
    - OpenCode: `.opencode/agent/Research/researcher.md`
2.  **Workflow Conversion:**
    - PAI: `skills/Research/workflows/deep-dive.md`
    - OpenCode: `.opencode/command/Research/deep-dive.md`
    - *Note:* OpenCode commands need specific frontmatter (`description`, `model`).
3.  **Tool Adaptation:**
    - Wrap PAI's `run_in_bash` reliance into native TypeScript tools or enable the `bash` permission for agents.

### 4.2 Configuration Template

A default `opencode.jsonc` for PAI would look like this:

```jsonc
{
  "username": "Kai", // Or {env:DA}
  "instructions": ["instructions/CONSTITUTION.md"],
  "agent": {
    "core": {
      "prompt": "You are the core PAI orchestrator...",
      "mode": "primary",
      "tools": { "memory": true }
    }
  },
  "permission": {
    "bash": "allow", // PAI relies heavily on CLI
    "webfetch": "allow"
  },
  "experimental": {
    "hook": {
      "session_completed": [{ "command": ["bun", ".opencode/hooks/post-session.ts"] }]
    }
  }
}
```

## 5. Strategic Gaps & Solutions

| Gap | Description | Solution |
| :--- | :--- | :--- |
| **"Session Start" Hook** | OpenCode doesn't have an explicit `onStart` hook to print banners or check env vars. | Use a custom entry point script (alias `pai`) that runs checks before launching `opencode`. Or use a `plugin` that runs on load. |
| **Complex Tooling (Bash)** | PAI allows agents to run any bash command. OpenCode has granular permissions. | Enable `bash` permission globally for the user's primary agent to maintain PAI's "Power User" philosophy. |
| **Voice Streaming** | PAI streams voice *during* generation (sometimes). OpenCode is TUI-focused. | Focus on post-turn or post-session TTS initially. Explore `process.stdout` interception in a plugin for real-time streaming. |

## 6. Conclusion

OpenCode is an excellent fit for PAI. Its architecture is cleaner (explicit Agent/Command separation) and more "native" to the PAI philosophy of defining behavior through configuration and files. The migration primarily involves re-organizing the file structure and utilizing OpenCode's native configuration loader instead of custom PAI hooks.
