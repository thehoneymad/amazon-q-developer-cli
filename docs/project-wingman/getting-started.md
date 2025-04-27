# Getting Started with Project Wingman

In this guide, we'll walk through setting up our development environment and creating our first extension for the Amazon Q Developer CLI. By the end, we'll understand the project structure and have built a simple custom tool.

## Prerequisites

Before diving into Project Wingman, we need to ensure our system has:

- Rust toolchain (via [rustup](https://rustup.rs/))
- Git
- A code editor with Rust support (VS Code with rust-analyzer works well)
- Basic familiarity with command-line interfaces

## Setting Up Our Development Environment

Our development environment setup follows a straightforward process that prepares our system for extending Amazon Q.

We'll start by cloning the repository to our local machine:

```bash
git clone https://github.com/aws/amazon-q-developer-cli.git
cd amazon-q-developer-cli
```

Next, we'll install the necessary dependencies. The project includes an automated setup script that handles most of the configuration:

```bash
npm run setup
```

This script installs the Rust toolchain, project dependencies, and configures pre-commit hooks to maintain code quality.

Once the setup completes, we'll build the project to verify everything works correctly:

```bash
cargo build
```

This command compiles all the crates in the workspace. The first build might take several minutes as it downloads and compiles dependencies.

To test our setup, we'll run the Amazon Q CLI in chat mode:

```bash
cargo run --bin q_cli -- chat
```

We should see the Amazon Q chat interface start up, indicating our development environment is ready.

## Project Structure

Understanding the project structure helps us navigate the codebase effectively. For Project Wingman, we focus primarily on extending the `q_chat` crate, which implements the chat functionality.

```
amazon-q-developer-cli/
├── crates/
│   ├── q_cli/               # Main CLI application
│   ├── q_chat/              # Chat functionality
│   │   ├── src/
│   │   │   ├── lib.rs       # Core implementation
│   │   │   ├── tools/       # Tool implementations
│   │   │   │   ├── mod.rs   # Tool system architecture
│   │   │   │   └── ...      # Individual tool files
│   │   │   └── ...
│   │   └── Cargo.toml
│   └── ...                  # Other supporting crates
└── ...
```

The tool system lives in `q_chat/src/tools/`, where each tool has its own implementation file. The `mod.rs` file defines the tool system architecture and handles registration of all tools.

## Creating Our First Extension

Let's create a simple "hello world" tool to understand the extension process. This example demonstrates the core components of a tool: parameters, validation, execution, and output.

### Step 1: Create the Tool Implementation

We'll create a new file at `crates/q_chat/src/tools/hello_world.rs`:

```rust
use std::io::Write;

use eyre::Result;
use serde::Deserialize;

use crate::tools::{InvokeOutput, OutputKind};
use fig_os_shim::Context;

#[derive(Debug, Clone, Deserialize)]
pub struct HelloWorld {
    pub name: Option<String>,
}

impl HelloWorld {
    pub async fn invoke(&self, _context: &Context, _updates: &mut impl Write) -> Result<InvokeOutput> {
        let name = self.name.as_deref().unwrap_or("World");
        let output = format!("Hello, {}! This is our first custom tool.", name);
        
        Ok(InvokeOutput {
            output: OutputKind::Text(output),
        })
    }

    pub fn queue_description(&self, updates: &mut impl Write) -> Result<()> {
        let name = self.name.as_deref().unwrap_or("World");
        write!(updates, "Saying hello to {}", name)?;
        Ok(())
    }

    pub async fn validate(&mut self, _ctx: &Context) -> Result<()> {
        // No validation needed for this simple tool
        Ok(())
    }
}
```

This code defines a simple tool that accepts an optional name parameter and returns a greeting.

### Step 2: Register the Tool

Now we'll modify `crates/q_chat/src/tools/mod.rs` to include our new tool:

```rust
// Add to the imports
pub mod hello_world;
use hello_world::HelloWorld;

// Add to the Tool enum
#[derive(Debug, Clone)]
pub enum Tool {
    // ... existing tools
    HelloWorld(HelloWorld),
}

// Update the implementation for Tool methods
impl Tool {
    pub fn display_name(&self) -> &'static str {
        match self {
            // ... existing cases
            Tool::HelloWorld(_) => "hello_world",
        }
    }

    // Update other methods similarly...
}

// Update TryFrom implementation
impl TryFrom<AssistantToolUse> for Tool {
    type Error = ToolUseResult;

    fn try_from(value: AssistantToolUse) -> std::result::Result<Self, Self::Error> {
        // ... existing code
        
        Ok(match value.name.as_str() {
            // ... existing cases
            "hello_world" => Self::HelloWorld(serde_json::from_value::<HelloWorld>(value.args).map_err(map_err)?),
            unknown => {
                // ... existing code for unknown tools
            },
        })
    }
}
```

### Step 3: Define the Tool Specification

We'll update `crates/q_chat/src/tools/tool_index.json` to include our new tool:

```json
{
  "hello_world": {
    "name": "hello_world",
    "description": "A simple tool that says hello to the provided name.",
    "input_schema": {
      "type": "object",
      "properties": {
        "name": {
          "type": "string",
          "description": "The name to greet. Defaults to 'World' if not provided."
        }
      }
    }
  },
  // ... existing tools
}
```

This JSON schema tells the model about our tool's name, purpose, and parameters.

### Step 4: Test Our Tool

We'll build and run the CLI to test our new tool:

```bash
cargo build --bin q_cli
cargo run --bin q_cli -- chat
```

In the chat interface, we'll ask Amazon Q to use our new tool:

```
Can you use the hello_world tool to greet me?
```

The model should recognize our tool and use it to generate a greeting.

## Tool Development Workflow

The diagram below illustrates our typical workflow for developing and testing a new tool:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  Implement Tool │────►│ Register Tool   │────►│ Define Schema   │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
┌─────────────────┐     ┌─────────────────┐     ┌────────▼────────┐
│                 │     │                 │     │                 │
│   Iterate      ◄│─────│  Test in Chat   │◄────│  Build Project  │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## Next Steps

After creating our first tool, we'll consider these paths to deepen our understanding:

1. Study the implementation of more complex tools like `fs_write` or `execute_bash` to learn advanced patterns
2. Enhance our tool with additional parameters and error handling
3. Create a tool that addresses a specific need in our team's workflow
4. Explore the MCP system for creating external tool providers

The [Extending Tools](./extending-tools.md) guide provides more detailed information on creating complex tools, while the [Rust Learning](./rust-learning.md) guide helps us master the language features used in the project.

## Troubleshooting Common Issues

When developing tools, we might encounter these common issues:

**Build Errors**: Rust's compiler provides detailed error messages. We should pay attention to the suggested fixes and ensure our types match the expected signatures.

**Runtime Errors**: We can add logging statements with `tracing::debug!()` to trace execution flow and identify where issues occur.

**Tool Not Found**: We need to verify our tool is properly registered in both `mod.rs` and `tool_index.json`. The names must match exactly.

**Tool Not Working**: We should check that our tool's implementation matches the schema in `tool_index.json`. Parameter names and types must be consistent between the schema and our Rust struct.

If we encounter persistent issues, the [Amazon Q Chat Architecture](./amazon-q-chat-architecture.md) document provides deeper insights into how the tool system works.
