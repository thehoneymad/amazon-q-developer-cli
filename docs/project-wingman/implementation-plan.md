# Project Wingman: Engineering Implementation Plan

This document outlines our detailed implementation plan for Project Wingman, structured as an engineering plan with tasks that can be assigned to two team members. We've revised this plan based on our discovery of the existing MCP implementation in the codebase.

## Current Architecture Assessment

After analyzing the Amazon Q CLI codebase, particularly the MCP implementation in the `mcp-enablement-squashed` branch, we've identified the following key points:

1. The MCP client provides us with a robust framework for external tool integration using JSON-RPC
2. Custom tools are already supported through the `Tool::Custom` variant and `custom_tool.rs`
3. The tool manager handles discovery, registration, and execution of tools
4. There is no existing context graph system or domain policy enforcement

## Team Structure and Responsibilities

We'll organize our implementation with two engineers:

- **Engineer A (Tool & Policy Specialist)**: Responsible for tool registry integration, domain-scoped browsing, and internal tool implementation
- **Engineer B (Context & Memory Specialist)**: Responsible for context graph implementation, conversation state integration, and real-time updates

This division allows each engineer to focus on related components while maintaining clear interfaces between subsystems.

## Implementation Timeline

```
Week 1    Week 2    Week 3    Week 4    Week 5    Week 6    Week 7    Week 8    Week 9    Week 10   Week 11
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│        │        │        │        │        │        │        │        │        │        │        │
│ 1.1    │ 1.2    │ 1.3    │ 2.1    │ 2.2    │ 5.1    │ 5.2    │ 5.3    │        │        │        │ Engineer A
│ YAML   │ Tool   │ Tool   │ Policy │ HTTP   │ Jira   │ Wiki   │ Git    │        │        │        │
│ Manifest│Discovery│Commands│Config  │Wrapper │ Tool   │ Tool   │ Tool   │        │        │        │
│        │        │        │        │        │        │        │        │        │        │        │
├────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┼────────┤
│        │        │        │        │        │        │        │        │        │        │        │
│ 3.1    │ 3.2    │ 3.3    │ 4.1    │ 6.1    │ 6.2    │        │        │        │        │        │ Engineer B
│ Context │Convers.│Context │Context │Automatic│Context │        │        │        │        │        │
│ Graph  │State   │Commands│Chat    │Updates │Daemon  │        │        │        │        │        │
│        │        │        │        │        │        │        │        │        │        │        │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
                                    │                                                              │
                                    │                                                              │
                                    ▼                                                              ▼
                               Alpha Release                                                  Final Release
                               - YAML Tool Registry                                           - All Features
                               - Domain-Scoped Browsing                                       - Documentation
                               - Basic Context Graph                                          - Testing
```

## Detailed Task Breakdown

### Phase 1: Tool Registry Integration (3 Weeks)

#### 1.1 YAML Manifest Extension (Week 1) - Engineer A

**Tasks:**
- Create YAML schema for tool manifests
- Extend `CustomToolConfig` to support YAML format
- Implement YAML parsing and validation
- Write unit tests for manifest parsing

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

**Deliverables:**
- YAML manifest parser implementation
- Unit tests for manifest parsing
- Documentation for manifest format

#### 1.2 Tool Discovery Enhancement (Week 2) - Engineer A

**Tasks:**
- Implement tool discovery for `.amazonq/tools/<tool>.yaml` files
- Add caching mechanism for discovered tools
- Integrate with MCP client initialization
- Write integration tests for tool discovery

**Integration Points:**
- Modify `q_chat/src/tool_manager.rs` to include YAML manifest discovery
- Update the MCP client initialization in `q_chat/src/lib.rs`

**Deliverables:**
- Tool discovery implementation
- Integration tests
- Documentation for tool discovery process

#### 1.3 Tool Command Extensions (Week 3) - Engineer A

**Tasks:**
- Add CLI commands for listing available tools
- Implement direct tool execution command
- Create tool information display
- Write end-to-end tests for tool commands

