# Project Wingman: Revised Implementation Plan

This document outlines the revised implementation plan for Project Wingman, based on the high-level plan and the discovery of the existing MCP implementation in the codebase.

## Current Architecture Assessment

After analyzing the Amazon Q CLI codebase, particularly the MCP implementation in the `mcp-enablement-squashed` branch, we've identified the following key points:

1. The MCP client provides a robust framework for external tool integration using JSON-RPC
2. Custom tools are already supported through the `Tool::Custom` variant and `custom_tool.rs`
3. The tool manager handles discovery, registration, and execution of tools
4. There is no existing context graph system or domain policy enforcement

## Revised Implementation Strategy

Based on our analysis, we'll implement Project Wingman by extending the existing MCP architecture rather than building new systems from scratch.

### Phase 1: Tool Registry Integration (3 Weeks)

#### 1.1 YAML Manifest Extension (Week 1)

**Implementation Details:**
- Extend the existing `CustomToolConfig` to support YAML manifests
- Create a parser for the proposed YAML format
- Map YAML fields to the existing configuration structure

**Code Structure:**
```rust
// Extension to q_chat/src/tools/custom_tool.rs
#[derive(Clone, Serialize, Deserialize, Debug)]
pub struct ToolManifest {
    pub name: String,
    pub type_: String,
    pub entry_point: PathBuf,
    pub capabilities: Vec<String>,
    pub mcp: Option<McpConfig>,
}

impl ToolManifest {
    pub fn from_yaml(path: &Path) -> Result<Self> {
        // Parse YAML file
    }
    
    pub fn to_custom_tool_config(&self) -> CustomToolConfig {
        // Convert to existing CustomToolConfig
    }
}
```

#### 1.2 Tool Discovery Enhancement (Week 2)

**Implementation Details:**
- Extend the tool discovery mechanism to scan for `.amazonq/tools/<tool>.yaml` files
- Integrate with the existing MCP client initialization
- Add caching for discovered tools

**Integration Points:**
- Modify `q_chat/src/tool_manager.rs` to include YAML manifest discovery
- Update the MCP client initialization in `q_chat/src/lib.rs`

**Code Structure:**
```rust
// Extension to q_chat/src/tool_manager.rs
impl ToolManager {
    pub fn discover_yaml_tools(&mut self) -> Result<()> {
        // Scan standard locations for YAML tool manifests
        // Convert to CustomToolConfig and register
    }
}
```

#### 1.3 Tool Command Extensions (Week 3)

**Implementation Details:**
- Add commands for listing and running tools directly
- Implement tool information display
- Create a mechanism for direct tool execution

**Integration Points:**
- Extend `q_cli/src/main.rs` to include new subcommands
- Create new command modules for tool management

**Code Structure:**
```rust
// New module in q_cli/src/commands/tools.rs
pub fn list(args: &ArgMatches) -> Result<()> {
    // List available tools using the existing tool manager
}

pub fn run(args: &ArgMatches) -> Result<()> {
    // Run a specific tool using the existing MCP client
}
```

### Phase 2: Domain-Scoped Browsing (2 Weeks)

#### 2.1 Policy Configuration (Week 1)

**Implementation Details:**
- Create a policy configuration system for domain whitelisting
- Implement YAML parsing for browser policies
- Add policy validation and error reporting

**Integration Points:**
- Create a new module in `q_chat/src/policy.rs`
- Integrate with the existing MCP client configuration

**Code Structure:**
```rust
// New module in q_chat/src/policy.rs
pub struct BrowserPolicy {
    pub allow: Vec<String>,
    pub deny_all_others: bool,
}

impl BrowserPolicy {
    pub fn from_yaml(path: &Path) -> Result<Self> {
        // Parse YAML file
    }
    
    pub fn is_allowed(&self, url: &str) -> bool {
        // Check if URL is allowed by policy
    }
}
```

#### 2.2 HTTP Request Wrapper (Week 2)

**Implementation Details:**
- Create a wrapper around HTTP requests in MCP tools
- Implement domain validation against whitelist
- Add clear error messaging for rejected requests

**Integration Points:**
- Extend `q_chat/src/tools/custom_tool.rs` to include policy enforcement
- Create a new HTTP client wrapper that enforces policies

