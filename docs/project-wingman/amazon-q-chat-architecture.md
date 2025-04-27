# Amazon Q Chat Architecture

This document provides a detailed analysis of the Amazon Q CLI chat functionality, including code architecture, component interactions, and sequence diagrams.

## Core Components

The Amazon Q chat functionality is primarily implemented in the `q_chat` crate, with the following key components:

1. **ChatContext**: The central structure that manages the chat session
2. **ConversationState**: Manages conversation history and context
3. **Tool System**: Framework for executing tools requested by the model
4. **StreamingClient**: Handles communication with the Amazon Bedrock API

## Code Architecture

### Main Components and Their Relationships

```
q_chat
├── lib.rs               # Main implementation with ChatContext
├── cli.rs               # CLI argument parsing
├── command.rs           # Command parsing and execution
├── context.rs           # Context management
├── conversation_state.rs # Conversation history and state
├── tools/               # Tool implementations
│   ├── mod.rs           # Tool system architecture
│   ├── tool_index.json  # Tool specifications
│   ├── execute_bash.rs  # Bash command execution
│   ├── fs_read.rs       # File system reading
│   ├── fs_write.rs      # File system writing
│   ├── use_aws.rs       # AWS CLI integration
│   └── gh_issue.rs      # GitHub issue reporting
└── util.rs              # Utility functions
```

### Key Structures and Their Responsibilities

#### ChatContext (lib.rs)

The `ChatContext` struct is the central component that manages the chat session:

```rust
pub struct ChatContext {
    ctx: Arc<Context>,
    settings: Settings,
    state: State,
    output: SharedWriter,
    initial_input: Option<String>,
    input_source: InputSource,
    interactive: bool,
    client: StreamingClient,
    terminal_width_provider: fn() -> Option<usize>,
    spinner: Option<Spinner>,
    conversation_state: ConversationState,
    tool_permissions: ToolPermissions,
    tool_use_telemetry_events: HashMap<String, ToolUseEventBuilder>,
    tool_use_status: ToolUseStatus,
    failed_request_ids: Vec<String>,
}
```

Responsibilities:
- Managing user input and output
- Handling the conversation flow
- Processing commands
- Executing tools
- Communicating with the model via StreamingClient

#### ConversationState (conversation_state.rs)

The `ConversationState` struct manages the conversation history and context:

```rust
pub struct ConversationState {
    conversation_id: String,
    history: Vec<ChatMessage>,
    transcript: String,
    tools: Vec<Tool>,
    context_manager: Option<ContextManager>,
    next_user_message: Option<String>,
    current_profile: String,
    output: Option<SharedWriter>,
}
```

Responsibilities:
- Storing conversation history
- Managing context files
- Preparing conversation state for API requests
- Handling tool results

#### Tool System (tools/mod.rs)

The tool system is implemented as an enum with variants for each tool type:

```rust
pub enum Tool {
    FsRead(FsRead),
    FsWrite(FsWrite),
    ExecuteBash(ExecuteBash),
    UseAws(UseAws),
    GhIssue(GhIssue),
}
```

Responsibilities:
- Defining the interface for tools
- Validating tool parameters
- Executing tools
- Formatting tool results

#### StreamingClient (fig_api_client/src/clients/streaming_client.rs)

The `StreamingClient` handles communication with the Amazon Bedrock API:

```rust
pub struct StreamingClient(inner::Inner);

enum Inner {
    Codewhisperer(CodewhispererStreamingClient),
    QDeveloper(QDeveloperStreamingClient),
    Mock(Arc<Mutex<std::vec::IntoIter<Vec<ChatResponseStream>>>>),
}
```

Responsibilities:
- Sending conversation state to the model
- Receiving streaming responses
- Handling tool use events
- Managing API errors

## Sequence Diagrams

### Chat Initialization Flow

```
┌──────────┐          ┌────────────┐          ┌─────────────────┐          ┌───────────────┐
│ User/CLI │          │ ChatContext │          │ StreamingClient │          │ Bedrock Model │
└────┬─────┘          └──────┬─────┘          └────────┬────────┘          └───────┬───────┘
     │                       │                         │                           │
     │ launch_chat()         │                         │                           │
     │──────────────────────>│                         │                           │
     │                       │                         │                           │
     │                       │ new()                   │                           │
     │                       │─────────────────────────>                           │
     │                       │                         │                           │
     │                       │ try_chat()              │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │        │ Initialize     │                           │
     │                       │        │ conversation   │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
     │                       │ Display welcome         │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
     │                       │ Enter ChatState::       │                           │
     │                       │ PromptUser              │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
```

### User Input Processing Flow

