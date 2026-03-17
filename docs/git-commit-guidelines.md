# Git Commit Guidelines

## Purpose

This document defines the commit message conventions used in this
repository.

The goals are to:

-   Keep commit history **clear and readable**
-   Make it easy to understand **what changed and why**
-   Facilitate **code reviews**
-   Support future **automation (changelogs, releases, etc.)**

# General Principles

Good commits should be:

-   **Atomic** -- each commit represents a single logical change
-   **Descriptive** -- the message explains what changed
-   **Concise** -- short but meaningful
-   **Consistent** -- follow the format defined below

A reader should be able to understand the purpose of the commit
**without reading the code**.

# Commit Message Structure

All commits should follow this structure:

    <type>: <short description>

    [optional longer description]

    [optional footer]

Example:

    feat: add axe-core accessibility audit integration

    Implements the accessibility audit runner using axe-core.
    Results are returned as structured JSON for further processing.

# Commit Types

| Type | Description |
|---|---|
| feat | A new feature |
| fix |      A bug fix |
| refactor |  Code change that does not fix a bug or add a feature |
|  docs |       Documentation changes |
|  style |      Formatting changes (no logic changes) |
|  test |       Adding or modifying tests |
|  chore |      Maintenance tasks (dependencies, config, etc.) |
|  perf |       Performance improvements |
|  build |      Changes affecting build tools or dependencies|
|  ci |         CI/CD configuration changes |

# Examples

### Feature

    feat: add accessibility audit endpoint

### Bug Fix

    fix: handle invalid payload in audit endpoint

### Refactor

    refactor: extract audit logic into separate module

### Documentation

    docs: add architecture documentation

### Tests

    test: add unit tests for accessibility auditor

### Chore

    chore: update eslint configuration

# Writing Good Commit Messages

### Use imperative mood

Good:

    feat: add audit runner

Bad:

    feat: added audit runner
    feat: adding audit runner

Think of the message as completing this sentence:

> If applied, this commit will **add audit runner**

### Keep the first line short

Recommended:

    ≤ 72 characters

### Explain **why**, not only **what**

Bad:

    fix: change validation

Better:

    fix: prevent crash when audit payload is missing required fields

# Commit Size

Prefer **small commits** over large ones.

Good:

    feat: add audit endpoint
    test: add tests for audit endpoint
    docs: document audit API

Avoid:

    feat: implement full accessibility audit system

# Branching Recommendations

Recommended branch naming:

    feature/<feature-name>
    fix/<bug-description>
    refactor/<module>
    docs/<topic>

Examples:

    feature/accessibility-audit
    fix/json-parsing-error
    docs/project-architecture

# Example Commit History

Good commit history:

    feat: add accessibility audit endpoint
    refactor: move audit logic to separate module
    test: add tests for accessibility auditor
    docs: add architecture documentation
    fix: handle invalid audit payload

# Summary

Follow these key rules:

1.  Use the correct **commit type**
2.  Write messages in **imperative mood**
3.  Keep commits **small and focused**
4.  Explain **why the change exists**
5.  Keep the commit history **clean and readable**
