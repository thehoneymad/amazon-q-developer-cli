# Getting Started with Project Wingman

This guide will help you set up your development environment and make your first extension to the Amazon Q Developer CLI.

## Prerequisites

Before you begin, ensure you have the following installed:

- Rust toolchain (via [rustup](https://rustup.rs/))
- Git
- A code editor with Rust support (VS Code with rust-analyzer is recommended)
- Basic familiarity with command-line interfaces

## Setting Up Your Development Environment

1. **Clone the Repository**

   ```bash
   git clone https://github.com/aws/amazon-q-developer-cli.git
   cd amazon-q-developer-cli
   ```

2. **Install Dependencies**

   Follow the setup instructions in the main README.md file, or use the automated setup:

   ```bash
   npm run setup
   ```

   This will install the necessary Rust toolchain, dependencies, and set up pre-commit hooks.

3. **Build the Project**

   ```bash
   cargo build
   ```

   This will compile all the crates in the workspace.

4. **Run the CLI**

   ```bash
   cargo run --bin q_cli -- chat
   ```

   This will start the Amazon Q chat interface.

## Project Structure for Extensions

For Project Wingman, we'll focus on extending the `q_chat` crate with new tools. Here's the relevant directory structure:

```
crates/
└── q_chat/
    ├── src/
    │   ├── lib.rs           # Main implementation of chat functionality
    │   ├── tools/           # Directory containing all tool implementations
    │   │   ├── mod.rs       # Tool system architecture and registration
    │   │   ├── tool_index.json # Tool specifications for the model
    │   │   ├── execute_bash.rs # Example tool implementation
    │   │   ├── fs_read.rs   # Example tool implementation
    │   │   └── ...
    │   └── ...
    └── Cargo.toml           # Dependencies for q_chat
```

## Making Your First Extension

Let's create a simple "hello world" tool to understand the extension process:

1. **Create a New Tool File**

   Create a new file at `crates/q_chat/src/tools/hello_world.rs`:

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
           let output = format!("Hello, {}! This is your first custom tool.", name);
           
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

2. **Update the Tool Module**

   Modify `crates/q_chat/src/tools/mod.rs` to include your new tool:

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

       pub fn requires_acceptance(&self, _ctx: &Context) -> bool {
           match self {
               // ... existing cases
               Tool::HelloWorld(_) => false, // This tool doesn't need approval
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

3. **Add Tool Specification**

   Update `crates/q_chat/src/tools/tool_index.json` to include your new tool:

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

4. **Build and Test**

   ```bash
   cargo build --bin q_cli
   cargo run --bin q_cli -- chat
   ```

   In the chat, ask Amazon Q to use your new tool:

   ```
   Can you use the hello_world tool to greet me?
   ```

## Next Steps

After creating your first simple tool, you can:

1. **Explore Existing Tools**: Study the implementation of more complex tools like `fs_write` or `execute_bash`
2. **Add More Features**: Enhance your tool with more parameters and functionality
3. **Create Team-Specific Tools**: Develop tools that address your team's specific needs
4. **Learn More Rust**: Use the [Rust Learning](./rust-learning.md) guide to deepen your understanding

## Development Workflow

For an efficient development workflow:

1. **Make Small, Incremental Changes**: Focus on one feature at a time
2. **Test Frequently**: Run the CLI after each significant change
3. **Use Version Control**: Commit your changes regularly
4. **Follow Rust Conventions**: Use `cargo fmt` and `cargo clippy` to maintain code quality
5. **Add Tests**: Write tests for your tools to ensure they work as expected

## Troubleshooting

If you encounter issues:

- **Build Errors**: Check the error message carefully, Rust's compiler provides helpful guidance
- **Runtime Errors**: Add logging with `tracing::debug!()` to understand the execution flow
- **Tool Not Found**: Ensure your tool is properly registered in both `mod.rs` and `tool_index.json`
- **Tool Not Working**: Verify that your tool's implementation matches the schema in `tool_index.json`
