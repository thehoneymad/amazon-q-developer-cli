# Extending Amazon Q with Custom Tools

This guide explains how to add new tools to the Amazon Q Developer CLI, with a focus on the tool system architecture and implementation process.

## Tool System Architecture

Tools in Amazon Q CLI follow a plugin-like architecture:

1. **Tool Enum**: Defined in `q_chat/src/tools/mod.rs`, representing different tool types
2. **Tool Implementations**: Each tool has its own implementation file (e.g., `execute_bash.rs`, `fs_read.rs`)
3. **Tool Specifications**: Defined in `q_chat/src/tools/tool_index.json`, describing tools to the model

### Available Tools

The current tools include:

1. **execute_bash**: Execute bash commands
2. **fs_read**: Read files and directories
3. **fs_write**: Create and modify files
4. **use_aws**: Make AWS CLI API calls
5. **report_issue**: Create GitHub issues for bug reports

### Tool Execution Flow

1. Model requests a tool use via the API
2. Tool is validated for correct parameters
3. User is prompted for approval (if required)
4. Tool is executed and results are captured
5. Results are sent back to the model

## Adding New Tools

To add a new tool to Amazon Q CLI:

1. **Create Tool Implementation**:
   - Add a new file in `q_chat/src/tools/` (e.g., `my_tool.rs`)
   - Implement the necessary functionality
   - Export the tool in `q_chat/src/tools/mod.rs`

2. **Add Tool Specification**:
   - Update `q_chat/src/tools/tool_index.json` with the tool's name, description, and input schema
   - The input schema follows JSONSchema format

3. **Update Tool Enum**:
   - Add a new variant to the `Tool` enum in `q_chat/src/tools/mod.rs`
   - Implement the necessary trait methods for the new tool

4. **Implement Tool Logic**:
   - Implement validation, description, and execution logic
   - Handle permissions and user approval flow

## Example: Creating a Simple Tool

Here's a simplified example of creating a new tool that provides information about the current system:

### 1. Create the Tool Implementation File

Create a new file `q_chat/src/tools/system_info.rs`:

```rust
use std::io::Write;

use eyre::Result;
use serde::Deserialize;

use crate::tools::{InvokeOutput, OutputKind};
use fig_os_shim::Context;

#[derive(Debug, Clone, Deserialize)]
pub struct SystemInfo {
    pub detail_level: Option<String>,
}

impl SystemInfo {
    pub async fn invoke(&self, _context: &Context, _updates: &mut impl Write) -> Result<InvokeOutput> {
        // Get basic system information
        let hostname = hostname::get()?
            .into_string()
            .map_err(|_| eyre::eyre!("Failed to convert hostname to string"))?;
        
        let os_info = os_info::get();
        
        let detail = self.detail_level.as_deref().unwrap_or("basic");
        
        let mut output = format!(
            "System Information ({})\n\nHostname: {}\nOS: {} {}\nArchitecture: {}",
            detail,
            hostname,
            os_info.os_type(),
            os_info.version(),
            std::env::consts::ARCH
        );
        
        // Add more details if requested
        if detail == "detailed" {
            let cpu_info = sys_info::cpu_num()?;
            let mem_info = sys_info::mem_info()?;
            
            output.push_str(&format!(
                "\n\nCPU Cores: {}\nMemory: {} MB total, {} MB free",
                cpu_info,
                mem_info.total / 1024,
                mem_info.free / 1024
            ));
        }
        
        Ok(InvokeOutput {
            output: OutputKind::Text(output),
        })
    }

    pub fn queue_description(&self, updates: &mut impl Write) -> Result<()> {
        let detail = self.detail_level.as_deref().unwrap_or("basic");
        write!(
            updates,
            "Getting {} system information about this machine",
            detail
        )?;
        Ok(())
    }

    pub async fn validate(&mut self, _ctx: &Context) -> Result<()> {
        // Validate detail_level if provided
        if let Some(detail) = &self.detail_level {
            if detail != "basic" && detail != "detailed" {
                return Err(eyre::eyre!(
                    "Invalid detail_level: {}. Must be 'basic' or 'detailed'",
                    detail
                ));
            }
        }
        Ok(())
    }
}
```

### 2. Update the Tool Enum in `q_chat/src/tools/mod.rs`

```rust
// Add to the imports
pub mod system_info;
use system_info::SystemInfo;

// Add to the Tool enum
#[derive(Debug, Clone)]
pub enum Tool {
    FsRead(FsRead),
    FsWrite(FsWrite),
    ExecuteBash(ExecuteBash),
    UseAws(UseAws),
    GhIssue(GhIssue),
    SystemInfo(SystemInfo), // Add this line
}

// Update the implementation for Tool
impl Tool {
    // Update display_name method
    pub fn display_name(&self) -> &'static str {
        match self {
            // ... existing cases
            Tool::SystemInfo(_) => "system_info",
        }
    }

    // Update requires_acceptance method
    pub fn requires_acceptance(&self, _ctx: &Context) -> bool {
        match self {
            // ... existing cases
            Tool::SystemInfo(_) => false, // This tool doesn't need approval
        }
    }

    // Update invoke method
    pub async fn invoke(&self, context: &Context, updates: &mut impl Write) -> Result<InvokeOutput> {
        match self {
            // ... existing cases
            Tool::SystemInfo(system_info) => system_info.invoke(context, updates).await,
        }
    }

    // Update queue_description method
    pub async fn queue_description(&self, ctx: &Context, updates: &mut impl Write) -> Result<()> {
        match self {
            // ... existing cases
            Tool::SystemInfo(system_info) => system_info.queue_description(updates),
        }
    }

    // Update validate method
    pub async fn validate(&mut self, ctx: &Context) -> Result<()> {
        match self {
            // ... existing cases
            Tool::SystemInfo(system_info) => system_info.validate(ctx).await,
        }
    }
}

// Update TryFrom implementation
impl TryFrom<AssistantToolUse> for Tool {
    type Error = ToolUseResult;

    fn try_from(value: AssistantToolUse) -> std::result::Result<Self, Self::Error> {
        let map_err = |parse_error| /* ... existing code ... */;

        Ok(match value.name.as_str() {
            // ... existing cases
            "system_info" => Self::SystemInfo(serde_json::from_value::<SystemInfo>(value.args).map_err(map_err)?),
            unknown => {
                // ... existing code for unknown tools
            },
        })
    }
}
```

### 3. Add Tool Specification to `q_chat/src/tools/tool_index.json`

```json
{
  "system_info": {
    "name": "system_info",
    "description": "Get information about the current system, including hostname, OS details, and hardware information.",
    "input_schema": {
      "type": "object",
      "properties": {
        "detail_level": {
          "type": "string",
          "enum": ["basic", "detailed"],
          "description": "Level of detail to include in the system information. 'basic' includes hostname and OS info, 'detailed' adds CPU and memory information."
        }
      }
    }
  }
}
```

### 4. Update Cargo.toml Dependencies

Add any necessary dependencies to `q_chat/Cargo.toml`:

```toml
[dependencies]
# Existing dependencies...
hostname = "0.3"
os_info = "3.7"
sys-info = "0.9"
```

## Testing Your New Tool

1. Build the CLI with your changes:
   ```bash
   cargo build --bin q_cli
   ```

2. Run the CLI with chat mode:
   ```bash
   cargo run --bin q_cli -- chat
   ```

3. In the chat, ask Amazon Q to use your new tool:
   ```
   Can you show me information about my system using the system_info tool?
   ```

The model should recognize your new tool and use it to display system information.
