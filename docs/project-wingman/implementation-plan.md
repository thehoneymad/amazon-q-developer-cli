# Project Wingman: Implementation Plan

This document outlines the detailed implementation plan for Project Wingman, based on the high-level plan and the existing Amazon Q CLI architecture.

## Current Architecture Assessment

After analyzing the Amazon Q CLI codebase, particularly the chat functionality, we've identified the following key points:

1. The tool system is well-structured but tightly integrated with the chat flow
2. Tools are currently defined within the codebase, not as external plugins
3. Context management exists but is limited to file-based context
4. There is no domain-scoped browsing or policy enforcement
5. No MCP branch was found in the repository, suggesting we need to implement this functionality from scratch

## Implementation Strategy

Based on our analysis, we'll implement Project Wingman in phases, focusing on extending the existing architecture rather than replacing it.

### Phase 1: Tool Registry Integration

#### 1.1 Tool Manifest Format (Week 1)

**Implementation Details:**
- Create a new crate `q_tool_registry` in the `crates` directory
- Define a YAML schema for tool manifests
- Implement parsing and validation for tool manifests

**Code Structure:**
```rust
// q_tool_registry/src/lib.rs
pub struct ToolManifest {
    pub name: String,
    pub type_: String,
    pub entry_point: PathBuf,
    pub capabilities: Vec<String>,
    pub mcp: Option<McpConfig>,
}

pub struct McpConfig {
    pub allowed_domains: Vec<String>,
}

impl ToolManifest {
    pub fn from_yaml(path: &Path) -> Result<Self> {
        // Parse YAML file
    }
}
```

#### 1.2 Tool Discovery (Week 2)

**Implementation Details:**
- Create a tool registry that discovers tools in standard locations
- Implement caching for discovered tools
- Add a command to list available tools

**Integration Points:**
- Extend `q_cli` to include a new subcommand: `tools list`
- Create a new module in `q_cli/src/commands/tools.rs`

**Code Structure:**
```rust
// q_tool_registry/src/registry.rs
pub struct ToolRegistry {
    tools: HashMap<String, ToolManifest>,
}

impl ToolRegistry {
    pub fn discover() -> Result<Self> {
        // Scan standard locations for tool manifests
    }
    
    pub fn list(&self) -> Vec<&ToolManifest> {
        // Return list of available tools
    }
}
```

#### 1.3 Tool Runner (Week 3)

**Implementation Details:**
- Create a tool runner that can execute tools by name
- Implement dynamic loading of tool entry points
- Add a command to run tools directly

**Integration Points:**
- Extend `q_cli` to include a new subcommand: `run <tool-name>`
- Integrate with the existing tool execution framework in `q_chat`

**Code Structure:**
```rust
// q_tool_registry/src/runner.rs
pub struct ToolRunner {
    registry: ToolRegistry,
}

impl ToolRunner {
    pub fn run(&self, name: &str, args: &[String]) -> Result<()> {
        // Find tool in registry
        // Load and execute tool
    }
}
```

### Phase 2: Domain-Scoped Browsing (Week 4-5)

#### 2.1 Policy Configuration

**Implementation Details:**
- Create a new crate `q_policy` in the `crates` directory
- Define a YAML schema for browser policies
- Implement parsing and validation for policies

**Code Structure:**
```rust
// q_policy/src/lib.rs
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

#### 2.2 HTTP Request Wrapper

**Implementation Details:**
- Create a wrapper around HTTP requests that enforces policies
- Implement domain validation against whitelist
- Add clear error messaging for rejected requests

**Integration Points:**
- Create a new module in `q_chat/src/tools/http_client.rs`
- Update existing tools to use the policy-enforced client

**Code Structure:**
```rust
// q_policy/src/http.rs
pub struct PolicyEnforcedClient {
    policy: BrowserPolicy,
    inner_client: reqwest::Client,
}