**Code Structure:**
```rust
// Extension to q_chat/src/tools/custom_tool.rs
impl CustomTool {
    fn enforce_domain_policy(&self, url: &str) -> Result<()> {
        // Get policy from configuration
        // Check if URL is allowed
        // Return error if not allowed
    }
}
```

### Phase 3: Context Provider Framework (3 Weeks)

#### 3.1 Context Graph Implementation (Week 1)

**Implementation Details:**
- Create a new context graph system for knowledge representation
- Implement a simple RDF-like triple store
- Create APIs for updating and querying the graph

**Integration Points:**
- Create a new crate `q_context_graph` in the `crates` directory
- Design interfaces for integration with the MCP system

**Code Structure:**
```rust
// q_context_graph/src/lib.rs
pub struct ContextGraph {
    triples: Vec<Triple>,
}

pub struct Triple {
    subject: String,
    predicate: String,
    object: String,
}

impl ContextGraph {
    pub fn add(&mut self, subject: &str, predicate: &str, object: &str) {
        // Add triple to graph
    }
    
    pub fn query(&self, subject: Option<&str>, predicate: Option<&str>, object: Option<&str>) -> Vec<&Triple> {
        // Query graph for matching triples
    }
}
```

#### 3.2 Conversation State Integration (Week 2)

**Implementation Details:**
- Extend the existing `ConversationState` to include a context graph
- Implement persistence for session context
- Create mechanisms for overlaying global and local context

**Integration Points:**
- Modify `q_chat/src/conversation_state.rs` to include context graph
- Add context serialization and deserialization

**Code Structure:**
```rust
// Extension to ConversationState in q_chat/src/conversation_state.rs
pub struct ConversationState {
    // Existing fields...
    context_graph: ContextGraph,
}

impl ConversationState {
    pub fn add_context(&mut self, subject: &str, predicate: &str, object: &str) {
        self.context_graph.add(subject, predicate, object);
    }
    
    pub fn save_context(&self) -> Result<()> {
        // Save context to file
    }
}
```

#### 3.3 Context Commands (Week 3)

**Implementation Details:**
- Add commands for managing context
- Implement context promotion from local to global
- Create visualization for context graphs

**Integration Points:**
- Extend `q_cli` to include new subcommands under `context`
- Create a new module in `q_cli/src/commands/context.rs`

**Code Structure:**
```rust
// q_cli/src/commands/context.rs
pub fn update(args: &ArgMatches) -> Result<()> {
    // Parse subject, predicate, object from args
    // Update context graph
}

pub fn query(args: &ArgMatches) -> Result<()> {
    // Parse query parameters from args
    // Query context graph and display results
}

pub fn save(args: &ArgMatches) -> Result<()> {
    // Promote local context to global
}
```

### Phase 4: Conversation Engine Enhancement (1 Week)

#### 4.1 Context-Aware Chat

**Implementation Details:**
- Modify the chat flow to include context graph in prompts
- Update the conversation state to track context changes
- Implement context-based reasoning

**Integration Points:**
- Extend `q_chat/src/lib.rs` to include context in prompts
- Update `ChatContext` to manage context graph

**Code Structure:**
```rust
// Extension to ChatContext in q_chat/src/lib.rs
impl ChatContext {
    fn prepare_prompt_with_context(&self, user_input: &str) -> String {
        // Format user input with relevant context
        // Extract context that might be relevant to the query
        // Combine into a prompt
    }
}
```

### Phase 5: Internal Tool Integration (3 Weeks)

#### 5.1 Jira Tool (Week 1)

**Implementation Details:**
- Create a new MCP tool for Jira integration
- Implement authentication and API interactions
- Add context graph updates for Jira entities

**Integration Points:**
- Create a new MCP tool implementation
- Register the tool with the MCP system

**Code Structure:**
```rust
// Implementation as an MCP tool
pub struct JiraTool {
    client: JiraClient,
}

impl JiraTool {
    pub async fn handle_request(&self, request: JsonRpcRequest) -> Result<JsonRpcResponse> {
        // Parse request parameters
        // Perform Jira API operation
        // Update context graph
        // Return response
    }
}
```

#### 5.2 Wiki Tool (Week 1)

**Implementation Details:**
- Create a new MCP tool for wiki integration
- Implement reading and updating wiki pages
- Add context graph updates for wiki entities

