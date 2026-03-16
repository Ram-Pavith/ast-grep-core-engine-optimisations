# Language Server Protocol

> **Source:** https://deepwiki.com/ast-grep/ast-grep/6-language-server-protocol

Relevant source files

-   [CHANGELOG.md](https://github.com/ast-grep/ast-grep/blob/3ae01aec/CHANGELOG.md)
-   [Cargo.lock](https://github.com/ast-grep/ast-grep/blob/3ae01aec/Cargo.lock)
-   [Cargo.toml](https://github.com/ast-grep/ast-grep/blob/3ae01aec/Cargo.toml)
-   [crates/cli/Cargo.toml](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/Cargo.toml)
-   [crates/cli/src/config.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs)
-   [crates/cli/src/lsp.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs)
-   [crates/cli/src/new.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/new.rs)
-   [crates/cli/src/verify.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/verify.rs)
-   [crates/config/Cargo.toml](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/config/Cargo.toml)
-   [crates/core/Cargo.toml](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/core/Cargo.toml)
-   [crates/lsp/Cargo.toml](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml)
-   [crates/napi/Cargo.toml](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/Cargo.toml)

## Purpose and Scope

This document describes the Language Server Protocol (LSP) implementation in ast-grep, which enables IDE integration for real-time diagnostics and code actions based on configured rules. The LSP server exposes ast-grep's pattern matching capabilities through standard LSP features such as diagnostics, quick fixes, and hover information.

For information about the CLI tool that also supports an `lsp` command, see [CLI Tool](/ast-grep/ast-grep/5-cli-tool). For rule configuration that the LSP server consumes, see [Rule System](/ast-grep/ast-grep/3-rule-system).

---

## System Architecture

The LSP integration sits at the interface layer of ast-grep's architecture, consuming the core matching engine and rule configuration system while exposing IDE-compatible services.

**LSP Server Architectural Layers**

The LSP server operates through three distinct layers:

1.  **Transport Layer**: `Server` from `tower-lsp-server` handles JSON-RPC communication over stdin/stdout
2.  **Service Layer**: `LspService` manages async request dispatch and response handling
3.  **Logic Layer**: `Backend` implements LSP request handlers using ast-grep's core functionality

Sources: [crates/lsp/Cargo.toml](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml) [crates/cli/src/lsp.rs1-37](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L1-L37)

---

## Server Initialization

The LSP server is started through the CLI's `lsp` subcommand, which bootstraps the tokio runtime and wires together the components.

### Startup Sequence

The server initialization follows this call chain:

1.  `main_with_args()` in [crates/cli/src/lib.rs126-146](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lib.rs#L126-L146) parses the `lsp` subcommand
2.  `run_language_server()` in [crates/cli/src/lsp.rs31-37](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L31-L37) builds the tokio runtime
3.  `run_language_server_impl()` in [crates/cli/src/lsp.rs10-29](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L10-L29) constructs the service
4.  The `rule_finder` closure is created in [crates/cli/src/lsp.rs20-23](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L20-L23)
5.  `LspService::build()` and `Backend::new()` in [crates/cli/src/lsp.rs26](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L26-L26) create the server
6.  `Server::new().serve()` in [crates/cli/src/lsp.rs27](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L27-L27) starts the event loop

Sources: [crates/cli/src/lsp.rs10-37](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L10-L37) [crates/cli/src/lib.rs141](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lib.rs#L141-L141)

### Rule Finder Closure Pattern

The LSP implementation uses a **closure-based rule discovery pattern** that decouples rule loading from the LSP crate itself. This design allows the CLI to provide the rule finding logic while the LSP crate remains agnostic to filesystem details.

The closure:

```rust
let rule_finder = move || {
    project.find_rules(&RuleOverwrite::default())
};
```

-   Captures `project_config: ProjectConfig` by move
-   Returns `Result<RuleCollection<SgLang>>`
-   Can be invoked multiple times to reload rules
-   Uses `RuleOverwrite::default()` for standard rule filtering

This pattern enables **dynamic rule reloading** without restarting the LSP server, as the `Backend` can re-invoke the closure when configuration files change.

Sources: [crates/cli/src/lsp.rs20-23](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L20-L23) [crates/cli/src/config.rs85-91](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L85-L91)

---

## Core Components

### Backend Structure

The `Backend` type from `ast-grep-lsp` implements the core LSP request handlers. It is constructed with three key dependencies:

| Parameter | Type | Purpose |
| --- | --- | --- |
| `client` | LSP client handle | Sends notifications/requests back to the editor |
| `config_base` | `PathBuf` | Base directory for resolving relative paths in rules |
| `rule_finder` | `FnMut() -> Result<RuleCollection<SgLang>>` | Closure to (re)load rules on demand |

The `Backend` stores:

-   A reference to the LSP client for bidirectional communication
-   The project base directory from `ProjectConfig.project_dir`
-   The rule finder closure for lazy/dynamic rule loading
-   Likely a `DashMap` for document state (inferred from dependency)

Sources: [crates/cli/src/lsp.rs26](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L26-L26) [crates/lsp/Cargo.toml20-26](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L20-L26)

### Tower-LSP Server Integration

The `ast-grep-lsp` crate uses `tower-lsp-server` version 0.23.0 as the foundation for the LSP implementation.

**Key Types from tower-lsp-server:**

-   `Server`: Main server struct that manages the JSON-RPC transport over stdio
-   `LspService`: Service wrapper that handles async LSP request dispatch
-   `LanguageServer` trait: Interface that `Backend` implements

**Service Construction:**

```rust
let (service, socket) = LspService::build(|client| {
    Backend::new(client, config_base, rule_finder)
}).finish();

Server::new(stdin, stdout, socket).serve(service).await;
```

The `LspService::build()` method:

1.  Takes a factory function that receives a client handle
2.  Returns a `(service, socket)` tuple
3.  The service handles LSP requests
4.  The socket manages the connection lifecycle

Sources: [crates/cli/src/lsp.rs25-27](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L25-L27) [crates/lsp/Cargo.toml26](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L26-L26)

### Asynchronous Request Handling

The LSP server operates entirely asynchronously using tokio as the async runtime.

**Runtime Configuration:**

```rust
tokio::runtime::Builder::new_multi_thread()
    .enable_all()
    .build()?
    .block_on(run_language_server_impl(project))
```

The runtime is configured with:

-   `new_multi_thread()`: Multi-threaded work-stealing scheduler
-   `enable_all()`: Enables both IO and timer drivers
-   `block_on()`: Bridges sync CLI entry point to async LSP implementation

Sources: [crates/cli/src/lsp.rs32-36](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L32-L36) [crates/lsp/Cargo.toml32-40](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L32-L40)

---

## LSP Capabilities

### Diagnostics

The LSP server publishes diagnostics for files that match configured rules. The diagnostic pipeline:

1.  Editor opens/modifies a file
2.  LSP server receives `textDocument/didOpen` or `textDocument/didChange` notification
3.  Server invokes `rule_finder()` to get current `RuleCollection`
4.  Server parses document into AST using language detection
5.  Each rule in collection is matched against the AST
6.  Matches are converted to LSP `Diagnostic` objects
7.  Server publishes diagnostics via `client.publish_diagnostics()`

**Diagnostic Fields Mapping:**

| Rule Field | LSP Diagnostic Field |
| --- | --- |
| `message` | `message` |
| `severity` (error/warning/info/hint) | `severity` (Error/Warning/Information/Hint) |
| Match range | `range` (start line/col, end line/col) |
| `id` | `code` or `codeDescription` |
| `note` | `relatedInformation` or hover content |

Sources: CHANGELOG.md v0.39.5

### Code Actions (Quick Fixes)

The LSP server provides code actions when rules include a `fix` field.

**Code Action Flow:**

1.  Editor requests code actions at cursor position
2.  Server checks if cursor is within a diagnostic's range
3.  If the rule has a fix, server generates `CodeAction`
4.  Code action includes `TextEdit` with the fix content
5.  User applies fix via editor's code action UI

**Enhanced Features:**

From CHANGELOG v0.39.7, the LSP supports `expandStart` and `expandEnd` in fixes, allowing quick fixes to modify a broader range than the matched node.

Sources: CHANGELOG.md v0.39.7, v0.39.4

### Hover Information

The LSP server provides hover information displaying rule metadata when the cursor is over a matched node. Based on CHANGELOG v0.38.3 "add note in lsp hover info", the hover includes:

-   Rule `message`
-   Rule `note` (detailed explanation)
-   Severity level
-   Rule `id`
-   Potentially the matched pattern and captured meta-variables

Sources: CHANGELOG.md v0.38.3

---

## Rule Management and Reloading

### Dynamic Rule Loading

The rule finder closure pattern enables **hot-reloading** of rules without restarting the LSP server. This is implemented through file watchers (mentioned in CHANGELOG v0.39.4: "update file watchers").

**Rule Reload Trigger Points:**

1.  **Configuration File Changes**: When `sgconfig.yml` is modified
2.  **Rule File Changes**: When files in `rule_dirs` are added/modified/deleted
3.  **Utility Rule Changes**: When files in `util_dirs` change
4.  **Manual Reload**: Via LSP custom commands (if implemented)

**Reload Sequence:**

1.  File watcher detects change
2.  `Backend` re-invokes `rule_finder` closure
3.  New `RuleCollection` is built
4.  All open documents are re-analyzed
5.  Updated diagnostics are published

Sources: [crates/cli/src/lsp.rs20-23](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L20-L23) [crates/cli/src/config.rs85-91](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L85-L91) CHANGELOG.md v0.39.4

### Rule Discovery Process

The `ProjectConfig.find_rules()` method orchestrates the complete rule discovery pipeline:

1.  **Find Utility Rules**: [crates/cli/src/config.rs133-162](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L133-L162)
    -   Walk `util_dirs` using `WalkBuilder`
    -   Parse utility rules into `GlobalRules`
    -   Make utilities available to all rules
2.  **Walk Rule Directories**: [crates/cli/src/config.rs164-206](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L164-L206)
    -   Iterate through each path in `rule_dirs`
    -   Use `ignore` crate's `WalkBuilder` with config file type filter
    -   Read and parse each YAML file
3.  **Apply Rule Overwrite**: [crates/cli/src/config.rs197](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L197-L197)
    -   `RuleOverwrite::default()` applies no filtering for LSP
4.  **Build RuleCollection**: [crates/cli/src/config.rs198](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L198-L198)
    -   `RuleCollection::try_new(configs)` validates rules
    -   Compiles patterns and creates efficient matchers

Sources: [crates/cli/src/config.rs85-206](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L85-L206)

---

## Document Management

### Document State Storage

Based on the `dashmap` dependency in [crates/lsp/Cargo.toml23](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L23-L23) the server likely uses a concurrent hashmap for document storage:

```rust
type DocumentMap = DashMap<Url, DocumentState>;
```

The `DashMap` provides:

-   **Concurrent access**: Multiple LSP requests can read documents simultaneously
-   **Fine-grained locking**: Each document locked independently
-   **Efficient updates**: No global lock for document modifications

### Document Lifecycle

**Lifecycle Handlers:**

-   `textDocument/didOpen`: Insert document into `DashMap`, trigger initial analysis
-   `textDocument/didChange`: Update content, reparse, update diagnostics
-   `textDocument/didSave`: Optionally trigger additional validation
-   `textDocument/didClose`: Remove document from `DashMap`, clean up resources

Sources: [crates/lsp/Cargo.toml23](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L23-L23)

---

## Editor Integration

### VS Code Extension

The LSP server is designed to integrate with any LSP-compatible editor. For VS Code, the integration typically involves:

**Extension Configuration (`package.json`):**

```json
{
  "contributes": {
    "languages": [{
      "id": "javascript",
      "extensions": [".js"]
    }],
    "commands": [{
      "command": "ast-grep.restart",
      "title": "Restart ast-grep Server"
    }]
  }
}
```

**Server Launch:**

The extension spawns the LSP server as a child process:

-   Executable: `ast-grep lsp` or `sg lsp`
-   Working directory: Project root containing `sgconfig.yml`
-   Communication: stdio (stdin/stdout)

**Server Capabilities Registration:**

The LSP server advertises these capabilities during initialization:

-   `textDocumentSync`: Full or incremental document sync
-   `diagnosticProvider`: Provides diagnostics
-   `codeActionProvider`: Provides quick fixes
-   `hoverProvider`: Provides hover information

Sources: Section 6.2 in wiki TOC for "Editor Integration", standard LSP patterns

### Neovim and Other Editors

**Neovim Built-in LSP:**

```lua
require('lspconfig').ast_grep.setup({
  cmd = {"ast-grep", "lsp"},
  filetypes = {"javascript", "typescript", "python", "rust"},
  root_dir = require('lspconfig.util').root_pattern("sgconfig.yml"),
})
```

**Configuration Requirements:**

For any editor integration:

1.  **Project Detection**: Editor must find `sgconfig.yml` to determine project root
2.  **Working Directory**: LSP server must be started in directory with `sgconfig.yml`
3.  **Language Mapping**: Editor must map file types to languages ast-grep supports
4.  **Diagnostic Display**: Editor must render diagnostics with severity levels

Sources: [crates/cli/src/lsp.rs17](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L17-L17) [crates/cli/src/config.rs68-83](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/config.rs#L68-L83)

---

## Error Handling

### Client Capability Detection

From CHANGELOG v0.39.5: "store client cap and do not send workspace folder req", the LSP server adapts its behavior based on client capabilities:

**Capability Checks:**

-   `window.workDoneProgress`: Whether to show progress notifications
-   `textDocument.publishDiagnostics`: Whether client supports diagnostics
-   `workspace.applyEdit`: Whether client supports edit application
-   `workspace.workspaceFolders`: Whether client supports multi-root workspaces

The v0.39.5 update specifically added support for "LSP clients without publish diagnostics data support", meaning the server gracefully degrades when diagnostics aren't supported.

Sources: CHANGELOG.md v0.39.5

### Error Contexts

The CLI's error handling includes specific error contexts for LSP:

| Error Context | Trigger | Handler |
| --- | --- | --- |
| `StartLanguageServer` | Tokio runtime creation fails | Return setup error to CLI |
| `ProjectNotExist` | No `sgconfig.yml` found | Notify client, operate in limited mode |
| `ReadConfiguration` | Cannot read sgconfig | Log error, use default config |
| `ParseConfiguration` | Invalid YAML in sgconfig | Report error, disable rules |

Sources: [crates/cli/src/lsp.rs35](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lsp.rs#L35-L35)

---

## Performance Considerations

### Concurrent Request Handling

The LSP server's architecture enables efficient concurrent operations:

**Parallel Processing:**

1.  **Document Parsing**: Each document parsed in tokio task
2.  **Rule Matching**: Worker threads process rules
3.  **Diagnostic Publishing**: Non-blocking via async client API
4.  **Rule Reloading**: Background task, doesn't block active requests

**Shared State:**

-   `DashMap<Url, DocumentState>`: Lock-free reads for most operations
-   `RuleCollection`: Immutable after loading, shared via `Arc`
-   AST nodes: Reference-counted or borrowed, minimal copying

### Incremental Updates

The LSP server supports incremental document sync (standard LSP feature):

-   `textDocument/didChange` with `contentChanges` array
-   Only changed portions need reparsing
-   Diagnostics regenerated only for affected regions
-   Reduces CPU usage for large files with small edits

Sources: [crates/lsp/Cargo.toml32-40](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L32-L40)

---

## Testing

The LSP crate includes test infrastructure using tokio-test utilities.

**Test Dependencies from [crates/lsp/Cargo.toml29-40](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L29-L40):**

-   `futures = "0.3.31"`: Future combinators for testing
-   `tokio` with features: `rt-multi-thread`, `io-std`, `io-util`, `macros`, `time`
-   `tokio-stream = "0.1.17"`: Stream utilities
-   `tokio-util = "0.7.16"`: Additional tokio utilities

The CHANGELOG v0.39.4 mentions "Complete LSP rule reloading implementation with tests", confirming test coverage for the rule reloading feature.

Sources: [crates/lsp/Cargo.toml29-40](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/lsp/Cargo.toml#L29-L40) CHANGELOG.md v0.39.4
