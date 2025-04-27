# Custom Tools for Project Wingman

This section documents the custom tools developed as part of Project Wingman to extend the Amazon Q Developer CLI for team-specific purposes.

## Overview

Custom tools allow Amazon Q to perform specific actions tailored to your team's workflows. Each tool follows the same architecture as the built-in tools but implements functionality specific to your needs.

## Tool Development Process

1. **Identify a Need**: Determine what functionality would benefit your team
2. **Design the Tool Interface**: Define the parameters and behavior
3. **Implement the Tool**: Create the Rust implementation
4. **Register the Tool**: Add it to the tool system
5. **Test and Refine**: Ensure the tool works as expected

## Tool Categories

Our custom tools are organized into the following categories:

### Team Workflow Tools

Tools that integrate with your team's specific workflows and processes.

### Development Utilities

Tools that assist with common development tasks specific to your environment.

### AWS Service Integration

Tools that provide specialized interaction with AWS services beyond what's available in the standard CLI.

### Documentation Helpers

Tools that help generate, update, or access documentation relevant to your team.

## Adding a New Tool

See the [Extending Tools](../extending-tools.md) guide for detailed instructions on adding new tools to the Amazon Q CLI.