**Integration Points:**
- Create a new MCP tool implementation
- Register the tool with the MCP system

**Code Structure:**
```rust
// Implementation as an MCP tool
pub struct WikiTool {
    client: WikiClient,
}

impl WikiTool {
    pub async fn handle_request(&self, request: JsonRpcRequest) -> Result<JsonRpcResponse> {
        // Parse request parameters
        // Perform wiki API operation
        // Update context graph
        // Return response
    }
}
```

#### 5.3 Git Tool (Week 1)

**Implementation Details:**
- Enhance the existing Git integration as an MCP tool
- Add support for PRs, branches, and comments
- Implement context graph updates for Git entities

**Integration Points:**
- Create a new MCP tool implementation
- Register the tool with the MCP system

**Code Structure:**
```rust
// Implementation as an MCP tool
pub struct GitTool {
    client: GitClient,
}

impl GitTool {
    pub async fn handle_request(&self, request: JsonRpcRequest) -> Result<JsonRpcResponse> {
        // Parse request parameters
        // Perform Git operation
        // Update context graph
        // Return response
    }
}
```

### Phase 6: Real-Time Graph Updates (2 Weeks)

#### 6.1 Automatic Context Updates (Week 1)

**Implementation Details:**
- Create a wrapper around MCP tool execution that updates context
- Implement inference rules for derived relationships
- Add hooks for context updates

**Integration Points:**
- Extend the MCP client to include context updates
- Create a new module for context update rules

**Code Structure:**
```rust
// New module for context updates
pub struct ContextUpdater {
    graph: ContextGraph,
}

impl ContextUpdater {
    pub fn update_from_tool_result(&mut self, tool: &str, result: &JsonRpcResponse) -> Result<()> {
        // Extract entities and relationships from tool result
        // Update context graph
    }
    
    pub fn apply_inference_rules(&mut self) -> Result<()> {
        // Apply inference rules to derive new relationships
    }
}
```

#### 6.2 Context Daemon (Week 2)

**Implementation Details:**
- Create a background process for context management
- Implement event listeners for context updates
- Add notification system for context changes

**Integration Points:**
- Create a new binary crate `q_contextd` in the `crates` directory
- Implement IPC between the daemon and CLI

**Code Structure:**
```rust
// q_contextd/src/main.rs
fn main() -> Result<()> {
    // Initialize context graph
    // Start event listeners
    // Run daemon loop
}

struct ContextDaemon {
    graph: ContextGraph,
    listeners: Vec<Box<dyn EventListener>>,
}

impl ContextDaemon {
    fn run(&mut self) -> Result<()> {
        // Process events and update context
    }
}
```

## Integration with Existing MCP Code

To integrate Project Wingman with the existing MCP implementation, we'll:

1. **Extend the MCP Client**:
   - Add support for YAML manifests
   - Implement domain policy enforcement
   - Integrate with the context graph system

2. **Enhance the Tool Manager**:
   - Add discovery for YAML tool manifests
   - Implement tool listing and direct execution
   - Add context-aware tool execution

3. **Modify the Conversation State**:
   - Include the context graph
   - Add context-aware prompt generation
   - Implement context persistence

## Testing Strategy

We'll implement a comprehensive testing strategy:

1. **Unit Tests**:
   - Test each component in isolation
   - Use mock objects for external dependencies
   - Ensure high test coverage for core functionality

2. **Integration Tests**:
   - Test interactions between components
   - Verify tool discovery and execution
   - Test context management and persistence

3. **End-to-End Tests**:
   - Test complete workflows
   - Verify chat with context and tools
   - Test policy enforcement

## Deployment and Release Plan

1. **Alpha Release** (Week 6):
   - Extended MCP client with YAML support
   - Domain-scoped browsing
   - Basic context graph

2. **Beta Release** (Week 9):
   - Complete context graph
   - Initial internal tools
   - Context visualization

3. **Final Release** (Week 11):
   - All planned features
   - Documentation
   - Context daemon

## Conclusion

This revised implementation plan leverages the existing MCP implementation while adding the key features of Project Wingman. By building on the current architecture rather than creating new systems, we can deliver the project more efficiently while ensuring compatibility with the existing codebase.