**Integration Points:**
- Extend `q_cli/src/main.rs` to include new subcommands
- Create new command modules for tool management

**Deliverables:**
- CLI command implementations
- End-to-end tests
- User documentation for tool commands

### Phase 2: Domain-Scoped Browsing (2 Weeks)

#### 2.1 Policy Configuration (Week 4) - Engineer A

**Tasks:**
- Create policy configuration schema
- Implement YAML parsing for browser policies
- Add policy validation and error reporting
- Write unit tests for policy configuration

**Integration Points:**
- Create a new module in `q_chat/src/policy.rs`
- Integrate with the existing MCP client configuration

**Deliverables:**
- Policy configuration implementation
- Unit tests for policy validation
- Documentation for policy format

#### 2.2 HTTP Request Wrapper (Week 5) - Engineer A

**Tasks:**
- Create HTTP client wrapper with policy enforcement
- Implement domain validation against whitelist
- Add clear error messaging for rejected requests
- Write integration tests for policy enforcement

**Integration Points:**
- Extend `q_chat/src/tools/custom_tool.rs` to include policy enforcement
- Create a new HTTP client wrapper that enforces policies

**Deliverables:**
- HTTP client wrapper implementation
- Integration tests for policy enforcement
- Documentation for domain-scoped browsing

### Phase 3: Context Provider Framework (3 Weeks)

#### 3.1 Context Graph Implementation (Week 1) - Engineer B

**Tasks:**
- Design context graph data structure
- Implement RDF-like triple store
- Create APIs for updating and querying the graph
- Write unit tests for context graph operations

**Integration Points:**
- Create a new crate `q_context_graph` in the `crates` directory
- Design interfaces for integration with the MCP system

**Deliverables:**
- Context graph implementation
- Unit tests for graph operations
- API documentation

#### 3.2 Conversation State Integration (Week 2) - Engineer B

**Tasks:**
- Extend `ConversationState` to include context graph
- Implement persistence for session context
- Create mechanisms for overlaying global and local context
- Write integration tests for conversation state with context

**Integration Points:**
- Modify `q_chat/src/conversation_state.rs` to include context graph
- Add context serialization and deserialization

**Deliverables:**
- Extended conversation state implementation
- Integration tests
- Documentation for context integration

#### 3.3 Context Commands (Week 3) - Engineer B

**Tasks:**
- Add CLI commands for managing context
- Implement context promotion from local to global
- Create visualization for context graphs
- Write end-to-end tests for context commands

**Integration Points:**
- Extend `q_cli` to include new subcommands under `context`
- Create a new module in `q_cli/src/commands/context.rs`

**Deliverables:**
- CLI command implementations
- End-to-end tests
- User documentation for context commands

### Phase 4: Conversation Engine Enhancement (1 Week)

#### 4.1 Context-Aware Chat (Week 4) - Engineer B

**Tasks:**
- Modify chat flow to include context graph in prompts
- Update conversation state to track context changes
- Implement context-based reasoning
- Write integration tests for context-aware chat

**Integration Points:**
- Extend `q_chat/src/lib.rs` to include context in prompts
- Update `ChatContext` to manage context graph

**Deliverables:**
- Context-aware chat implementation
- Integration tests
- Documentation for context-aware features

### Phase 5: Internal Tool Integration (3 Weeks)

#### 5.1 Jira Tool (Week 6) - Engineer A

**Tasks:**
- Create MCP tool for Jira integration
- Implement authentication and API interactions
- Add context graph updates for Jira entities
- Write unit and integration tests for Jira tool

**Integration Points:**
- Create a new MCP tool implementation
- Register the tool with the MCP system
- Coordinate with Engineer B for context graph updates

**Deliverables:**
- Jira tool implementation
- Tests for Jira integration
- Documentation for Jira tool usage

#### 5.2 Wiki Tool (Week 7) - Engineer A

**Tasks:**
- Create MCP tool for wiki integration
- Implement reading and updating wiki pages
- Add context graph updates for wiki entities
- Write unit and integration tests for wiki tool

