# Amazon Q API Integration and Conversation Memory

In this document, we explore how Amazon Q CLI integrates with backend AI services and manages conversation memory. Understanding these aspects is crucial for our Project Wingman extensions.

## API Integration

### Our Custom Streaming Clients

Unlike many AI applications that directly use public APIs, we've discovered that Amazon Q CLI uses custom streaming clients that provide specialized functionality:

```
┌─────────────────┐     ┌─────────────────────────────┐
│                 │     │                             │
│  Amazon Q CLI   │────►│ amzn-codewhisperer-client   │
│                 │     │                             │
└─────────────────┘     └─────────────────────────────┘
        │
        │               ┌─────────────────────────────┐
        │               │                             │
        └──────────────►│ amzn-qdeveloper-client      │
                        │                             │
                        └─────────────────────────────┘
```

We use two different clients depending on the environment:

1. **amzn-codewhisperer-streaming-client**: Our standard client for most environments
2. **amzn-qdeveloper-streaming-client**: Used when running in AWS CloudShell or when specified by environment variables

The selection logic is implemented in `fig_api_client/src/clients/streaming_client.rs`:

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

### Our Service Endpoints

Our clients connect to these managed service endpoints:
- CodeWhisperer: `https://codewhisperer.us-east-1.amazonaws.com`
- Q Developer: `https://q.us-east-1.amazonaws.com`

These endpoints provide access to the underlying AI capabilities while adding specialized features like tool use and context management.

### Our Model Abstraction Layer

We've found that our code doesn't directly reference specific Bedrock models (like Claude or Anthropic). This suggests we're using an abstraction layer where:

1. Model selection happens on the server side
2. Our client communicates with Amazon's managed services, which then interact with the underlying models

This architecture gives us several advantages:
- We can control model selection and parameters centrally
- We provide a consistent API regardless of the underlying model
- We implement specialized features that may not be directly supported by all models

## Our Conversation Memory Management

### Conversation State

We manage conversation memory primarily through the `ConversationState` struct in `q_chat/src/conversation_state.rs`:

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

This structure handles several key aspects of memory management:

1. We maintain conversation history as a queue of user and assistant message pairs
2. We handle tool use results and integrate them into the conversation
3. We enforce conversation invariants like maximum history length
4. We prepare the conversation state for sending to the API

### Context Management

Our `ContextManager` in `q_chat/src/context.rs` provides additional context capabilities:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│ ConversationState│────►│ ContextManager  │────►│ Context Files   │
│                 │     │                 │     │                 │
└─────────────────┘     └────────┬────────┘     └─────────────────┘
                                 │
                                 │
                        ┌────────▼────────┐
                        │                 │
                        │ Context Hooks   │
                        │                 │
                        └─────────────────┘
```

With this system, we:
1. Manage profiles for different context configurations
2. Load context files from the filesystem
3. Execute context hooks to dynamically generate context
4. Provide mechanisms for switching between profiles

Our context configuration is stored in:
- Global configuration: `~/.amazonq/chat/global_context.json`
- Profile configurations: `~/.amazonq/chat/profiles/<profile>/context.json`

### Memory Persistence Limitations

Currently, we maintain conversation history in memory during a session, but we don't automatically persist it between sessions. We do have some limited persistence mechanisms:

1. Our `ContextManager` allows for persistent context across sessions through profiles
2. Context files and hooks provide a way to inject consistent context into new conversations
3. We support summarizing conversations to preserve key information while reducing token usage

### Our History Management Methods

We implement several methods in `ConversationState` to manage history:

```rust
// Ensures the conversation history follows required patterns and doesn't exceed limits
pub fn enforce_conversation_invariants(&mut self) {
    // Implementation details...
}

// Clears the conversation history while optionally preserving summaries
pub fn clear(&mut self, preserve_summary: bool) {
    // Implementation details...
}

// Adds new assistant responses to the history
pub fn push_assistant_message(&mut self, message: AssistantMessage) {
    // Implementation details...
}

// Incorporates tool execution results into the conversation
pub fn add_tool_results(&mut self, tool_results: Vec<ToolUseResult>) {
    // Implementation details...
}
```

## Implications for Our Project Wingman Extensions

Based on our analysis of the API integration and conversation memory systems, we've identified several key implications for our Project Wingman extensions:

1. **API Integration**: We'll extend the existing streaming clients rather than directly interacting with Bedrock APIs. This ensures compatibility with the current architecture while adding our new capabilities.

2. **Context Graph**: Our context graph system will integrate with the existing `ContextManager`, extending it to support structured knowledge representation beyond simple file-based context.

3. **Tool Registry**: Our MCP-based tool registry will work with the conversation state management, ensuring tools can access and update the context graph.

4. **Conversation History System**: We'll build our new persistent conversation history system on top of the existing conversation state structure, adding local storage capabilities while maintaining compatibility with the current memory management approach.

By building on these existing systems rather than replacing them, we ensure our extensions integrate seamlessly with the Amazon Q CLI while adding powerful new capabilities.
