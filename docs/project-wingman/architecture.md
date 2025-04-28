# Amazon Q Developer CLI Architecture

In this document, we explore the architecture of the Amazon Q Developer CLI, focusing on the `q chat` functionality that forms the foundation for our Project Wingman extensions.

## High-Level Architecture

The Amazon Q Developer CLI is organized into several interconnected components that work together to provide AI-powered assistance in the terminal. Understanding this structure helps us identify where and how to implement our extensions.

```
amazon-q-developer-cli/
├── crates/                  # Rust implementation
│   ├── q_cli/               # Main CLI application
│   ├── q_chat/              # Chat functionality
│   ├── mcp_client/          # Model-Client-Provider system
│   └── ...                  # Supporting crates
├── packages/                # TypeScript UI components
│   ├── autocomplete/        # Autocomplete React app
│   └── dashboard-app/       # Dashboard interface
└── extensions/              # IDE integrations
    ├── vscode/              # VSCode plugin
    └── jetbrains/           # JetBrains IDE plugin
```

Our Project Wingman extensions will primarily focus on the `q_chat` and `mcp_client` crates, where we'll implement our tool registry, context graph, and domain-scoped browsing capabilities.

## Q Chat Implementation

The `q chat` functionality forms the core of our extension target. It's built around several key components that we'll need to understand and extend.

### Core Components

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  ChatContext    │────►│ ConversationState│────►│ StreamingClient │
│                 │     │                 │     │                 │
└────────┬────────┘     └────────┬────────┘     └─────────────────┘
         │                       │
         │                       │
         │                       │
┌────────▼────────┐     ┌────────▼────────┐
│                 │     │                 │
│  Tool System    │     │ Context Manager │
│                 │     │                 │
└─────────────────┘     └─────────────────┘
```

1. **ChatContext**: The central structure that manages the chat session, coordinating between user input, model responses, and tool execution.

2. **ConversationState**: Maintains the conversation history, tracks context, and prepares the state that's sent to the model. This is where we'll integrate our context graph.

3. **Tool System**: The extensible framework for executing tools requested by the model. We'll expand this with our tool registry system.

4. **StreamingClient**: Handles communication with the backend AI services. We'll need to ensure our extensions work with this client.

### Data Flow

When a user interacts with `q chat`, the following sequence occurs:

1. We capture user input via the terminal
2. We process the input and send it to the model via the StreamingClient
3. The model responds with text and/or tool use requests
4. We validate tool use requests, get user approval if needed, and execute the tools
5. We send results back to the model for further processing
6. We display the final response to the user

This flow provides several integration points for our Project Wingman extensions, particularly around tool execution and context management.

## API and Model Integration

Amazon Q CLI doesn't directly use the public Amazon Bedrock API. Instead, it uses custom streaming clients:

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

The client selection happens in `fig_api_client/src/clients/streaming_client.rs`:

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

These clients connect to Amazon's managed services, which in turn interact with the underlying AI models. This architecture allows Amazon to control model selection and parameters while providing a consistent API.

## MCP System

The Model-Client-Provider (MCP) system is a key component we'll leverage for our tool registry. It allows external processes to register as tool providers through a JSON-RPC protocol.

```
┌─────────────────┐         ┌─────────────────┐
│                 │         │                 │
│  Amazon Q CLI   │◄───────►│   MCP Server    │
│  (MCP Client)   │  JSON   │  (Tool Provider)│
│                 │   RPC   │                 │
└─────────────────┘         └─────────────────┘
```

The MCP client is implemented in the `mcp_client` crate, with key files including:
- `client.rs`: Implements the client-side communication protocol
- `facilitator_types.rs`: Defines data structures for MCP communication
- `transport/`: Contains implementations for different transport mechanisms

## Key Files for Our Extensions

As we implement Project Wingman, we'll focus on these key files:

1. **q_chat/src/lib.rs**: The main implementation of the chat functionality
2. **q_chat/src/tools/mod.rs**: The tool system architecture we'll extend
3. **q_chat/src/tools/tool_index.json**: Tool specifications we'll enhance
4. **q_chat/src/conversation_state.rs**: Where we'll integrate our context graph
5. **mcp_client/src/client.rs**: The MCP client we'll extend for our tool registry

## Context Window Considerations

As we add more tools through Project Wingman, we need to be mindful of context window limits. The current implementation sends all available tool specifications to the model with each conversation state, which could become problematic as our tool ecosystem expands.

In future phases, we might need to implement:

1. **Tool Filtering**: Selectively including only relevant tools based on conversation context
2. **Tool Summarization**: Creating condensed versions of tool specifications
3. **Progressive Tool Loading**: Implementing a system where the model can request additional tool specifications as needed
4. **Tool Namespaces**: Organizing tools into logical groups
5. **Context Compression**: Developing techniques to compress tool specifications

Our conversation history system will help address this by maintaining records of which tools are most useful in which contexts, allowing us to predict which tools are most likely to be needed.

## Next Steps

With this architectural understanding, we're ready to implement our Project Wingman extensions. We'll start with the tool registry system, leveraging the existing MCP implementation, then move on to the context graph and domain-scoped browsing capabilities.
