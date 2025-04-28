# Project Wingman: High-Level Plan

In this document, we outline our high-level plan for extending Amazon Q CLI with additional capabilities focused on internal tool integration, context management, and domain-scoped browsing.

## Our Vision

With Project Wingman, we aim to build a scoped, tool-based AI CLI platform by extending Amazon Q CLI to include:

- A tool registry (MCP-style)
- Scoped browsing via URL/domain whitelist
- Context providers (knowledge graph, persistent memory)
- Scoped/local + global context merging
- Integration with internal tools (Jira, wiki, etc.)
- Agentic behavior driven by these layers

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                     Amazon Q Developer CLI                  │
│                                                             │
├─────────────────┬─────────────────┬─────────────────────────┤
│                 │                 │                         │
│  Tool Registry  │  Context Graph  │  Domain-Scoped Browser  │
│                 │                 │                         │
├─────────────────┴─────────────────┴─────────────────────────┤
│                                                             │
│                     Internal Tool Integration               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Our Implementation Phases

We'll implement Project Wingman in six phases, each building on the previous one to create a comprehensive extension to Amazon Q CLI.

### Phase 1: Tool Registry Integration

In this phase, we'll establish the foundation for tool registration and discovery.

#### 1. Tool Specification Schema

We'll define a standard format for our tool manifests:

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

This schema allows us to specify tool metadata, capabilities, and security constraints in a clear, declarative format.

#### 2. Tool Loader

We'll build a system to discover, parse, and list available tools:

- We'll parse tool manifests from standard locations
- We'll validate tool specifications against our schema
- We'll implement an `amazonq tools list` command for easy discovery

#### 3. Tool Runner

We'll create a mechanism to execute tools by name:

- We'll implement `amazonq run <tool-name> --arg path=/foo/bar`
- We'll handle tool execution and result processing
- We'll integrate with the existing Amazon Q CLI tool execution framework

### Phase 2: Domain-Scoped Browsing

In this phase, we'll implement safe and auditable web access for our tools.

#### 4. Global Policy File

We'll create a configuration for allowed domains:

```yaml
# ~/.amazonq/browser_policy.yaml
allow:
  - github.com
  - jira.myorg.com
deny_all_others: true
```

This policy file will define which domains our tools can access, providing security and compliance controls.

#### 5. Central Policy Enforcer

We'll develop a security layer for HTTP requests:

- We'll create a Rust module that wraps tool HTTP calls
- We'll implement domain validation against our whitelist
- We'll add clear error messaging for rejected requests

### Phase 3: Context Provider Framework

In this phase, we'll build a system for managing and persisting context.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  Local Context  │────►│  Context Graph  │◄────│ Global Context  │
│                 │     │                 │     │                 │
└─────────────────┘     └────────┬────────┘     └─────────────────┘
                                 │
                                 │
                        ┌────────▼────────┐
                        │                 │
                        │ Conversation    │
                        │ Engine          │
                        │                 │
                        └─────────────────┘
```

#### 6. Basic Context Graph Provider

We'll implement a local knowledge graph:

- We'll define a local RDF triple store (in-memory or JSON-LD)
- We'll create an API for updating and querying the graph:
  ```
  context update --subject pr:42 --predicate :linkedTo --object wiki:123
  context query --subject me --predicate :workingOn
  ```

#### 7. Conversation-Scoped Context Store

We'll develop per-conversation memory:

- We'll store session data in `~/.amazonq/session/<id>.json`
- We'll overlay global context during conversations
- We'll implement context inheritance and overrides

#### 8. Global Context Management

We'll create mechanisms for promoting local context:

- We'll implement an `amazonq context save` command
- We'll handle merging of local facts into global context
- We'll resolve conflicts during context promotion

### Phase 4: Conversation Engine Enhancement

In this phase, we'll extend the Amazon Q chat functionality with context awareness.

#### 9. Memory-Aware Conversation Engine

We'll enhance the existing chat system:

- We'll load global and local context when starting a conversation
- We'll process messages with context-aware reasoning
- We'll update our local context graph based on conversation and tool results

#### 10. Context Visualization

We'll add tools for inspecting context:

- We'll implement `amazonq context show --local`
- We'll create `amazonq context diff` to compare local and global context
- We'll visualize context relationships

### Phase 5: Internal Tool Integration

In this phase, we'll develop specific tools for internal workflows.

#### 11. Jira Tool

We'll create a Jira integration tool:

- We'll implement commands: get-ticket, create-ticket, assign
- We'll add facts to our context graph (e.g., `ticket:JIRA-123 :assignedTo user:alice`)
- We'll handle authentication and API interactions

#### 12. Wiki Tool

We'll develop a wiki integration:

- We'll support reading and updating wiki pages
- We'll add relationship data to our graph (e.g., `wiki:abc :mentions ticket:JIRA-123`)
- We'll implement search and discovery features

#### 13. Git Tool

We'll build enhanced Git integration:

- We'll support PRs, branches, and comments
- We'll capture code-review relationships in our context graph
- We'll integrate with existing Git workflows

### Phase 6: Real-Time Graph Updates

In this phase, we'll implement automatic context management.

#### 14. Automatic Context Updates

We'll enhance tool execution with context awareness:

- We'll wrap each tool call with context updates
- We'll allow our graph to evolve naturally as actions are taken
- We'll implement inference rules for derived relationships

#### 15. Context Daemon (Optional)

We'll create a background process for context management:

- We'll implement a `contextd` daemon that listens for events
- We'll update the context graph in real-time
- We'll provide notifications for significant context changes

### Bonus Phase: UX and Sharing

In this optional phase, we'll improve developer experience and collaboration.

#### 16. Tool Template Generator

We'll create scaffolding for new tools:

- We'll implement `amazonq scaffold tool jira-internal`
- We'll generate tool manifest and skeleton code
- We'll include documentation and examples

#### 17. Public Tool Registry

We'll plan for community sharing:

- We'll design a system for publishing tools to a registry or GitHub repo
- We'll implement versioning and dependency management
- We'll create discovery mechanisms for available tools

## Timeline and Milestones

We'll organize our implementation into these key milestones:

1. **Foundation (Phases 1-2)**: Tool registry and domain-scoped browsing - 5 weeks
2. **Context System (Phase 3)**: Context graph and memory management - 3 weeks
3. **Integration (Phases 4-5)**: Conversation engine and internal tools - 6 weeks
4. **Advanced Features (Phase 6+)**: Real-time updates and UX improvements - 4 weeks

This phased approach allows us to deliver value incrementally while building toward our complete vision for Project Wingman.
