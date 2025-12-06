# PAI Skill System & Workflow Execution Analysis

This document details the findings from an analysis of the Personal AI (PAI) skill system, focusing on its architecture, workflow execution model, and the implementation of complex skills. The analysis is based on the canonical specification in `.claude/skills/CORE/SkillSystem.md` and case studies of the `Research` and `Fabric` skills.

## Key Findings

The PAI skill system is an agent-native, instruction-based framework. It does not use a programmatic runner for workflow execution. Instead, workflows are detailed markdown documents that the AI agent reads and interprets in real-time. This architecture has two major implications:

1.  **Flexibility and Power**: The agent can leverage its full context and reasoning capabilities to execute tasks, rather than being constrained by a rigid script.
2.  **"Native" Execution**: Skills like `Fabric` can have their core logic (prompts) applied directly by the agent, avoiding the overhead of external CLI calls and using the agent's more powerful AI model.

---

## 1. The Canonical Skill Specification

The foundation of the system is the `.claude/skills/CORE/SkillSystem.md` specification, which enforces a strict and consistent structure for all skills.

### Key Structural Requirements:

*   **TitleCase Naming**: All skill directories, workflow files, and reference documents must use `TitleCase` naming (e.g., `CreateSkill`, `UpdateWorkflow.md`). The only exception is the root `SKILL.md` file, which is uppercase.
*   **YAML Frontmatter**: Every `SKILL.md` must contain a YAML frontmatter block with a `name` and a single-line `description`.
*   **Intent-Based Activation**: The `description` must include a `USE WHEN` clause that defines natural language triggers for the skill. This allows the system to use intent matching, not just keyword matching, to activate the correct skill.
*   **Directory Structure**: A mandatory directory structure is enforced:
    *   `SkillName/`
        *   `SKILL.md`: The main skill definition and routing file.
        *   `tools/`: Contains any associated CLI tools, written in TypeScript. This directory must exist, even if empty.
        *   `workflows/`: Contains the markdown-based workflow instructions.

### Workflow Routing

The `SKILL.md` file acts as a central router. Its "Workflow Routing" section maps user triggers to specific workflow files within the `workflows/` directory. When a user's intent matches a trigger, the agent is directed to read and execute the corresponding markdown file.

---

## 2. Workflow Execution Model: Agent-Led Interpretation

**Workflows are not automated scripts.** They are instruction sets for the AI agent.

When a workflow is triggered, the agent loads the relevant `.md` file (e.g., `.claude/skills/Research/workflows/Conduct.md`) into its context and follows the instructions laid out within it.

This model is exemplified by the `Research` skill. The `Conduct.md` workflow is a detailed operational guide for the agent, instructing it on:

*   **Multi-Agent Orchestration**: How to decompose a research query and launch multiple, parallel "researcher" sub-agents.
*   **Operational Modes**: How to behave differently based on user requests like "quick research" vs. "extensive research," including adjusting the number of agents and setting specific timeout rules (e.g., 2 minutes for quick, 10 for extensive).
*   **Synthesis and Formatting**: How to collect results from the parallel agents, synthesize the findings, and format the final report according to a mandatory structure.

This proves that the "execution" is the agent's cognitive process of following detailed instructions, not a separate runner parsing a script.

---

## 3. Case Study: The `Fabric` Skill and Native Patterns

The `Fabric` skill provides the clearest example of the system's "agent-native" execution philosophy.

While a `fabric` CLI tool exists, the `SKILL.md` file explicitly instructs the agent to **avoid** using it for most operations. Instead, the agent should:

1.  Identify the requested pattern (e.g., `extract_wisdom`).
2.  Read the corresponding prompt file directly from the skill's directory (`.claude/skills/Fabric/tools/patterns/extract_wisdom/system.md`).
3.  Apply the instructions from that file to the user's content within its own context.

### Advantages of this Native Approach:

*   **Speed**: It avoids the overhead of spawning an external process.
*   **Power**: It utilizes the agent's primary, high-capability model (e.g., Claude 3 Opus) rather than whatever model the `fabric` CLI might be configured with.
*   **Context-Awareness**: The pattern is executed with the full context of the current conversation, leading to more relevant results.

The CLI is only reserved for tasks that inherently require external access, such as fetching a YouTube transcript or updating the patterns from a remote repository.

## Conclusion

The PAI skill and workflow system is a sophisticated, agent-centric architecture. It relies on a standardized skill structure for discoverability and routing, but its core power comes from using markdown files as detailed instruction sets that the AI agent itself interprets and executes. This model favors flexibility, context-awareness, and the full utilization of the agent's cognitive abilities over rigid, programmatic automation.