**Integration Points:**
- Create a new MCP tool implementation
- Register the tool with the MCP system
- Coordinate with Engineer B for context graph updates

**Deliverables:**
- Wiki tool implementation
- Tests for wiki integration
- Documentation for wiki tool usage

#### 5.3 Git Tool (Week 8) - Engineer A

**Tasks:**
- Enhance existing Git integration as an MCP tool
- Add support for PRs, branches, and comments
- Implement context graph updates for Git entities
- Write unit and integration tests for Git tool

**Integration Points:**
- Create a new MCP tool implementation
- Register the tool with the MCP system
- Coordinate with Engineer B for context graph updates

**Deliverables:**
- Git tool implementation
- Tests for Git integration
- Documentation for Git tool usage

### Phase 6: Real-Time Graph Updates (2 Weeks)

#### 6.1 Automatic Context Updates (Week 5) - Engineer B

**Tasks:**
- Create wrapper around MCP tool execution for context updates
- Implement inference rules for derived relationships
- Add hooks for context updates
- Write unit tests for context update rules

**Integration Points:**
- Extend the MCP client to include context updates
- Create a new module for context update rules
- Coordinate with Engineer A for tool integration

**Deliverables:**
- Context update implementation
- Unit tests for update rules
- Documentation for context update system

#### 6.2 Context Daemon (Week 6) - Engineer B

**Tasks:**
- Create background process for context management
- Implement event listeners for context updates
- Add notification system for context changes
- Write integration tests for context daemon

**Integration Points:**
- Create a new binary crate `q_contextd` in the `crates` directory
- Implement IPC between the daemon and CLI

**Deliverables:**
- Context daemon implementation
- Integration tests
- Documentation for daemon configuration

## Conversation History System (Optional)

If time permits after completing the core features, we'll implement a local conversation history system:

**Tasks:**
- Design storage format for conversation history
- Implement persistence to local files or embedded database
- Create commands for browsing and searching history
- Add integration with context graph

**Integration Points:**
- Extend `ConversationState` to support history persistence
- Add new CLI commands for history management

## Testing Strategy

We'll implement a comprehensive testing strategy across both engineers' work:

1. **Unit Tests**:
   - Each component will have unit tests with >80% coverage
   - Mock objects will be used for external dependencies
   - Tests will be run as part of CI/CD pipeline

2. **Integration Tests**:
   - Test interactions between components
   - Verify tool discovery and execution
   - Test context management and persistence

3. **End-to-End Tests**:
   - Test complete workflows
   - Verify chat with context and tools
   - Test policy enforcement

## Release Plan

1. **Alpha Release** (Week 6):
   - Extended MCP client with YAML support
   - Domain-scoped browsing
   - Basic context graph
   - Internal testing only

2. **Beta Release** (Week 9):
   - Complete context graph
   - Initial internal tools
   - Context visualization
   - Limited user testing

3. **Final Release** (Week 11):
   - All planned features
   - Comprehensive documentation
   - Context daemon
   - Full deployment

## Coordination and Communication

To ensure smooth collaboration between engineers:

1. **Daily Standup**: Brief daily meeting to discuss progress and blockers
2. **Interface Documentation**: Clear documentation of interfaces between components
3. **Code Reviews**: All PRs require review from the other engineer
4. **Weekly Demo**: Demo of completed features each week
5. **Shared Test Environment**: Common environment for integration testing

## Risk Management

We've identified these potential risks and mitigation strategies:

1. **MCP API Changes**: Keep close communication with MCP team; design for API versioning
2. **Performance Issues**: Regular performance testing; optimize context graph operations
3. **Security Concerns**: Security review of domain policy enforcement; penetration testing
4. **Integration Challenges**: Clear interface definitions; early integration testing

By following this implementation plan with clear task assignments and coordination mechanisms, we can efficiently deliver Project Wingman with two engineers while maintaining high quality and managing risks effectively.
