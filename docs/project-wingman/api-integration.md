# Amazon Q API Integration and Conversation Memory

This document provides an analysis of how Amazon Q CLI integrates with backend AI services and manages conversation memory.

## API Integration

### Custom Streaming Clients

The Amazon Q CLI doesn't directly use the public Amazon Bedrock API. Instead, it uses two custom streaming clients:

1. **amzn-codewhisperer-streaming-client**: Used in standard environments
2. **amzn-qdeveloper-streaming-client**: Used in AWS CloudShell or when specified by environment variables

The client selection logic is implemented in `fig_api_client/src/clients/streaming_client.rs`:

```rust
pub async fn new() -> Result<Self, Error> {
    let client = if fig_util::system_info::in_cloudshell()
        || std::env::var("Q_USE_SENDMESSAGE").is_ok_and(|v| !v.is_empty())
    {
        Self::new_qdeveloper_client(&Endpoint::load_q()).await?
    } else {
        Self::new_codewhisperer_client(&Endpoint::load_codewhisperer()).await
    };
    Ok(client)
}
```

### Endpoints

The clients connect to these endpoints:
- CodeWhisperer: `https://codewhisperer.us-east-1.amazonaws.com`
- Q Developer: `https://q.us-east-1.amazonaws.com`

### Model Abstraction

The code doesn't directly reference specific Bedrock models (like Claude or Anthropic), suggesting that:

1. The model selection happens on the server side
2. The client communicates with Amazon's managed services, which in turn interact with the underlying models

This architecture allows Amazon to:
- Control the model selection and parameters
- Provide a consistent API regardless of the underlying model
- Implement specialized features like tool use that may not be directly supported by all Bedrock models

## Conversation Memory Management

### Conversation State

The conversation memory is primarily managed by the `ConversationState` struct in `q_chat/src/conversation_state.rs`. This structure:

1. Maintains the conversation history as a queue of user and assistant message pairs
2. Handles tool use results and their integration into the conversation
3. Enforces conversation invariants like maximum history length
4. Prepares the conversation state for sending to the API

Key components of the `ConversationState` struct:

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

### Context Management

The `ContextManager` in `q_chat/src/context.rs` handles additional context for conversations:

1. Manages profiles for different context configurations
2. Loads context files from the filesystem
3. Executes context hooks to dynamically generate context
4. Provides mechanisms for switching between profiles

The context configuration is stored in:
- Global configuration: `~/.amazonq/chat/global_context.json`
- Profile configurations: `~/.amazonq/chat/profiles/<profile>/context.json`

### Memory Persistence

Conversation history is maintained in memory during a session but is not automatically persisted between sessions. However:

1. The `ContextManager` allows for persistent context across sessions through profiles
2. Context files and hooks provide a way to inject consistent context into new conversations
3. The system supports summarizing conversations to preserve key information while reducing token usage

### History Management

The `ConversationState` implements several methods to manage history:

1. `enforce_conversation_invariants()`: Ensures the conversation history follows required patterns and doesn't exceed limits
2. `clear()`: Clears the conversation history while optionally preserving summaries
3. `push_assistant_message()`: Adds new assistant responses to the history
4. `add_tool_results()`: Incorporates tool execution results into the conversation

## Implications for Project Wingman

For Project Wingman, this architecture means:

1. **API Integration**: We'll extend the existing streaming clients rather than directly interacting with Bedrock APIs
2. **Context Graph**: Our context graph system will need to integrate with the existing `ContextManager`
3. **Tool Registry**: The MCP implementation and tool system will need to work with the conversation state management
4. **Memory Management**: Our persistent memory features will build on top of the existing conversation state structure

The context graph system will be a significant extension, as the current system primarily manages file-based context rather than a structured knowledge graph.
