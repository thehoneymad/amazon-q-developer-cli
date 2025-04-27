# Project Wingman

## Vision and Purpose

Project Wingman transforms the Amazon Q Developer CLI into a team-specific productivity platform. While Amazon Q provides AI-powered assistance, our vision extends beyond its capabilities to create an integrated experience tailored to our team's workflows and internal systems.

We're bridging the gap between Amazon Q's general-purpose capabilities and our team's needs. By extending the CLI with custom tools, context awareness, and domain-specific knowledge, we're creating an AI assistant that understands our development environment and interacts with our internal systems.

## Key Objectives

Our Project Wingman pursues four objectives that guide our development:

First, we're building a tool registry system that allows Amazon Q to interact with our internal tools and services. This means we'll be able to ask Amazon Q to create Jira tickets, search the wiki, or review code changes without leaving our terminal.

Second, we're implementing a context graph that gives Amazon Q persistent memory about our systems. Unlike the standard Amazon Q that starts each conversation fresh, our extended version will remember previous interactions and maintain understanding of our projects, repositories, and team structure.

Third, we're adding domain-scoped browsing capabilities to ensure secure access to internal resources. This allows Amazon Q to retrieve information from approved sources while maintaining security boundaries.

Fourth, we're documenting our entire journey to serve as both a technical reference for implementation and a learning resource for team members new to Rust development or the Amazon Q architecture.

## Document Structure

We've organized the Project Wingman documentation into sections:

- **Getting Started** provides setup instructions and first steps
- **Architecture** explains the system design and component interactions
- **Amazon Q Chat Architecture** details the existing chat functionality
- **API Integration and Memory** covers how the system interacts with backend services
- **MCP Implementation Analysis** examines the Model-Client-Provider framework
- **High-Level Plan** outlines our strategic roadmap
- **Implementation Plan** provides technical implementation steps
- **Extending Tools** guides us through adding new tools to the system
- **Rust Learning** offers resources for developing in Rust
- **Custom Tools** documents the team-specific tools we're building

## System Overview

```
┌─────────────────────┐      ┌───────────────────┐      ┌─────────────────┐
│                     │      │                   │      │                 │
│  Amazon Q CLI       │◄────►│  Tool Registry    │◄────►│  Internal Tools │
│  (Extended)         │      │                   │      │                 │
│                     │      └───────────────────┘      └─────────────────┘
│                     │                                         ▲
│                     │      ┌───────────────────┐              │
│                     │      │                   │              │
│                     │◄────►│  Context Graph    │◄─────────────┘
│                     │      │                   │
│                     │      └───────────────────┘
│                     │                                         ▲
│                     │      ┌───────────────────┐              │
│                     │      │                   │              │
│                     │◄────►│  Domain Policies  │◄─────────────┘
│                     │      │                   │
└─────────────────────┘      └───────────────────┘
```

> **Note:** We'll evolve this documentation alongside our project. As we implement features and gain insights, we'll update these pages to reflect our current understanding and approach.