impl PolicyEnforcedClient {
    pub async fn get(&self, url: &str) -> Result<reqwest::Response> {
        // Check if URL is allowed by policy
        // If allowed, make request
        // If not allowed, return error
    }
}
```

### Phase 3: Context Provider Framework (Week 6-8)

#### 3.1 Context Graph

**Implementation Details:**
- Create a new crate `q_context_graph` in the `crates` directory
- Implement a simple RDF-like triple store
- Create APIs for updating and querying the graph

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

#### 3.2 Session Context

**Implementation Details:**
- Extend the existing `ConversationState` to include a context graph
- Implement persistence for session context
- Create mechanisms for overlaying global and local context

**Integration Points:**
- Modify `q_chat/src/conversation_state.rs` to include context graph
- Add context serialization and deserialization

**Code Structure:**
```rust
// Extension to ConversationState
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

#### 3.3 Context Commands

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

### Phase 4: Conversation Engine Enhancement (Week 9-10)

#### 4.1 Context-Aware Chat

**Implementation Details:**
- Modify the chat flow to include context graph in prompts
- Update the conversation state to track context changes
- Implement context-based reasoning

**Integration Points:**
- Modify `q_chat/src/lib.rs` to include context in prompts
- Update `ChatContext` to manage context graph

**Code Structure:**
```rust
// Extension to ChatContext
impl ChatContext {
    fn prepare_prompt_with_context(&self, user_input: &str) -> String {
        // Format user input with relevant context
        // Extract context that might be relevant to the query
        // Combine into a prompt
    }
}
```

#### 4.2 Context Visualization

**Implementation Details:**
- Create a simple visualization for context graphs
- Implement diff functionality for comparing contexts
- Add commands for inspecting context

**Integration Points:**
- Add new subcommands to `q_cli` for context visualization
- Create a new module in `q_cli/src/commands/context_viz.rs`

**Code Structure:**
```rust
// q_cli/src/commands/context_viz.rs
pub fn show(args: &ArgMatches) -> Result<()> {
    // Load context graph
    // Format and display
}

pub fn diff(args: &ArgMatches) -> Result<()> {
    // Load local and global context
    // Compare and display differences
}
```

### Phase 5: Internal Tool Integration (Week 11-14)

#### 5.1 Jira Tool

**Implementation Details:**
- Create a new tool for Jira integration
- Implement authentication and API interactions
- Add context graph updates for Jira entities

**Integration Points:**
- Create a new tool in `q_chat/src/tools/jira.rs`
- Register the tool in `q_chat/src/tools/mod.rs`
- Add tool specification to `q_chat/src/tools/tool_index.json`

**Code Structure:**
```rust
// q_chat/src/tools/jira.rs
#[derive(Debug, Clone, Deserialize)]
pub struct JiraTool {
    pub action: String,
    pub ticket_id: Option<String>,
    pub title: Option<String>,
    pub description: Option<String>,
    pub assignee: Option<String>,
}

impl JiraTool {
    pub async fn invoke(&self, context: &Context, updates: &mut impl Write) -> Result<InvokeOutput> {
        // Implement Jira API interactions
        // Update context graph with ticket information
    }
}
```

#### 5.2 Wiki Tool

**Implementation Details:**
- Create a new tool for wiki integration
- Implement reading and updating wiki pages
- Add context graph updates for wiki entities

**Integration Points:**
- Create a new tool in `q_chat/src/tools/wiki.rs`
- Register the tool in `q_chat/src/tools/mod.rs`
- Add tool specification to `q_chat/src/tools/tool_index.json`

**Code Structure:**
```rust
// q_chat/src/tools/wiki.rs
#[derive(Debug, Clone, Deserialize)]
pub struct WikiTool {
    pub action: String,
    pub page_id: Option<String>,
    pub title: Option<String>,
    pub content: Option<String>,
}

impl WikiTool {
    pub async fn invoke(&self, context: &Context, updates: &mut impl Write) -> Result<InvokeOutput> {
        // Implement wiki API interactions
        // Update context graph with wiki information
    }
}
```

#### 5.3 Git Tool

**Implementation Details:**
- Enhance the existing Git integration
- Add support for PRs, branches, and comments
- Implement context graph updates for Git entities

**Integration Points:**
- Create a new tool in `q_chat/src/tools/git.rs`
- Register the tool in `q_chat/src/tools/mod.rs`
- Add tool specification to `q_chat/src/tools/tool_index.json`

