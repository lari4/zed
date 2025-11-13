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

## Edit Agent Pipeline

### Overview

The Edit Agent handles direct file modifications through AI-generated edits. It supports two modes: editing existing files and creating new files from scratch. The agent parses streaming model output to extract edit instructions and applies them to buffers.

**Primary File:** `crates/agent/src/edit_agent.rs`

**Key Components:**
- `EditAgent` - Manages edit operations
- `EditParser` - Parses streaming edit instructions
- `StreamingFuzzyMatcher` - Resolves old_text to buffer locations
- `StreamingDiff` - Computes character-level diffs

### Mode 1: Edit Existing File

This mode modifies an existing file by matching old text and replacing it with new text.

```
┌─────────────────────────────────────┐
│  User Request: Edit file           │
│  - buffer: Entity<Buffer>          │
│  - edit_description: String         │
│  - conversation: previous context   │
└────────┬────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│              EditAgent::edit()                           │
│                                                          │
│  1. Select edit format based on model:                  │
│     ├─ Claude/GPT-4 -> EditFormat::XmlTags              │
│     └─ Gemini -> EditFormat::DiffFenced                 │
│                                                          │
│  2. Render appropriate prompt template:                 │
│     ├─ edit_file_prompt_xml.hbs (XML format)            │
│     └─ edit_file_prompt_diff_fenced.hbs (diff format)   │
│                                                          │
│  3. Build model request:                                │
│     ├─ Include previous conversation context            │
│     ├─ Add edit prompt with file path                   │
│     └─ Set CompletionIntent::EditFile                   │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│      model.stream_completion(request)                   │
│      [STREAMING EDITS]                                   │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────┐
│              apply_edit_chunks()                               │
│                                                                │
│  Three parallel streams:                                      │
│                                                                │
│  ┌──────────────────────────────────────────────────┐        │
│  │  Stream 1: parse_edit_chunks()                   │        │
│  │  (background thread)                             │        │
│  │                                                  │        │
│  │  Response chunks                                 │        │
│  │       ↓                                          │        │
│  │  EditParser::push_chunk()                        │        │
│  │       ├─ XmlEditParser                           │        │
│  │       │  States: Pending → WithinOldText →       │        │
│  │       │         AfterOldText → WithinNewText     │        │
│  │       │  Extracts: line hints, old/new text      │        │
│  │       │                                          │        │
│  │       └─ DiffFencedEditParser                    │        │
│  │          States: Pending → WithinSearch →        │        │
│  │                 WithinReplace                    │        │
│  │       ↓                                          │        │
│  │  Emit EditParserEvent:                           │        │
│  │   ├─ OldTextChunk { chunk, done, line_hint }    │        │
│  │   └─ NewTextChunk { chunk, done }               │        │
│  └──────────────────────────────────────────────────┘        │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────┐        │
│  │  Stream 2: resolve_old_text()                    │        │
│  │  (foreground task)                               │        │
│  │                                                  │        │
│  │  For each old_text chunk:                        │        │
│  │    ├─ StreamingFuzzyMatcher::match_against()    │        │
│  │    │  - Searches buffer for matching text       │        │
│  │    │  - Uses line hints to narrow search        │        │
│  │    │  - Calculates fuzzy match scores           │        │
│  │    │  - Reports candidates to UI:               │        │
│  │    │    * ResolvingEditRange (refining)         │        │
│  │    │    * AmbiguousEditRange (multiple matches) │        │
│  │    │    * UnresolvedEditRange (no match)        │        │
│  │    │                                             │        │
│  │    └─ Returns:                                   │        │
│  │       ├─ 0 matches -> Skip this edit            │        │
│  │       ├─ 1 match -> Continue to diff             │        │
│  │       └─ 2+ matches -> Report ambiguity, skip   │        │
│  │                                                  │        │
│  │  Output: Option<Range<usize>>                    │        │
│  └──────────────────────────────────────────────────┘        │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────┐        │
│  │  Stream 3: compute_edits()                       │        │
│  │  (foreground task)                               │        │
│  │                                                  │        │
│  │  For each resolved range + new_text:             │        │
│  │    ├─ Convert Range<usize> to Range<Anchor>     │        │
│  │    ├─ StreamingDiff::new(old_text, new_text)    │        │
│  │    │  - Character-level comparison               │        │
│  │    │  - Generates edit operations                │        │
│  │    │                                             │        │
│  │    └─ Collect edits in batches of 32             │        │
│  │       └─> Vec<(Range<Anchor>, String)>          │        │
│  └──────────────────────────────────────────────────┘        │
│         │                                                     │
│         ▼                                                     │
│  ┌──────────────────────────────────────────────────┐        │
│  │  Apply Edit Batch                                │        │
│  │                                                  │        │
│  │  For each batch:                                 │        │
│  │    ├─ buffer.edit(edits, None, cx)               │        │
│  │    ├─ Log: action_log.buffer_edited()            │        │
│  │    ├─ project.update_agent_location()            │        │
│  │    └─ Emit EditAgentOutputEvent::Edited(range)   │        │
│  └──────────────────────────────────────────────────┘        │
└────────┬───────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│         Collect all edit events                         │
│  - EditAgentOutputEvent::Edited for each edit           │
│  - EditAgentOutputEvent::Finished with summary          │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  Buffer Updated │
│  UI Refreshed   │
└─────────────────┘
```

### Mode 2: Create New File

This mode creates a new file with AI-generated content from scratch.

