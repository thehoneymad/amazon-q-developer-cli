# MCP Implementation Analysis

In this document, we analyze the existing MCP (Model-Client-Provider) implementation we discovered in the Amazon Q CLI codebase, specifically in the `mcp-enablement-squashed` branch. This analysis informs our approach to extending the system for Project Wingman.

## Overview

The MCP implementation provides us with a framework for integrating custom tools through a client-server architecture. This allows us to extend the CLI with external tools that can be dynamically loaded and executed - a capability that aligns perfectly with our Project Wingman goals.

## Key Components

### 1. MCP Client

We found the MCP client implemented in the `mcp_client` crate, providing core functionality for communicating with MCP servers. Our key files include:

- `client.rs`: Implements our client-side communication protocol
- `facilitator_types.rs`: Defines our data structures for MCP communication
- `transport/`: Contains our implementations for different transport mechanisms

```
mcp_client/
├── src/
│   ├── client.rs           # Core client implementation
│   ├── facilitator_types.rs # Data structures for communication
│   ├── transport/          # Transport layer implementations
│   │   ├── stdio.rs        # Standard I/O transport
│   │   └── ...
│   └── lib.rs              # Library entry point
└── Cargo.toml
```

We use JSON-RPC over stdio for communication with MCP servers, allowing for bidirectional messaging between our CLI and external tools.

### 2. Custom Tool Integration

We implement custom tool integration in `q_chat/src/tools/custom_tool.rs`, which provides a bridge between our Amazon Q chat system and external MCP tools. Our key features include:

- Tool configuration via our `CustomToolConfig` struct:
  ```rust
  pub struct CustomToolConfig {
      pub command: String,
      pub args: Vec<String>,
      pub env: Option<HashMap<String, String>>,
      pub timeout: u64,
  }
  ```

- Integration with our existing tool system through the `Tool::Custom` variant in our `Tool` enum

### 3. Tool Manager

Our `q_chat/src/tool_manager.rs` file implements the management of tools, including custom MCP tools. It handles:

- Our tool discovery and registration process
- Our tool execution and result processing
- Our communication with MCP servers

## Architecture

Our MCP implementation follows a client-server architecture:

```
┌─────────────────┐         ┌─────────────────┐
│                 │         │                 │
│  Amazon Q CLI   │◄───────►│   MCP Server    │
│  (MCP Client)   │  JSON   │  (Tool Provider)│
│                 │   RPC   │                 │
└─────────────────┘         └─────────────────┘
        │                           │
        │                           │
        ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│                 │         │                 │
│  Tool System    │         │  Tool           │
│  Integration    │         │  Implementation │
│                 │         │                 │
└─────────────────┘         └─────────────────┘
```

1. **MCP Client**: Our Amazon Q CLI acts as a client that communicates with MCP servers
2. **MCP Server**: External processes that implement our MCP protocol and provide tool functionality
3. **JSON-RPC Protocol**: Our communication between client and server using JSON-RPC over stdio

## Integration with Chat System

Our MCP system integrates with the Amazon Q chat functionality through:

1. **Tool Registration**: We register custom tools alongside built-in tools
2. **Tool Execution**: When the model requests a custom tool, we forward the request to the appropriate MCP server
3. **Result Processing**: We format results from MCP servers and return them to the model

The sequence flow for tool execution looks like this:

```
┌──────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│          │     │            │     │            │     │            │
│  Model   │────►│ Tool System│────►│ MCP Client │────►│ MCP Server │
│          │     │            │     │            │     │            │
└──────────┘     └────────────┘     └────────────┘     └────────────┘
      ▲                                                      │
      │                                                      │
      │                                                      │
      │                                                      ▼
┌──────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│          │     │            │     │            │     │            │
│  Model   │◄────│ Tool System│◄────│ MCP Client │◄────│ MCP Server │
│          │     │            │     │            │     │            │
└──────────┘     └────────────┘     └────────────┘     └────────────┘
```

## Configuration

We configure MCP tools through a configuration system that specifies:

- The command to execute for our MCP server
- Arguments to pass to our server
- Environment variables for our server process
- Timeout settings for our tool execution

## Recent Changes

Based on our review of the commit history, recent changes to our MCP implementation include:

- Improved error handling and reporting
- Timeout configuration
- Telemetry for MCP server initialization and tool execution
- Status reporting for MCP server loading
- Validation for MCP server configurations

## Implications for Project Wingman

The existing MCP implementation provides us with a solid foundation for the tool registry system we proposed in Project Wingman. Our key considerations include:

1. **Leverage Existing Code**: We can use our MCP client and custom tool integration as the foundation for our tool registry system
2. **Extend Configuration**: We can extend our current configuration to support the YAML manifest format we proposed
3. **Add Domain Scoping**: We can implement our domain-scoped browsing capability as an extension to our existing MCP system
4. **Context Integration**: We need to integrate our new context graph system with our existing MCP implementation

## Next Steps

Based on our analysis, our next steps are:

1. **Detailed Code Review**: We'll conduct a thorough review of our MCP implementation to understand all capabilities and limitations
2. **Gap Analysis**: We'll identify gaps between our existing implementation and Project Wingman requirements
3. **Extension Plan**: We'll develop a plan for extending our MCP implementation to support Project Wingman features
4. **Integration Strategy**: We'll define how our context graph and domain-scoped browsing will integrate with our existing MCP system

By building on our existing MCP implementation rather than starting from scratch, we can accelerate our development timeline and ensure compatibility with the current Amazon Q CLI architecture.
