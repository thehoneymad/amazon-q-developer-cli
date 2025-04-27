# Rust Learning for Amazon Q CLI Development

This guide provides Rust learning resources and concepts specifically relevant to extending the Amazon Q Developer CLI.

## Key Rust Concepts for Amazon Q CLI

### 1. Enums and Pattern Matching

The tool system in Amazon Q CLI makes heavy use of Rust's enums and pattern matching. Understanding these concepts is crucial:

```rust
// Example from q_chat/src/tools/mod.rs
pub enum Tool {
    FsRead(FsRead),
    FsWrite(FsWrite),
    ExecuteBash(ExecuteBash),
    // ...
}

impl Tool {
    pub fn display_name(&self) -> &'static str {
        match self {
            Tool::FsRead(_) => "fs_read",
            Tool::FsWrite(_) => "fs_write",
            // ...
        }
    }
}
```

**Resources:**
- [Rust Book: Enums and Pattern Matching](https://doc.rust-lang.org/book/ch06-00-enums.html)
- [Rust By Example: Match](https://doc.rust-lang.org/rust-by-example/flow_control/match.html)

### 2. Traits and Trait Objects

The Amazon Q CLI uses traits to define common behavior across different types:

```rust
// Conceptual example (not actual code)
trait ToolBehavior {
    fn validate(&self) -> Result<()>;
    fn execute(&self) -> Result<Output>;
    fn describe(&self) -> String;
}
```

**Resources:**
- [Rust Book: Traits](https://doc.rust-lang.org/book/ch10-02-traits.html)
- [Rust By Example: Traits](https://doc.rust-lang.org/rust-by-example/trait.html)

### 3. Error Handling with Result and eyre

The codebase uses the `eyre` crate for error handling, which is an extension of Rust's standard `Result` type:

```rust
use eyre::Result;

pub async fn validate(&mut self, ctx: &Context) -> Result<()> {
    if !self.path.exists() {
        return Err(eyre::eyre!("Path does not exist: {}", self.path.display()));
    }
    Ok(())
}
```

**Resources:**
- [Rust Book: Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
- [eyre crate documentation](https://docs.rs/eyre/latest/eyre/)

### 4. Asynchronous Programming with Tokio

Amazon Q CLI uses Tokio for asynchronous operations, which is essential for handling network requests and tool execution:

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // Async code here
}

pub async fn invoke(&self, context: &Context, updates: &mut impl Write) -> Result<InvokeOutput> {
    // Async implementation
}
```

**Resources:**
- [Rust Book: Asynchronous Programming](https://doc.rust-lang.org/book/ch16-00-concurrency.html)
- [Tokio Documentation](https://tokio.rs/tokio/tutorial)

### 5. Serde for Serialization/Deserialization

The CLI uses Serde extensively for handling JSON data, especially in tool parameters:

```rust
#[derive(Debug, Clone, Deserialize)]
pub struct FsRead {
    pub path: PathBuf,
    pub mode: ReadMode,
    pub start_line: Option<i64>,
    pub end_line: Option<i64>,
    // ...
}
```

**Resources:**
- [Serde Documentation](https://serde.rs/)
- [Rust By Example: Serde](https://doc.rust-lang.org/rust-by-example/trait/derive.html)

## Development Workflow Tips

### 1. Using `cargo check` for Quick Validation

Before running a full build, use `cargo check` to quickly validate your code:

```bash
cargo check -p q_chat
```

### 2. Running Tests for a Specific Crate

Test your changes to a specific crate:

```bash
cargo test -p q_chat
```

### 3. Debugging with `dbg!` and Logging

Use the `dbg!` macro for quick debugging:

```rust
let result = dbg!(compute_something());
```

For more structured logging, use the `tracing` crate:

```rust
use tracing::{debug, error, info};

debug!("Processing request: {:?}", request);
```

### 4. Using Rust Analyzer in Your IDE

Rust Analyzer provides excellent IDE support for Rust development. Make sure it's installed and configured in your editor.

## Common Patterns in Amazon Q CLI

### 1. Builder Pattern

Many structs in the codebase use the builder pattern:

```rust
let client = StreamingClient::builder()
    .with_endpoint(endpoint)
    .with_credentials(credentials)
    .build()?;
```

### 2. Command Pattern

The CLI uses a command pattern for handling user input:

```rust
match Command::parse(&user_input)? {
    Command::Ask { prompt } => handle_ask(prompt).await,
    Command::Clear => handle_clear().await,
    Command::Help => handle_help().await,
    // ...
}
```

### 3. State Machine Pattern

The chat functionality uses a state machine to manage conversation flow:

```rust
enum ChatState {
    PromptUser { ... },
    HandleInput { ... },
    ExecuteTools(Vec<QueuedTool>),
    // ...
}
```

## Recommended Learning Path

1. **Start with Rust Fundamentals**:
   - Complete the [Rust Book](https://doc.rust-lang.org/book/)
   - Practice with [Rustlings](https://github.com/rust-lang/rustlings)

2. **Understand Key Crates**:
   - Study the `tokio` crate for async programming
   - Learn `serde` for serialization/deserialization
   - Explore `eyre` for error handling

3. **Explore the Amazon Q CLI Codebase**:
   - Start with `q_chat/src/lib.rs` to understand the overall structure
   - Examine existing tools in `q_chat/src/tools/` to see implementation patterns
   - Look at `q_chat/src/tools/mod.rs` to understand the tool system

4. **Make Small Changes First**:
   - Modify an existing tool to add a new parameter
   - Add logging to understand the execution flow
   - Create a simple new tool based on existing patterns

5. **Gradually Increase Complexity**:
   - Implement more complex tools
   - Modify core functionality
   - Contribute to the main codebase