```
┌─────────────────────────────────────┐
│  User Request: Create file         │
│  - buffer: Entity<Buffer>          │
│  - edit_description: String         │
└────────┬────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│            EditAgent::overwrite()                        │
│                                                          │
│  1. Render create_file_prompt.hbs:                      │
│     ├─ file_path: path to new file                      │
│     └─ edit_description: what to create                 │
│                                                          │
│  2. Build model request:                                │
│     ├─ Add create prompt                                │
│     ├─ Set CompletionIntent::CreateFile                 │
│     └─ Disable tool calls                               │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│      model.stream_completion(request)                   │
│      [STREAMING FILE CONTENT]                            │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│            CreateFileParser                              │
│                                                          │
│  States:                                                 │
│    Pending → WithinCodeBlock → Complete                 │
│                                                          │
│  Processing:                                             │
│    ├─ Detect opening ``` marker                         │
│    ├─ Accumulate file content                           │
│    └─ Detect closing ``` marker                         │
│                                                          │
│  Emit: FileContentChunk events                          │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│         Apply File Content                              │
│                                                          │
│  1. Clear buffer: buffer.edit([(0..text.len(), "")])    │
│  2. Insert content: buffer.edit([(0..0, new_content)])  │
│  3. Log: action_log.buffer_created()                    │
│  4. Emit EditAgentOutputEvent::Finished                 │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────┐
│  File Created   │
│  UI Refreshed   │
└─────────────────┘
```

### Edit Format Comparison

#### XML Tags Format (edit_file_prompt_xml.hbs)

**Prompt Structure:**
- Instructions for using `<old_text>` and `<new_text>` tags
- Line number hints via `line` attribute
- Example showing proper format

**Model Output Example:**
```xml
<edits>

<old_text line=10>
function calculateTotal() {
    return price * quantity;
}
</old_text>
<new_text>
function calculateTotal() {
    const tax = 0.08;
    return price * quantity * (1 + tax);
}
</new_text>

</edits>
```

**Parser:** `XmlEditParser` (edit_parser.rs lines 142-250)
- Uses regex to extract line hints: `line="?(\d+)"`
- States track position in XML structure
- Emits chunks as text is parsed

#### Diff Fenced Format (edit_file_prompt_diff_fenced.hbs)

**Prompt Structure:**
- Instructions for SEARCH/REPLACE diff format
- Git-style conflict markers
- Example with multiple hunks

**Model Output Example:**
```
<<<<<<< SEARCH line=10
function calculateTotal() {
    return price * quantity;
}
=======
function calculateTotal() {
    const tax = 0.08;
    return price * quantity * (1 + tax);
}
>>>>>>> REPLACE
```

**Parser:** `DiffFencedEditParser` (edit_parser.rs lines 252+)
- Detects `<<<<<<< SEARCH` and `>>>>>>> REPLACE` markers
- Extracts line hints from SEARCH line
- States track position in diff block

### Fuzzy Matching Algorithm

The `StreamingFuzzyMatcher` resolves old_text to buffer locations:

```
Old Text Chunks (streaming)
         │
         ▼
┌─────────────────────────────────────────┐
│  StreamingFuzzyMatcher                  │
│                                         │
│  Accumulate chunks into full old_text  │
│         │                               │
│         ▼                               │
│  Search buffer for candidates:          │
│    ├─ If line_hint provided:           │
│    │  └─ Search within ±50 lines       │
│    └─ Otherwise: search full buffer    │
│         │                               │
│         ▼                               │
│  Score each candidate:                  │
│    ├─ Exact match: score = 1.0         │
│    └─ Fuzzy match: Levenshtein score   │
│         │                               │
│         ▼                               │
│  Filter and rank:                       │
│    ├─ Keep scores > threshold          │
│    └─ Sort by score (highest first)    │
│         │                               │
│         ▼                               │
│  Report results:                        │
│    ├─ 0 matches: UnresolvedEditRange   │
│    ├─ 1 match: Single Range<usize>     │
│    └─ 2+ matches: AmbiguousEditRange   │
└─────────────────────────────────────────┘
```

### Prompts Used

1. **Edit File (XML)** (`edit_file_prompt_xml.hbs`)
   - Used for Claude and GPT models
   - Provides XML tag-based edit format
   - Includes line number hints

2. **Edit File (Diff)** (`edit_file_prompt_diff_fenced.hbs`)
   - Used for Gemini models
   - Provides Git-style diff format
   - Includes SEARCH/REPLACE blocks

3. **Create File** (`create_file_prompt.hbs`)
   - Used for new file creation
   - Expects triple-backtick wrapped content
   - Tool calls disabled

### Data Transformations

| From | To | Transformation |
|------|-----|----------------|
| User edit description | Rendered prompt | Template rendering with path and description |
| Model response chunks | Edit events | Parser extracts old_text and new_text |
| Old text chunks | Buffer range | Fuzzy matching resolves to Range<usize> |
| Range<usize> | Range<Anchor> | Conversion for stable references |
| Old + new text | Edit operations | StreamingDiff computes character-level changes |
| Edit operations | Buffer updates | `buffer.edit()` applies changes |

### Key Functions

- `EditAgent::edit()` (lines 216-250) - Entry point for file edits
- `EditAgent::overwrite()` (lines 103-135) - Entry point for file creation
- `apply_edit_chunks()` (lines 255-385) - Orchestrates three-stream processing
- `parse_edit_chunks()` (lines 387-420) - Background parsing of model output
- `resolve_old_text()` (lines 274-328) - Fuzzy matching to resolve locations
- `compute_edits()` (lines 332-381) - Diff computation for edits
- `EditParser::push_chunk()` - Parser state machine for extracting edits
- `StreamingFuzzyMatcher::match_against_snapshot()` - Fuzzy text matching

---

