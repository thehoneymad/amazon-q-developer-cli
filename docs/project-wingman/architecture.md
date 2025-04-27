# Amazon Q Developer CLI Architecture

This document provides a comprehensive overview of the Amazon Q Developer CLI repository structure, focusing on the `q chat` functionality, its architecture, and how to extend it with new tools.

## High-Level Architecture

The repository is organized into several key components:

1. **Rust Crates (`/crates`)**: Core functionality implemented in Rust
   - `q_cli`: Main CLI application entry point
   - `q_chat`: Implementation of the chat functionality
   - Various supporting crates for authentication, API clients, etc.

2. **TypeScript Packages (`/packages`)**: UI components
   - `autocomplete`: React app for autocomplete functionality
   - `dashboard-app`: React app for the dashboard interface

3. **Extensions (`/extensions`)**: IDE integrations
   - `vscode`: VSCode plugin
   - `jetbrains`: JetBrains IDE plugin

## Q Chat Implementation

### Core Components

The `q chat` functionality is primarily implemented in the `q_chat` crate. The main components are:

1. **ChatContext**: The central structure that manages the chat session
2. **ConversationState**: Manages the conversation history and context
3. **Tool System**: Extensible framework for executing tools requested by the model
4. **StreamingClient**: Handles communication with the Amazon Bedrock API

### API and Model Integration

Amazon Q CLI uses Amazon Bedrock as its underlying AI service. The communication happens through:

1. **StreamingClient**: Implemented in `fig_api_client/src/clients/streaming_client.rs`
2. **Model Interfaces**: 
   - Uses either `amzn-codewhisperer-streaming-client` or `amzn-qdeveloper-streaming-client` depending on the environment
   - Automatically selects the appropriate client based on environment variables and context

### Data Flow

1. User input is captured via the terminal
2. Input is processed and sent to the model via the StreamingClient
3. The model responds with text and/or tool use requests
4. Tool use requests are validated, approved (if needed), and executed
5. Results are sent back to the model for further processing
6. Final response is displayed to the user

## Key Files for Understanding Q Chat

1. **q_chat/src/lib.rs**: Main implementation of the chat functionality
2. **q_chat/src/tools/mod.rs**: Tool system architecture
3. **q_chat/src/tools/tool_index.json**: Tool specifications
4. **fig_api_client/src/clients/streaming_client.rs**: API client implementation
5. **fig_api_client/src/model.rs**: Data models for API communication

## Bedrock API Integration

Amazon Q CLI uses Amazon Bedrock's streaming API for communication with the AI model. The integration:

1. Uses a streaming connection for real-time responses
2. Supports tool use through a specialized protocol
3. Handles conversation context and history
4. Manages token limits and context window constraints

The API client automatically selects between:
- `amzn-codewhisperer-streaming-client` for standard environments
- `amzn-qdeveloper-streaming-client` for AWS CloudShell or when specified by environment variables
