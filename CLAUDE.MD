# Claude Code Instructions for FinDogAI

This file provides essential coding standards for AI assistants working on the FinDogAI project.
Before starting any task, read and follow ALL guidelines mentioned in this file.

## Coding Standards

### 1. Universal Coding Philosophy

**Location**:[.claude_rules/coding-general.md](.claude_rules/coding-general.md)** - Core coding practices, change management, and code preservation

These are project-agnostic rules that govern **how** code changes should be made:

- **Simplicity Over Cleverness**: Prefer simple, maintainable solutions
- **Minimal Changes**: Make the smallest reasonable changes to achieve the goal
- **No Reimplementation**: Never rewrite from scratch without permission
- **Task Boundaries**: Only make changes directly related to the current task
- **Incremental Fixes**: Prefer incremental improvements over wholesale replacements
- **Match Surrounding Style**: Consistency within a file is paramount
- **Evergreen Names**: Never use names like "improved", "new", "enhanced"
- **Error Handling**: Either log and handle exceptions OR throw them, never both
- **Version Control**: Wait for human request before committing changes

### 2. Project-Specific Technical Standards

**Location**: `docs/architecture/coding-standards.md`

### 3. Various specific rules

- **[.claude_rules/code-comments.md](.claude_rules/code-comments.md)** - Comment standards and file documentation requirements
- **[.claude_rules/NgRx-Usage.md](.claude_rules/NgRx-Usage.md)** - Mandatory NgRx patterns for Angular state management
- **[.claude_rules/testing.md](.claude_rules/testing.md)** - TDD practices, test coverage, and quality standards

These are FinDogAI-specific implementation standards.

## Key Principles

- **Minimal Changes**: Make the smallest reasonable changes to achieve goals
- **No Reimplementation**: Never rewrite code without explicit permission
- **Test-Driven Development**: Follow Red-Green-Refactor cycle
- **NgRx Mandatory**: All Angular state management through NgRx Store and Effects
- **File Headers**: Every code file must have "ABOUTME: " documentation header
- **Don't Commit**: Never commit code changes without human request
- **NEVER Throw Away Existing Code**: NEVER throw away existing code when fixing bugs or errors - always get explicit permission for rewrites.

## Exception Keywords

Only skip mandatory requirements if explicitly authorized:

- Testing: "I AUTHORIZE YOU TO SKIP WRITING TESTS THIS TIME"
- NgRx: "I AUTHORIZE YOU TO SKIP NGRX THIS TIME"