```
┌──────────┐          ┌────────────┐          ┌─────────────────┐          ┌───────────────┐
│ User/CLI │          │ ChatContext │          │ StreamingClient │          │ Bedrock Model │
└────┬─────┘          └──────┬─────┘          └────────┬────────┘          └───────┬───────┘
     │                       │                         │                           │
     │ Enter prompt          │                         │                           │
     │──────────────────────>│                         │                           │
     │                       │                         │                           │
     │                       │ Parse as Command        │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
     │                       │ If not a command:       │                           │
     │                       │ set_next_user_message() │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
     │                       │ as_sendable_conversation_state()                    │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
     │                       │ send_message()          │                           │
     │                       │─────────────────────────>                           │
     │                       │                         │                           │
     │                       │                         │ API Request              │
     │                       │                         │──────────────────────────>│
     │                       │                         │                           │
     │                       │                         │ Streaming Response        │
     │                       │                         │<──────────────────────────│
     │                       │                         │                           │
     │                       │ HandleResponseStream    │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
```

### Tool Execution Flow

```
┌──────────┐          ┌────────────┐          ┌─────────────────┐          ┌───────────────┐
│ User/CLI │          │ ChatContext │          │ Tool System     │          │ Bedrock Model │
└────┬─────┘          └──────┬─────┘          └────────┬────────┘          └───────┬───────┘
     │                       │                         │                           │
     │                       │                         │                           │
     │                       │ Receive ToolUseEvent    │                           │
     │                       │<───────────────────────────────────────────────────│
     │                       │                         │                           │
     │                       │ ValidateTools           │                           │
     │                       │─────────────────────────>                           │
     │                       │                         │                           │
     │                       │ Tool validation result  │                           │
     │                       │<─────────────────────────                           │
     │                       │                         │                           │
     │                       │ If tool requires        │                           │
     │                       │ acceptance:             │                           │
     │                       │ Prompt user             │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │<──────────────────────┘        │                │                           │
     │                                │                │                           │
     │ User approves tool             │                │                           │
     │────────────────────────────────>                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
     │                       │ ExecuteTools            │                           │
     │                       │─────────────────────────>                           │
     │                       │                         │                           │
     │                       │ Tool execution result   │                           │
     │                       │<─────────────────────────                           │
     │                       │                         │                           │
     │                       │ add_tool_results()      │                           │
     │                       │────────┐                │                           │
     │                       │        │                │                           │
     │                       │<───────┘                │                           │
     │                       │                         │                           │
     │                       │ send_message() with     │                           │
     │                       │ tool results            │                           │
     │                       │───────────────────────────────────────────────────>│
     │                       │                         │                           │
     │                       │ Continue conversation   │                           │
     │                       │<───────────────────────────────────────────────────│
     │                       │                         │                           │
```

## State Machine

The chat functionality is implemented as a state machine with the following states:

```rust
enum ChatState {
    PromptUser {
        tool_uses: Option<Vec<QueuedTool>>,
        pending_tool_index: Option<usize>,
        skip_printing_tools: bool,
    },
    HandleInput {
        input: String,
        tool_uses: Option<Vec<QueuedTool>>,
        pending_tool_index: Option<usize>,
    },
    ValidateTools(Vec<AssistantToolUse>),
    ExecuteTools(Vec<QueuedTool>),
    HandleResponseStream(SendMessageOutput),
    CompactHistory {
        tool_uses: Option<Vec<QueuedTool>>,
        pending_tool_index: Option<usize>,
        prompt: Option<String>,
        show_summary: bool,
        help: bool,
    },
    Exit,
}
```

This state machine drives the conversation flow, handling user input, model responses, and tool execution.

## API Integration

Amazon Q CLI integrates with Amazon Bedrock through the following components:

1. **StreamingClient**: Handles API communication
2. **ConversationState**: Prepares the conversation for the API
3. **ChatResponseStream**: Processes streaming responses from the API

The API client automatically selects between:
- `amzn-codewhisperer-streaming-client` for standard environments
- `amzn-qdeveloper-streaming-client` for AWS CloudShell or when specified by environment variables

## Context Management

The context management system allows for:

1. **File Context**: Adding files to the conversation context
2. **Profile Management**: Switching between different context profiles
3. **Context Hooks**: Running commands to dynamically add context

This system is implemented in the `context.rs` file and managed by the `ContextManager` class.

## Tool System

The tool system in Amazon Q CLI follows a plugin-like architecture:

1. **Tool Enum**: Represents different tool types
2. **Tool Implementations**: Individual tool functionality
3. **Tool Specifications**: JSON schema for each tool
4. **Tool Permissions**: Controls which tools require user approval

Tools are executed through a standardized flow:
1. Model requests a tool use
2. Tool is validated
3. User approves (if required)
4. Tool is executed
5. Results are sent back to the model
