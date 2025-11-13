# Zed AI Agent Pipelines Documentation

This document describes all the agent workflows and pipelines in the Zed editor, including data flow, prompt usage, and component interactions.

---

## Table of Contents

1. [Conversational Agent Pipeline](#conversational-agent-pipeline)
2. [Edit Agent Pipeline](#edit-agent-pipeline)
3. [Tool Execution Pipeline](#tool-execution-pipeline)
4. [Context Retrieval Pipeline](#context-retrieval-pipeline)
5. [Edit Prediction and Application Pipeline](#edit-prediction-and-application-pipeline)
6. [System Prompt Construction Pipeline](#system-prompt-construction-pipeline)

---

## Conversational Agent Pipeline

### Overview

The conversational agent handles multi-turn conversations with the AI model, managing message history, tool calls, and streaming responses. This is the main agent interaction pattern.

**Primary File:** `crates/agent/src/thread.rs`

**Key Components:**
- `Thread` - Manages conversation state and message history
- `Message` - Union of user and agent messages
- `PromptId` - Unique identifier for each conversation turn

### Data Flow Diagram

```
┌─────────────────┐
│   User Input    │
│  (text + ctx)   │
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│           Thread::send(message_id, content)              │
│  - Creates new PromptId                                  │
│  - Adds UserMessage to messages vector                   │
│  - Includes context mentions (files, symbols, etc.)      │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│                    Thread::run_turn()                    │
│  - Cancels any existing running turn                     │
│  - Gets language model from registry                     │
│  - Retrieves agent profile settings                      │
│  - Filters enabled tools for this model/profile          │
│  - Spawns async task for execution                       │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│          Thread::build_completion_request()              │
│                                                          │
│  1. Render system prompt (system_prompt.hbs):           │
│     ├─ ProjectContext (worktrees, rules, OS, shell)     │
│     ├─ Available tools list                             │
│     └─ User custom instructions                         │
│                                                          │
│  2. Convert messages to request format:                 │
│     ├─ System message (rendered template)               │
│     ├─ Previous messages (user/agent)                   │
│     └─ Current pending message (if any)                 │
│                                                          │
│  3. Include tool definitions:                           │
│     ├─ Tool names                                       │
│     ├─ Descriptions                                     │
│     └─ Input schemas (JSON Schema)                      │
│                                                          │
│  4. Configure settings:                                 │
│     ├─ Temperature (varies by model)                    │
│     ├─ thinking_allowed = true                          │
│     └─ Max tokens, stop sequences                       │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│         model.stream_completion(request)                 │
│                   [STREAMING LOOP]                       │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│                  handle_completion_event()                         │
│                                                                    │
│  Event Types:                                                     │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ StartMessage                                             │    │
│  │  └─> Create new AgentMessage in pending_message          │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Text                                                      │    │
│  │  ├─> Append to AgentMessage::Text content                │    │
│  │  └─> Emit ThreadEvent::AgentText (for UI update)         │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Thinking                                                  │    │
│  │  ├─> Add AgentMessage::Thinking                          │    │
│  │  └─> Emit ThreadEvent::AgentThinking                     │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ ToolUse                                                   │    │
│  │  ├─> handle_tool_use_event()                             │    │
│  │  │   ├─ Lookup tool by name                              │    │
│  │  │   ├─ Parse input JSON                                 │    │
│  │  │   ├─ Create ToolCallEventStream                       │    │
│  │  │   └─ Spawn tool.run() async task                      │    │
│  │  │                                                        │    │
│  │  └─> Await tool result                                   │    │
│  │      └─> Add LanguageModelToolResult to pending_message  │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ UsageUpdate                                               │    │
│  │  └─> Update token usage statistics                       │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Stop                                                      │    │
│  │  ├─ StopReason::EndTurn -> End loop                      │    │
│  │  ├─ StopReason::ToolUse -> Continue with tool results    │    │
│  │  ├─ StopReason::MaxTokens -> Return error                │    │
│  │  └─ StopReason::Refusal -> Return error                  │    │
│  └──────────────────────────────────────────────────────────┘    │
└────────┬───────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│            flush_pending_message()                       │
│  - Move AgentMessage from pending to messages vector     │
│  - messages.push(Message::Agent(pending))                │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│         Emit ThreadEvent::Stop(reason)                   │
│  - Notifies UI that turn is complete                     │
│  - Includes final stop reason                            │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  UI Response    │
│   (rendered)    │
└─────────────────┘
```

### Stop Reason Handling

The agent handles different stop reasons to determine whether to continue processing:

```
StopReason::EndTurn
  └─> Complete turn normally
      └─> Return response to user

StopReason::ToolUse
  └─> Tools were called, await results
      └─> Send tool results back to model
          └─> Continue streaming for next response

StopReason::MaxTokens
  └─> Hit token limit
      └─> Return CompletionError::MaxTokens
          └─> User sees error

StopReason::Refusal
  └─> Model refused to respond
      └─> Return CompletionError::Refusal
          └─> User sees error message
```

### Prompts Used

1. **System Prompt** (`system_prompt.hbs`)
   - Rendered once per turn
   - Includes project context, tools, and custom rules
   - See [System Prompt Construction Pipeline](#system-prompt-construction-pipeline)

### Data Transformations

| From | To | Transformation |
|------|-------|----------------|
| User input text | `UserMessage` | Wraps text with context mentions |
| `UserMessage` | `LanguageModelRequestMessage` | Converts to API format with roles |
| Model events | `AgentMessage` | Accumulates text, thinking, tool uses |
| `LanguageModelToolUse` | Tool execution task | Parses JSON input, spawns async task |
| Tool result | `LanguageModelToolResult` | Wraps output in result message |
| `AgentMessage` (pending) | `Message::Agent` | Finalizes and adds to history |

### Key Functions

- `Thread::send()` (lines 1132-1155) - Entry point for user messages
- `Thread::run_turn()` (lines 1165-1216) - Orchestrates a single conversation turn
- `Thread::run_turn_internal()` (lines 1218-1310) - Main streaming loop
- `Thread::build_completion_request()` (lines 1843-1894) - Builds model request
- `Thread::handle_completion_event()` (lines 1399-1499) - Processes streaming events
- `Thread::handle_tool_use_event()` (lines 1500-1608) - Executes tools
- `Thread::flush_pending_message()` (lines 1610-1700) - Finalizes agent response

---

