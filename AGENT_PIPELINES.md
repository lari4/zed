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

## Tool Execution Pipeline

### Overview

Tools extend the agent's capabilities by allowing it to interact with the file system, terminal, web, and other resources. The tool execution pipeline handles tool discovery, selection, invocation, and result integration back into the conversation.

**Primary File:** `crates/agent/src/tools.rs` and individual tool implementations in `crates/agent/src/tools/`

**Key Components:**
- `AgentTool` trait - Defines tool interface
- `AnyAgentTool` - Type-erased tool wrapper
- `ToolCallEventStream` - Reports tool execution progress to UI
- `ContextServerRegistry` - Manages external MCP tools

### Tool Availability and Filtering

```
┌───────────────────────────────────────────────────────────┐
│           Thread::enabled_tools(profile, model, cx)       │
│                                                           │
│  Step 1: Gather Built-in Tools                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Available Built-in Tools:                       │    │
│  │  ✓ CopyPathTool       ✓ ListDirectoryTool      │    │
│  │  ✓ CreateDirectoryTool ✓ MovePathTool          │    │
│  │  ✓ DeletePathTool     ✓ NowTool                │    │
│  │  ✓ DiagnosticsTool    ✓ OpenTool               │    │
│  │  ✓ EditFileTool       ✓ ReadFileTool           │    │
│  │  ✓ FetchTool          ✓ TerminalTool           │    │
│  │  ✓ FindPathTool       ✓ ThinkingTool           │    │
│  │  ✓ GrepTool           ✓ WebSearchTool          │    │
│  └─────────────────────────────────────────────────┘    │
│          │                                               │
│          ▼                                               │
│  Filter by provider compatibility:                       │
│    └─ tool.supports_provider(&model.provider_id())       │
│          │                                               │
│          ▼                                               │
│  Filter by profile settings:                             │
│    └─ profile.is_tool_enabled(tool_name)                │
│                                                           │
│  Step 2: Add Context Server Tools                        │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Get running servers from registry               │    │
│  │  └─ For each server:                            │    │
│  │     └─ For each tool in server.tools:           │    │
│  │        └─ Check: profile.is_context_server_     │    │
│  │                  tool_enabled(server_id, name)  │    │
│  └─────────────────────────────────────────────────┘    │
│          │                                               │
│          ▼                                               │
│  Step 3: Handle Name Collisions                          │
│    ├─ Track duplicate names in HashSet                  │
│    └─ Disambiguate with server_id prefix               │
│       (if name + prefix <= 64 chars)                    │
│                                                           │
│  Output: BTreeMap<SharedString, Arc<dyn AnyAgentTool>>  │
└───────────────────────────────────────────────────────────┘
```

### Tool Execution Flow

```
┌───────────────────────────────────┐
│  Model Output: ToolUse Event      │
│  {                                │
│    id: ToolUseId,                 │
│    name: "read_file",             │
│    input: {"path": "src/main.rs"},│
│    is_input_complete: true        │
│  }                                │
└────────┬──────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│      Thread::handle_tool_use_event()                     │
│                                                           │
│  1. Wait for complete input:                             │
│     └─ If !is_input_complete, buffer and wait            │
│                                                           │
│  2. Lookup tool by name:                                 │
│     ├─ Check self.tools (built-in)                       │
│     └─ Check registry.servers() (context servers)        │
│                                                           │
│  3. If tool not found:                                   │
│     └─ Return LanguageModelToolResult with error:        │
│        "No tool named 'X' exists"                        │
│                                                           │
│  4. If tool found:                                       │
│     ├─ Create ToolCallEventStream (for UI updates)       │
│     ├─ Set initial status: InProgress                    │
│     ├─ Parse input JSON to tool's Input type             │
│     └─ Spawn foreground task:                            │
│        tool.run(input, event_stream, cx)                 │
└────────┬──────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────┐
│              Tool Implementation                           │
│              (e.g., ReadFileTool)                          │
│                                                            │
│  async fn run(input, event_stream, cx) -> Result<Output>  │
│  {                                                         │
│    ├─ Perform operations:                                 │
│    │  - Read file, execute command, search web, etc.      │
│    │                                                       │
│    ├─ Emit events to event_stream:                        │
│    │  event_stream.send(ToolCallEvent::Output { ... })    │
│    │  └─> UI updates in real-time                         │
│    │                                                       │
│    └─ Return result:                                      │
│       Ok(LanguageModelToolResultContent::Text(content))   │
│  }                                                         │
└────────┬───────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│         Wrap Result in LanguageModelToolResult          │
│         {                                                │
│           tool_use_id: id,                               │
│           tool_name: "read_file",                        │
│           is_error: false,                               │
│           content: ToolResultContent::Text(file_content),│
│           output: debug JSON                             │
│         }                                                │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│     Add to pending_message.tool_results                 │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│     If StopReason::ToolUse:                             │
│       ├─ pending_message.to_request() creates:          │
│       │  ├─ Assistant message with ToolUse content       │
│       │  └─ User message with ToolResult content         │
│       │                                                   │
│       └─ Send back to model for next iteration           │
└────────┬─────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────┐
│  Model processes    │
│  tool results and   │
│  continues response │
└─────────────────────┘
```

