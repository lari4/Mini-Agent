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

## Basic Agent Execution Loop

### Overview

The core agent execution loop is the main pipeline that processes user tasks through iterative LLM calls, tool executions, and message management. It continues until the task is complete (no tool calls) or max steps is reached.

### Pipeline Flow

```
START: agent.run()
    ↓
┌──────────────────────────────────────────────────────────────┐
│ LOOP: while step < max_steps                                 │
│                                                               │
│   ┌──────────────────────────────────────┐                   │
│   │ 1. Check Token Limit                 │                   │
│   │    - Estimate current token count    │                   │
│   │    - If > token_limit:               │                   │
│   │      → Trigger summarization         │                   │
│   │        (see Summarization Pipeline)  │                   │
│   └────────────────┬─────────────────────┘                   │
│                    ↓                                          │
│   ┌──────────────────────────────────────┐                   │
│   │ 2. Call LLM                          │                   │
│   │    Input:                            │                   │
│   │    - messages: [system, user,        │                   │
│   │                 assistant, tool...]  │                   │
│   │    - tools: [tool definitions]       │                   │
│   │                                      │                   │
│   │    Output:                           │                   │
│   │    - response.content (text)         │                   │
│   │    - response.thinking (optional)    │                   │
│   │    - response.tool_calls (optional)  │                   │
│   │    - response.finish_reason          │                   │
│   └────────────────┬─────────────────────┘                   │
│                    ↓                                          │
│   ┌──────────────────────────────────────┐                   │
│   │ 3. Add Assistant Message to History  │                   │
│   │    messages.append(Message(          │                   │
│   │      role="assistant",               │                   │
│   │      content=response.content,       │                   │
│   │      thinking=response.thinking,     │                   │
│   │      tool_calls=response.tool_calls  │                   │
│   │    ))                                │                   │
│   └────────────────┬─────────────────────┘                   │
│                    ↓                                          │
│   ┌──────────────────────────────────────┐                   │
│   │ 4. Display Response                  │                   │
│   │    - Print thinking (if present)     │                   │
│   │    - Print assistant content         │                   │
│   └────────────────┬─────────────────────┘                   │
│                    ↓                                          │
│              ┌─────────────┐                                 │
│              │ Tool calls? │                                 │
│              └──┬───────┬──┘                                 │
│                 │ NO    │ YES                                │
│                 ↓       ↓                                    │
│         ┌───────────┐  ┌────────────────────────────┐       │
│         │ COMPLETE  │  │ 5. Execute Tool Calls      │       │
│         │ Return    │  │    FOR EACH tool_call:     │       │
│         │ content   │  │      - Extract function    │       │
│         └───────────┘  │        name & arguments    │       │
│                        │      - Get tool from dict  │       │
│                        │      - Execute tool:       │       │
│                        │        result = await      │       │
│                        │        tool.execute(**args)│       │
│                        │      - Handle errors       │       │
│                        │      - Display result      │       │
│                        │      - Add tool message:   │       │
│                        │        messages.append(    │       │
│                        │          Message(          │       │
│                        │            role="tool",    │       │
│                        │            content=result, │       │
│                        │            tool_call_id=id,│       │
│                        │            name=fn_name    │       │
│                        │          ))                │       │
│                        └──────────┬─────────────────┘       │
│                                   ↓                          │
│                        ┌──────────────────────┐             │
│                        │ 6. Increment Step    │             │
│                        │    step += 1         │             │
│                        └──────────┬───────────┘             │
│                                   ↓                          │
│                                 LOOP                         │
└──────────────────────────────────────────────────────────────┘
    ↓
END: Return final response or "Max steps reached"
```

### Data Flow

**Input:**
- `self.messages`: Current message history
- `self.tools`: Dictionary of available tools
- `self.max_steps`: Maximum iteration limit (default: 50)
- `self.token_limit`: Token threshold for summarization (default: 80000)

**Output:**
- Final assistant response (string)
- Updated message history
- Execution logs (via AgentLogger)

**Message History Evolution:**

```
Step 0:
[system, user]

Step 1 (after LLM):
[system, user, assistant(tool_calls)]

Step 1 (after tools):
[system, user, assistant(tool_calls), tool_result_1, tool_result_2, ...]

Step 2 (after LLM):
[system, user, assistant(tool_calls), tool_result_1, tool_result_2, assistant(tool_calls)]

Step 2 (after tools):
[system, user, assistant(tool_calls), tool_result_1, tool_result_2, assistant(tool_calls), tool_result_3, ...]

... continues until no tool_calls or max_steps reached
```

### Prompts Involved

1. **System Prompt**: Included as first message in every LLM call
2. **User Task**: Initial user message that starts the execution
3. **Assistant Responses**: Agent's reasoning and tool call decisions
4. **Tool Results**: Structured data returned from tool executions

**Prompt Data Sent to LLM Each Step:**

```python
messages = [
    Message(role="system", content=system_prompt),      # Always included
    Message(role="user", content="User task..."),       # Initial task
    Message(role="assistant", content="...", tool_calls=[...]),  # Previous responses
    Message(role="tool", content="...", name="tool_name"),  # Tool results
    # ... more assistant/tool pairs
]

tools = [
    {
        "name": "read_file",
        "description": "Read file contents...",
        "parameters": {...}
    },
    {
        "name": "bash",
        "description": "Execute shell commands...",
        "parameters": {...}
    },
    # ... all available tools
]
```

### Key Implementation Details

**Location:** `mini_agent/agent.py` (lines 259-410)

**Token Estimation:** Uses `tiktoken` library with `cl100k_base` encoding for accurate token counting

**Tool Execution:** All tool execution is wrapped in try-except to convert exceptions into `ToolResult` objects with error details

**Logging:** Every LLM request, response, and tool execution is logged to `logs/agent_run_YYYYMMDD_HHMMSS.jsonl`

**Error Handling:**
- LLM failures: Return error message and stop execution
- Tool failures: Convert to ToolResult with error, continue execution
- Max steps: Return warning message

### Exit Conditions

1. **Success**: `response.tool_calls` is empty (task complete)
2. **Max Steps**: `step >= max_steps` (iteration limit reached)
3. **LLM Error**: Exception during LLM call (critical failure)

---