**Code Structure:**
```rust
// q_chat/src/tools/git.rs
#[derive(Debug, Clone, Deserialize)]
pub struct GitTool {
    pub action: String,
    pub repo: Option<String>,
    pub branch: Option<String>,
    pub pr_number: Option<String>,
    pub comment: Option<String>,
}

impl GitTool {
    pub async fn invoke(&self, context: &Context, updates: &mut impl Write) -> Result<InvokeOutput> {
        // Implement Git operations
        // Update context graph with Git information
    }
}
```

### Phase 6: Real-Time Graph Updates (Week 15-16)

#### 6.1 Automatic Context Updates

**Implementation Details:**
- Create a wrapper around tool execution that updates context
- Implement inference rules for derived relationships
- Add hooks for context updates

**Integration Points:**
- Modify `q_chat/src/tools/mod.rs` to include context updates
- Create a new module in `q_chat/src/context_updater.rs`

**Code Structure:**
```rust
// q_chat/src/context_updater.rs
pub struct ContextUpdater {
    graph: ContextGraph,
}

impl ContextUpdater {
    pub fn update_from_tool_result(&mut self, tool: &Tool, result: &InvokeOutput) -> Result<()> {
        // Extract entities and relationships from tool result
        // Update context graph
    }
    
    pub fn apply_inference_rules(&mut self) -> Result<()> {
        // Apply inference rules to derive new relationships
    }
}
```

#### 6.2 Context Daemon (Optional)

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

### Bonus Phase: UX and Sharing (Week 17-18)

#### 7.1 Tool Template Generator

**Implementation Details:**
- Create a scaffolding system for new tools
- Implement template generation
- Add documentation generation

**Integration Points:**
- Add a new subcommand to `q_cli`: `scaffold tool <name>`
- Create a new module in `q_cli/src/commands/scaffold.rs`

**Code Structure:**
```rust
// q_cli/src/commands/scaffold.rs
pub fn scaffold_tool(args: &ArgMatches) -> Result<()> {
    // Get tool name from args
    // Generate tool manifest
    // Generate skeleton code
    // Generate documentation
}
```

#### 7.2 Public Tool Registry (Future)

**Implementation Details:**
- Design a system for publishing tools
- Implement versioning and dependency management
- Create discovery mechanisms

**Integration Points:**
- Create a new crate `q_tool_registry_client` in the `crates` directory
- Add commands for publishing and discovering tools

**Code Structure:**
```rust
// q_tool_registry_client/src/lib.rs
pub struct RegistryClient {
    api_url: String,
}

impl RegistryClient {
    pub fn publish(&self, tool_path: &Path) -> Result<()> {
        // Package and publish tool
    }
    
    pub fn discover(&self, query: &str) -> Result<Vec<ToolInfo>> {
        // Search registry for tools
    }
}
```

## Integration with Existing Code

To integrate Project Wingman with the existing Amazon Q CLI codebase, we'll need to:

1. **Extend the Tool System**:
   - Modify `q_chat/src/tools/mod.rs` to support external tools
   - Update `Tool` enum to include dynamically loaded tools
   - Create a bridge between the tool registry and chat system

2. **Enhance Context Management**:
   - Extend `q_chat/src/context.rs` to include the context graph
   - Modify `ConversationState` to use the enhanced context
   - Update prompt generation to include relevant context

3. **Add New Commands**:
   - Extend `q_cli/src/main.rs` to include new subcommands
   - Create new command modules for tool and context management
   - Implement command handlers

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

1. **Alpha Release** (Week 8):
   - Basic tool registry
   - Domain-scoped browsing
   - Simple context management

2. **Beta Release** (Week 14):
   - Complete context graph
   - Initial internal tools
   - Context visualization

3. **Final Release** (Week 18):
   - All planned features
   - Documentation
   - Tool template generator

## Conclusion

This implementation plan provides a detailed roadmap for extending the Amazon Q CLI with the features outlined in Project Wingman. By following this plan, we can incrementally build on the existing architecture while adding powerful new capabilities for internal tool integration, context management, and domain-scoped browsing.
