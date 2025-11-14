# Mini-Agent Pipelines Documentation

This document describes all agent execution workflows, data flows, and prompt pipelines in the Mini-Agent system. Each pipeline is documented with ASCII diagrams, data flows, and implementation details.

## Table of Contents

1. [Agent Initialization Pipeline](#agent-initialization-pipeline)
2. [Basic Agent Execution Loop](#basic-agent-execution-loop)
3. [Message Summarization Pipeline](#message-summarization-pipeline)
4. [Skill Loading Pipeline (Progressive Disclosure)](#skill-loading-pipeline-progressive-disclosure)
5. [Session Memory Pipeline](#session-memory-pipeline)
6. [Tool Execution Pipeline](#tool-execution-pipeline)

---

## Agent Initialization Pipeline

### Overview

The agent initialization pipeline prepares the system before any user interaction. This includes loading configuration, initializing tools, loading skills metadata, and injecting it into the system prompt.

### Pipeline Flow

```
START: User runs mini-agent CLI
    ↓
┌─────────────────────────────────────┐
│ 1. Load Configuration               │
│    - Read config.yaml               │
│    - Parse LLM settings             │
│    - Get system_prompt_path         │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 2. Initialize LLM Client            │
│    - Create Anthropic/OpenAI client │
│    - Set API keys & model params    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 3. Initialize Base Tools            │
│    - Create SkillLoader             │
│    - Discover skills in skills_dir  │
│    - Create GetSkillTool            │
│    - Load MCP servers (if enabled)  │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 4. Add Workspace Tools              │
│    - Create ReadTool(workspace)     │
│    - Create WriteTool(workspace)    │
│    - Create EditTool(workspace)     │
│    - Create BashTool                │
│    - Create SessionNoteTool         │
│    - Create RecallNoteTool          │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 5. Load System Prompt               │
│    - Read system_prompt.md          │
│    - If not found, use fallback     │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 6. Inject Skills Metadata           │
│    - Get skills_metadata_prompt()   │
│    - Replace {SKILLS_METADATA}      │
│    - (Progressive Disclosure Lvl 1) │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 7. Create Agent Instance            │
│    - Agent(llm, system_prompt,      │
│            tools, max_steps)        │
│    - Initialize message history     │
│    - messages = [system_message]    │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│ 8. Add User Task Message            │
│    - agent.add_user_message(task)   │
└──────────────┬──────────────────────┘
               ↓
    READY TO RUN: agent.run()
```

### Data Flow

**Input:**
- Configuration file (`config.yaml`)
- System prompt file (`system_prompt.md`)
- Skills directory (`mini_agent/skills/`)
- User task string

**Output:**
- Initialized `Agent` instance
- Message history: `[system_message, user_message]`
- Registered tools dictionary
- SkillLoader with discovered skills

**Key Files:**
- `mini_agent/cli.py` (lines 370-450)
- `mini_agent/agent.py` (lines 45-78)
- `mini_agent/tools/skill_loader.py`

### Prompts Involved

1. **System Prompt** (`system_prompt.md`): Loaded and enhanced with skills metadata
2. **Skills Metadata**: Generated dynamically from discovered skills (Level 1)

**Example Skills Metadata Injection:**
```
Before:
{SKILLS_METADATA}

After:
### Available Skills

**skill-creator** - Guide for creating effective skills...
**artifacts-builder** - Suite of tools for creating elaborate HTML artifacts...
**mcp-builder** - Guide for creating high-quality MCP servers...
[... more skills ...]
```

---

