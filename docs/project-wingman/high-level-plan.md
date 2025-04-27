# Project Wingman: High-Level Plan

This document outlines the high-level plan for extending Amazon Q CLI with additional capabilities focused on internal tool integration, context management, and domain-scoped browsing.

## Vision

Project Wingman aims to build a scoped, tool-based AI CLI platform by extending Amazon Q CLI to include:

- A tool registry (MCP-style)
- Scoped browsing via URL/domain whitelist
- Context providers (knowledge graph, persistent memory)
- Scoped/local + global context merging
- Integration with internal tools (Jira, wiki, etc.)
- Agentic behavior driven by these layers

## Implementation Phases

### Phase 1: Tool Registry Integration

Establish the foundation for tool registration and discovery.

#### 1. Tool Specification Schema

Define a standard format for tool manifests:

```yaml
# .amazonq/tools/<tool>.yaml
name: wiki-browser
type: tool
entry_point: ./tools/wiki.rs
capabilities:
  - browse
  - read
mcp:
  allowed_domains:
    - wiki.myorg.com
```

#### 2. Tool Loader

Build a system to discover, parse, and list available tools:

- Parse tool manifests from standard locations
- Validate tool specifications
- Implement `amazonq tools list` command

#### 3. Tool Runner

Create a mechanism to execute tools by name:

- Implement `amazonq run <tool-name> --arg path=/foo/bar`
- Handle tool execution and result processing
- Integrate with existing Amazon Q CLI tool execution framework

### Phase 2: Domain-Scoped Browsing

Implement safe and auditable web access for tools.

#### 4. Global Policy File

Create a configuration for allowed domains:

```yaml
# ~/.amazonq/browser_policy.yaml
allow:
  - github.com
  - jira.myorg.com
deny_all_others: true
```

#### 5. Central Policy Enforcer

Develop a security layer for HTTP requests:

- Create a Rust module that wraps tool HTTP calls
- Implement domain validation against whitelist
- Add clear error messaging for rejected requests

### Phase 3: Context Provider Framework

Build a system for managing and persisting context.

#### 6. Basic Context Graph Provider

Implement a local knowledge graph:

- Define a local RDF triple store (in-memory or JSON-LD)
- Create an API for updating and querying the graph:
  ```
  context update --subject pr:42 --predicate :linkedTo --object wiki:123
  context query --subject me --predicate :workingOn
  ```

#### 7. Conversation-Scoped Context Store

Develop per-conversation memory:

- Store session data in `~/.amazonq/session/<id>.json`
- Overlay global context during conversations
- Implement context inheritance and overrides

#### 8. Global Context Management

Create mechanisms for promoting local context:

- Implement `amazonq context save` command
- Handle merging of local facts into global context
- Resolve conflicts during context promotion

### Phase 4: Conversation Engine Enhancement

Extend the Amazon Q chat functionality with context awareness.

#### 9. Memory-Aware Conversation Engine

Enhance the existing chat system:

- Load global and local context when starting a conversation
- Process messages with context-aware reasoning
- Update local context graph based on conversation and tool results

#### 10. Context Visualization

Add tools for inspecting context:

- Implement `amazonq context show --local`
- Create `amazonq context diff` to compare local and global context
- Visualize context relationships

### Phase 5: Internal Tool Integration

Develop specific tools for internal workflows.

#### 11. Jira Tool

Create a Jira integration tool:

- Implement commands: get-ticket, create-ticket, assign
- Add facts to context graph (e.g., `ticket:JIRA-123 :assignedTo user:alice`)
- Handle authentication and API interactions

#### 12. Wiki Tool

Develop a wiki integration:

- Support reading and updating wiki pages
- Add relationship data to graph (e.g., `wiki:abc :mentions ticket:JIRA-123`)
- Implement search and discovery features

#### 13. Git Tool

Build enhanced Git integration:

- Support PRs, branches, and comments
- Capture code-review relationships in context graph
- Integrate with existing Git workflows

### Phase 6: Real-Time Graph Updates

Implement automatic context management.

#### 14. Automatic Context Updates

Enhance tool execution with context awareness:

- Wrap each tool call with context updates
- Allow the graph to evolve naturally as actions are taken
- Implement inference rules for derived relationships

#### 15. Context Daemon (Optional)

Create a background process for context management:

- Implement a `contextd` daemon that listens for events
- Update the context graph in real-time
- Provide notifications for significant context changes

### Bonus Phase: UX and Sharing

Improve developer experience and collaboration.

#### 16. Tool Template Generator

Create scaffolding for new tools:

- Implement `amazonq scaffold tool jira-internal`
- Generate tool manifest and skeleton code
- Include documentation and examples

#### 17. Public Tool Registry

Plan for community sharing:

- Design a system for publishing tools to a registry or GitHub repo
- Implement versioning and dependency management
- Create discovery mechanisms for available tools
