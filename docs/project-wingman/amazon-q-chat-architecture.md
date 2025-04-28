# Amazon Q Chat Architecture

In this document, we dive deeper into the Amazon Q CLI chat functionality, examining its code architecture, component interactions, and sequence flows. This understanding will guide our Project Wingman extensions.

## Core Components

The Amazon Q chat functionality is built around four primary components that work together to create the interactive experience:

1. **ChatContext**: The central structure that orchestrates the entire chat session
2. **ConversationState**: Manages our conversation history and context
3. **Tool System**: Provides the framework for executing tools requested by the model
4. **StreamingClient**: Handles our communication with the backend AI services

These components form a cohesive system that allows us to interact with powerful AI models through a simple terminal interface.

## Code Architecture

Understanding the code organization helps us navigate the codebase and identify where to implement our extensions.

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

The `ChatContext` struct serves as the conductor of our chat experience:

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

This structure handles:
- Managing our user input and output
- Controlling the conversation flow
- Processing commands like `/context` and `/clear`
- Coordinating tool execution
- Communicating with the model via StreamingClient

For our Project Wingman extensions, we'll need to extend this structure to handle our new tool registry and context graph features.

#### ConversationState (conversation_state.rs)

The `ConversationState` struct maintains the memory of our conversations:

```rust
pub struct ConversationState {
    conversation_id: String,
    next_message: Option<UserMessage>,
    history: VecDeque<(UserMessage, AssistantMessage)>,
    valid_history_range: (usize, usize),
    transcript: VecDeque<String>,
    tools: HashMap<ToolOrigin, Vec<Tool>>,
    context_manager: Option<ContextManager>,
    context_message_length: Option<usize>,
    latest_summary: Option<String>,
    updates: Option<SharedWriter>,
}
```

This structure is responsible for:
- Storing our conversation history
- Managing context files through the ContextManager
- Preparing the conversation state for API requests
- Handling tool results

Our context graph implementation will integrate closely with this structure to provide persistent memory across conversations.

#### Tool System (tools/mod.rs)

The tool system is implemented as an enum with variants for each tool type:

```rust
pub enum Tool {
    FsRead(FsRead),
    FsWrite(FsWrite),
    ExecuteBash(ExecuteBash),
    UseAws(UseAws),
    GhIssue(GhIssue),
    Custom(CustomTool),
}
```

This system:
- Defines the interface for all tools
- Validates tool parameters before execution
- Executes tools and captures their output
- Formats tool results for the model

Our tool registry will extend this system to support dynamically loaded tools from YAML manifests.

#### StreamingClient (fig_api_client/src/clients/streaming_client.rs)

The `StreamingClient` handles our communication with the backend AI services:

```rust
pub struct StreamingClient(inner::Inner);

enum Inner {
    Codewhisperer(CodewhispererStreamingClient),
    QDeveloper(QDeveloperStreamingClient),
    Mock(Arc<Mutex<std::vec::IntoIter<Vec<ChatResponseStream>>>>),
}
```

This client:
- Sends our conversation state to the model
- Receives streaming responses
- Handles tool use events
- Manages API errors

## Sequence Flows

Understanding the flow of information through the system helps us identify integration points for our extensions.

### Chat Initialization Flow

```
┌──────────┐          ┌────────────┐          ┌─────────────────┐          ┌───────────────┐
│ User/CLI │          │ ChatContext │          │ StreamingClient │          │ AI Model      │
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
│ User/CLI │          │ ChatContext │          │ StreamingClient │          │ AI Model      │
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
│ User/CLI │          │ ChatContext │          │ Tool System     │          │ AI Model      │
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

The chat functionality is implemented as a state machine that drives the conversation flow:

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

This state machine handles:
- Prompting the user for input
- Processing user commands and messages
- Validating and executing tools
- Processing model responses
- Managing conversation history

## Integration Points for Project Wingman

Based on our analysis of the Amazon Q chat architecture, we've identified several key integration points for our Project Wingman extensions:

1. **Tool Registry**: We'll extend the `Tool` enum and the tool discovery mechanism to support our YAML manifest format.

2. **Context Graph**: We'll enhance the `ConversationState` and `ContextManager` to incorporate our persistent knowledge graph.

3. **Domain-Scoped Browsing**: We'll implement policy enforcement in the tool validation phase to ensure secure access to resources.

4. **Conversation History**: We'll extend the `ConversationState` to persist conversations to our local storage system.

By understanding the existing architecture, we can ensure our extensions integrate seamlessly with the Amazon Q CLI while adding powerful new capabilities.
