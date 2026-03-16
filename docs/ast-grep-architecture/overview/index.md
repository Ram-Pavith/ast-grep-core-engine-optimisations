# Overview Section

This section provides a comprehensive introduction to ast-grep, covering the project's purpose, architecture, core concepts, and getting started guide.

## Documents in this Section

### [1. Overview](./1-overview.md)

A high-level introduction to ast-grep, covering:
- What is ast-grep and its purpose
- High-level architecture diagram
- Workspace organization and crate structure
- Distribution and packaging channels
- Key file locations

### [1.1. System Architecture](./1.1-system-architecture.md)

Detailed technical architecture documentation, including:
- Layered architecture overview
- Core library type hierarchy
- Module structure for each crate
- Interface layer implementations (CLI, LSP, NAPI, PyO3)
- Component interaction patterns
- Build artifacts and distribution

### [1.2. Key Concepts](./1.2-key-concepts.md)

Fundamental concepts and abstractions:
- AST node representation (`Root`, `Node`, `Position`)
- Patterns and meta-variables
- The `Matcher` trait and implementations
- Rules and configuration
- Variable scopes and checking
- Severity levels

### [1.3. Getting Started](./1.3-getting-started.md)

Installation and basic usage guide:
- Installation via npm, pip, cargo, Homebrew, Scoop
- Verifying installation
- Basic usage examples
- Command structure
- Next steps for different use cases

## Quick Links

| Topic | Document |
|-------|----------|
| What is ast-grep? | [Overview](./1-overview.md#what-is-ast-grep) |
| Architecture diagram | [Overview](./1-overview.md#high-level-architecture) |
| Core types | [Key Concepts](./1.2-key-concepts.md#ast-node-representation) |
| Meta-variables | [Key Concepts](./1.2-key-concepts.md#patterns-and-meta-variables) |
| Installation | [Getting Started](./1.3-getting-started.md#installation) |
| CLI commands | [Getting Started](./1.3-getting-started.md#command-structure) |

## Related Sections

- [Core Library](../core-library/2-core-library.md) - Pattern matching and node manipulation APIs
- [Rule System](../rule-system/3-rule-system.md) - Rule configuration format and execution
- [Language Support](../language-support/4-language-support.md) - Adding new languages
- [CLI Tool](../cli-tool/5-cli-tool.md) - Command-line interface documentation