### Tool Trait Interface

```rust
pub trait AgentTool where Self: 'static + Sized {
    // Input/output types
    type Input: Deserialize + Serialize + JsonSchema;
    type Output: Deserialize + Serialize + Into<LanguageModelToolResultContent>;

    // Metadata
    fn name() -> &'static str;
    fn description() -> SharedString;
    fn kind() -> acp::ToolKind;

    // Schema and compatibility
    fn input_schema(format: LanguageModelToolSchemaFormat) -> Schema;
    fn supports_provider(provider: &LanguageModelProviderId) -> bool;

    // UI and execution
    fn initial_title(input: Result<Self::Input, Value>, cx: &mut App) -> SharedString;
    fn run(self: Arc<Self>,
           input: Self::Input,
           event_stream: ToolCallEventStream,
           cx: &mut App) -> Task<Result<Self::Output>>;
    fn replay(input: Self::Input,
              output: Self::Output,
              event_stream: ToolCallEventStream,
              cx: &mut App) -> Result<()>;
}
```

### Prompts Used

- **System Prompt** includes tool descriptions and schemas
  - Tool list rendered in `## Tool Use` section
  - Each tool gets: name, description, JSON schema
  - See [System Prompt Construction Pipeline](#system-prompt-construction-pipeline)

### Data Transformations

| From | To | Transformation |
|------|-----|----------------|
| Tool metadata | JSON Schema | `tool.input_schema()` generates schema |
| `LanguageModelToolUse` | Parsed input | Deserialize JSON to tool's Input type |
| Tool output | `ToolResultContent` | `Into<LanguageModelToolResultContent>` |
| Tool result | Model message | Wrapped in User message with ToolResult role |

### Key Functions

- `Thread::enabled_tools()` (lines 1896-1962) - Filters and collects available tools
- `Thread::handle_tool_use_event()` (lines 1500-1608) - Initiates tool execution
- `AgentTool::run()` - Tool implementation (varies by tool)
- `ToolCallEventStream::send()` - Reports progress to UI

---

## Context Retrieval Pipeline

### Overview

The context retrieval pipeline gathers project information, custom rules, and user preferences to include in the system prompt. This provides the agent with awareness of the project structure, conventions, and user requirements.

**Primary Files:**
- `crates/agent/src/agent.rs` (lines 384-529)
- `crates/prompt_store/src/prompts.rs`
- `crates/prompt_store/src/prompt_store.rs`

**Key Components:**
- `ProjectContext` - Aggregates all project-level context
- `WorktreeContext` - Information for each workspace root
- `PromptStore` - Manages user-defined custom prompts

### ProjectContext Construction Flow

```
┌───────────────────────────────────────────────────────────┐
│    NativeAgent::build_project_context()                   │
│    (Spawned as async task)                                │
└────────┬──────────────────────────────────────────────────┘
         │
         ├─────────────────────────┬─────────────────────────┐
         │                         │                         │
         ▼                         ▼                         ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  Worktree 1      │   │  Worktree 2      │   │  Worktree N      │
│  Context         │   │  Context         │   │  Context         │
│                  │   │                  │   │                  │
│ ┌──────────────┐ │   │ ┌──────────────┐ │   │ ┌──────────────┐ │
│ │ Root name:   │ │   │ │ Root name:   │ │   │ │ Root name:   │ │
│ │  "zed"       │ │   │ │  "extension" │ │   │ │  "docs"      │ │
│ └──────────────┘ │   │ └──────────────┘ │   │ └──────────────┘ │
│                  │   │                  │   │                  │
│ ┌──────────────┐ │   │ ┌──────────────┐ │   │ ┌──────────────┐ │
│ │ Abs path:    │ │   │ │ Abs path:    │ │   │ │ Abs path:    │ │
│ │  /home/user/ │ │   │ │  /home/user/ │ │   │ │  /home/user/ │ │
│ │  zed         │ │   │ │  ext         │ │   │ │  docs        │ │
│ └──────────────┘ │   │ └──────────────┘ │   │ └──────────────┘ │
│                  │   │                  │   │                  │
│ ┌──────────────┐ │   │ ┌──────────────┐ │   │ ┌──────────────┐ │
│ │ Rules file:  │ │   │ │ Rules file:  │ │   │ │ Rules file:  │ │
│ │  Search for: │ │   │ │  Search for: │ │   │ │  Search for: │ │
│ │  - .rules    │ │   │ │  - .rules    │ │   │ │  - .rules    │ │
│ │  - .cursor*  │ │   │ │  - .cursor*  │ │   │ │  - .cursor*  │ │
│ │  - CLAUDE.md │ │   │ │  - CLAUDE.md │ │   │ │  - CLAUDE.md │ │
│ │  - etc.      │ │   │ │  - etc.      │ │   │ │  - etc.      │ │
│ │              │ │   │ │              │ │   │ │              │ │
│ │ → Found:     │ │   │ │ → Found:     │ │   │ │ → None found │ │
│ │   CLAUDE.md  │ │   │ │   .rules     │ │   │ │              │ │
│ └──────────────┘ │   │ └──────────────┘ │   │ └──────────────┘ │
└──────────────────┘   └──────────────────┘   └──────────────────┘
         │                         │                         │
         └─────────────────────────┴─────────────────────────┘
                                   │
                                   ▼
         ┌─────────────────────────────────────────────────────┐
         │      Load User Default Prompts                      │
         │      (from PromptStore)                             │
         │                                                     │
         │  prompt_store.default_prompt_metadata()             │
         │    └─ For each default prompt:                     │
         │       ├─ Load prompt title and contents            │
         │       └─ Wrap in UserRulesContext                  │
         └─────────────────────────────────────────────────────┘
                                   │
                                   ▼
         ┌─────────────────────────────────────────────────────┐
         │      Combine into ProjectContext                    │
         │      {                                              │
         │        worktrees: Vec<WorktreeContext>,             │
         │        has_rules: bool,                             │
         │        user_rules: Vec<UserRulesContext>,           │
         │        has_user_rules: bool,                        │
         │        os: String,                                  │
         │        shell: String                                │
         │      }                                              │
         └─────────────────────────────────────────────────────┘
                                   │
                                   ▼
         ┌─────────────────────────────────────────────────────┐
         │      Store in NativeAgent.project_context           │
         │      (watch::channel for updates)                   │
         └─────────────────────────────────────────────────────┘
```

### Rules File Search Order

The agent searches for rules files in this order (first match wins):

```
RULES_FILE_NAMES = [
  ".rules",
  ".cursorrules",
  ".windsurfrules",
  ".clinerules",
  ".github/copilot-instructions.md",
  "CLAUDE.md",
  "AGENT.md",
  "AGENTS.md",
  "GEMINI.md"
]
```

### Context Update Triggers

The ProjectContext is automatically rebuilt when:

```
┌──────────────────────────────────────────┐
│  Project Events                          │
├──────────────────────────────────────────┤
│  • WorktreeAdded                         │
│  • WorktreeRemoved                       │
│  • WorktreeUpdatedEntries (if rules file)│
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  PromptStore Events                      │
├──────────────────────────────────────────┤
│  • PromptsUpdatedEvent                   │
└────────┬─────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│  Rebuild ProjectContext                  │
│  • Spawns async task                     │
│  • Updates watch::channel                │
│  • Next conversation turn uses new ctx   │
└──────────────────────────────────────────┘
```

### Message Context Mentions

User messages can include mentions that add context:

```
User Input: "@src/main.rs Fix the bug in #calculate_total"
                ↓
┌────────────────────────────────────────────────────────┐
│  Parse Mentions:                                       │
│  ├─ File: src/main.rs                                  │
│  └─ Symbol: #calculate_total                           │
└────────┬───────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────┐
│  Resolve and Format:                                   │
│                                                        │
│  <context>                                             │
│  <files>                                               │
│  ```src/main.rs#L1-50                                  │
│  fn calculate_total(price: f64, qty: i32) -> f64 {    │
│    // ... file contents ...                            │
│  }                                                      │
│  ```                                                   │
│  </files>                                              │
│                                                        │
│  <symbols>                                             │
│  ```src/main.rs#L42-48                                 │
│  fn calculate_total(price: f64, qty: i32) -> f64 {    │
│    price * qty as f64                                  │
│  }                                                      │
│  ```                                                   │
│  </symbols>                                            │
│  </context>                                            │
└────────────────────────────────────────────────────────┘
```

### Mention Types and Formatting

| Mention Type | Context Section | Format |
|--------------|-----------------|--------|
| File URI | `<files>` | Markdown code block with file path |
| Directory | `<directories>` | Plain text path |
| Symbol | `<symbols>` | Markdown code block with path:line |
| Selection | `<selections>` | Markdown code block with path:line |
| Thread | `<threads>` | Plain text reference |
| Fetch URL | `<fetched_urls>` | URL + fetched content |
| Rule | `<user_rules>` | Code block with rule content |

### Prompts Used

- Custom rules files (`.rules`, `CLAUDE.md`, etc.) are included in system prompt
- User default prompts from PromptStore included as user rules
- See [System Prompt Construction Pipeline](#system-prompt-construction-pipeline)

### Key Functions

- `NativeAgent::build_project_context()` (lines 384-450) - Main builder
- `load_worktree_info_for_system_prompt()` (lines 452-500) - Per-worktree info
- `load_worktree_rules_file()` (lines 502-529) - Rules file loading
- `maintain_project_context()` (lines 568-598) - Auto-update task
- `UserMessage::to_context_section()` (thread.rs lines 200-354) - Formats mentions

---

## System Prompt Construction Pipeline

### Overview

The system prompt is the foundation of every conversation turn, providing the agent with its identity, capabilities, project context, and user instructions. It's constructed by rendering the `system_prompt.hbs` template with rich contextual data.

**Primary Files:**
- `crates/agent/src/templates/system_prompt.hbs` - Template
- `crates/agent/src/templates.rs` - Template rendering
- `crates/agent/src/thread.rs` (lines 1968-1990) - Template data preparation

### Construction Flow

```
┌───────────────────────────────────────────────────────────┐
│         Thread::build_request_messages()                  │
│         (Called on every turn)                            │
└────────┬──────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│         Prepare Template Data                             │
│                                                           │
│  ProjectContext (from NativeAgent):                       │
│    ├─ worktrees: Vec<WorktreeContext>                    │
│    │  └─ For each worktree:                              │
│    │     ├─ root_name: String                            │
│    │     ├─ abs_path: Arc<Path>                          │
│    │     └─ rules_file: Option<RulesFileContext>         │
│    │        ├─ path_in_worktree: Arc<RelPath>            │
│    │        └─ text: String                              │
│    │                                                      │
│    ├─ has_rules: bool                                    │
│    ├─ user_rules: Vec<UserRulesContext>                  │
│    │  └─ For each prompt:                                │
│    │     ├─ title: Option<String>                        │
│    │     └─ contents: String                             │
│    │                                                      │
│    ├─ has_user_rules: bool                               │
│    ├─ os: String (e.g., "Linux", "macOS")               │
│    └─ shell: String (e.g., "bash", "zsh")               │
│                                                           │
│  Available Tools:                                         │
│    └─ available_tools: Vec<SharedString>                 │
│       (tool names from enabled_tools)                    │
│                                                           │
│  Model Information:                                       │
│    └─ model_name: Option<String>                         │
│       (e.g., "claude-opus-4-20250514")                   │
└────────┬──────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│         Render Template: system_prompt.hbs                │
│         (via Handlebars)                                  │
└────────┬──────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│         Generated System Prompt Structure                 │
│                                                           │
│  1. Identity and Communication                            │
│     └─ "You are a highly skilled software engineer..."   │
│                                                           │
│  2. Tool Use Instructions (if tools available)            │
│     ├─ Tool schema adherence                             │
│     ├─ Required vs available tools                       │
│     └─ Tool-specific guidelines                          │
│                                                           │
│  3. Searching and Reading (if tools available)            │
│     ├─ Project root directories list                     │
│     ├─ File path guidelines                              │
│     └─ grep tool recommendations                         │
│                                                           │
│  4. Code Block Formatting                                 │
│     ├─ Path-based format requirement                     │
│     ├─ Examples of correct format                        │
│     └─ Examples of incorrect formats to avoid            │
│                                                           │
│  5. Fixing Diagnostics (if tools available)               │
│     └─ Guidelines for diagnostic resolution              │
│                                                           │
│  6. Debugging (if tools available)                        │
│     └─ Best practices for debugging                      │
│                                                           │
│  7. Calling External APIs                                 │
│     └─ API usage and security guidelines                 │
│                                                           │
│  8. System Information                                    │
│     ├─ Operating System: {{os}}                          │
│     └─ Default Shell: {{shell}}                          │
│                                                           │
│  9. Model Information (if available)                      │
│     └─ "You are powered by {{model_name}}"               │
│                                                           │
│  10. User's Custom Instructions (if available)            │
│      ├─ Project rules from rules files                   │
│      └─ User default prompts from PromptStore            │
└───────────────────────────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│         Add to LanguageModelRequest                       │
│         messages[0] = System message (rendered prompt)    │
└───────────────────────────────────────────────────────────┘
```

### Template Conditional Sections

The system prompt adapts based on available data:

```
{{#if (gt (len available_tools) 0)}}
  ## Tool Use
  [Tool usage instructions]

  ## Searching and Reading
  [File search guidelines]
  {{#if (contains available_tools 'grep')}}
    [grep-specific recommendations]
  {{/if}}

  ## Fixing Diagnostics
  [Diagnostic handling]

  ## Debugging
  [Debugging guidelines]
{{else}}
  [No-tools mode instructions]
{{/if}}

{{#if model_name}}
  ## Model Information
  You are powered by {{model_name}}
{{/if}}

{{#if (or has_rules has_user_rules)}}
  ## User's Custom Instructions

  {{#if has_rules}}
    [Project rules from worktrees]
  {{/if}}

  {{#if has_user_rules}}
    [User default prompts]
  {{/if}}
{{/if}}
```

### Example Rendered Sections

**Project Roots:**
```markdown
If appropriate, use tool calls to explore the current project, which contains the following root directories:

- `/home/user/zed`
- `/home/user/zed-extensions`
```

**Custom Rules:**
```markdown
## User's Custom Instructions

There are project rules that apply to these root directories:
`zed/CLAUDE.md`:
``````
# Rust coding guidelines
* Prioritize code correctness and clarity.
* Do not write organizational comments.
...
``````
```

**Available Tools:**
```markdown
## Tool Use

1. Make sure to adhere to the tools schema.
2. Provide every required argument.
3. DO NOT use tools to access items that are already available in the context section.
...
```

### Template Rendering Process

```
Template Variables (HashMap)
  ├─ available_tools: Vec<SharedString>
  ├─ worktrees: Vec<WorktreeContext>
  ├─ os: String
  ├─ shell: String
  ├─ model_name: Option<String>
  ├─ has_rules: bool
  ├─ has_user_rules: bool
  └─ user_rules: Vec<UserRulesContext>
         │
         ▼
    Handlebars::render("system_prompt", &data)
         │
         ├─ Process conditionals: {{#if}}, {{#each}}
         ├─ Interpolate variables: {{os}}, {{shell}}
         ├─ Apply helpers: (gt), (len), (contains), (or)
         └─ Escape only when needed (triple braces for raw: {{{text}}})
         │
         ▼
    Complete System Prompt String
```

### Prompts Used

1. **System Prompt Template** (`system_prompt.hbs`)
   - Comprehensive agent instructions
   - Dynamically includes project context
   - Adapts based on tool availability

### Data Sources

| Data | Source | Purpose |
|------|--------|---------|
| Worktree info | Project's visible worktrees | File path validation, project structure |
| Rules files | `.rules`, `CLAUDE.md`, etc. | Project-specific coding conventions |
| User prompts | PromptStore default prompts | User preferences and instructions |
| Available tools | Filtered tool list | Tool usage instructions |
| OS/Shell | System information | Environment-specific commands |
| Model name | Current language model | Model-specific capabilities |

### Key Functions

- `SystemPromptTemplate::render()` (templates.rs) - Template rendering
- `Thread::build_request_messages()` (lines 1968-1990) - Data preparation
- `NativeAgent::build_project_context()` - Context gathering
- `Handlebars::render()` - Template engine

---

