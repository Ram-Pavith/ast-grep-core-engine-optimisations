# CLI Tool

> **Source:** https://deepwiki.com/ast-grep/ast-grep/5-cli-tool

Relevant source files

-   [crates/cli/src/lib.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lib.rs)
-   [crates/cli/src/main.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/main.rs)
-   [crates/cli/src/run.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs)
-   [crates/cli/src/scan.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs)

The CLI tool provides the command-line interface for ast-grep, implementing six main commands for different use cases: ad-hoc pattern searches (`run`), rule-based scanning (`scan`), rule testing (`test`), project scaffolding (`new`), language server (`lsp`), and shell completion generation (`completions`). The CLI handles argument parsing, file traversal, pattern matching coordination, and output formatting across multiple modes (colored terminal, JSON, SARIF, interactive).

For detailed documentation on rule-based analysis, see [Rule System](/ast-grep/ast-grep/3-rule-system). For language server integration, see [Language Server Protocol](/ast-grep/ast-grep/6-language-server-protocol). For output format specifications, see [Output Formatting](/ast-grep/ast-grep/5.5-output-formatting).

## Entry Point and Command Dispatch

The CLI uses a two-tier argument parsing strategy that supports both explicit subcommands (`sg run -p pattern`) and implicit default command invocation (`sg -p pattern`).

**Entry Point Flow Diagram**: Shows how `main()` delegates to `execute_main()` and `main_with_args()`, which performs early project configuration loading and command detection before dispatching to command handlers.

Sources: [crates/cli/src/main.rs1-9](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/main.rs#L1-L9) [crates/cli/src/lib.rs68-146](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lib.rs#L68-L146)

The argument parsing occurs in three phases:

1.  **Project Discovery** ([lib.rs102-123](https://github.com/ast-grep/ast-grep/blob/3ae01aec/lib.rs#L102-L123)): Scans arguments for `--config`/`-c` flag and calls `ProjectConfig::setup()` to load `sgconfig.yml`. This happens before command parsing to allow `sg help` to work without valid configuration.
    
2.  **Default Command Detection** ([lib.rs88-99](https://github.com/ast-grep/ast-grep/blob/3ae01aec/lib.rs#L88-L99)): Checks if arguments contain `--pattern`/`-p` flag without a subcommand name, enabling `sg -p 'pattern'` shorthand for `sg run -p 'pattern'`.
    
3.  **Full Command Parsing** ([lib.rs134](https://github.com/ast-grep/ast-grep/blob/3ae01aec/lib.rs#L134-L134)): Uses `clap` to parse the `App` structure, which contains a `Commands` enum with six variants.
    

## Command Structure

| Command | Purpose | Configuration Required | Primary Arguments |
| --- | --- | --- | --- |
| `run` | Ad-hoc pattern search/replace | No (optional) | `--pattern`, `--lang`, `--rewrite` |
| `scan` | Rule-based code analysis | Yes (or `--rule`/`--inline-rules`) | `--rule`, `--inline-rules`, `--format` |
| `test` | Rule verification with test cases | Yes | `--update-all`, `--skip-snapshot-tests` |
| `new` | Project/rule scaffolding | Depends on item type | Item type (project/rule/test/util) |
| `lsp` | Start language server | Yes | Port and configuration options |
| `completions` | Generate shell completions | No | Shell name (bash/zsh/fish) |

Sources: [crates/cli/src/lib.rs49-66](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/lib.rs#L49-L66)

For detailed command-specific documentation, see:

-   [Commands Overview](/ast-grep/ast-grep/5.1-commands-overview)
-   [Run Command](/ast-grep/ast-grep/5.2-run-command)
-   [Scan Command](/ast-grep/ast-grep/5.3-scan-command)
-   [New Command](/ast-grep/ast-grep/5.4-new-command)

## Worker Pattern Architecture

Both `run` and `scan` commands use a common `Worker` trait system to handle file processing in parallel. The architecture separates concerns between argument parsing, file walking, pattern matching, and output formatting.

**Worker Pattern Trait Hierarchy**: Shows how different command implementations use the `Worker`, `PathWorker`, and `StdInWorker` traits to standardize file processing while allowing command-specific behavior.

Sources: [crates/cli/src/utils/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/utils/mod.rs) [crates/cli/src/run.rs187-344](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L187-L344) [crates/cli/src/scan.rs129-374](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs#L129-L374)

### Worker Trait Responsibilities

The `Worker` trait ([utils/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/mod.rs)) defines the core interface:

This trait's `consume_items()` method receives an iterator of processed items and a `Printer`, coordinates output, and returns an exit code indicating success/failure.

The `PathWorker` trait extends `Worker` for file-based processing:

-   `build_walk()` - Creates a `WalkParallel` instance configured with appropriate file filters
-   `produce_item()` - Processes a single file path, returning processed items for the printer
-   `get_trace()` - Returns a `FileTrace` for diagnostics
-   `should_stop()` - Allows early termination (e.g., when `--max-results` is reached)

The `StdInWorker` trait extends `Worker` for stdin processing:

-   `parse_stdin()` - Parses stdin content and returns processed items

### Language Inference Strategy

The `run` command uses two different worker implementations depending on whether language is specified:

**RunWithSpecificLang** ([run.rs260-344](https://github.com/ast-grep/ast-grep/blob/3ae01aec/run.rs#L260-L344)):

-   Language provided via `--lang` flag
-   Pattern compiled once during initialization
-   Files filtered by specified language's extensions
-   Used when reading from stdin (language required)

**RunWithInferredLang** ([run.rs187-258](https://github.com/ast-grep/ast-grep/blob/3ae01aec/run.rs#L187-L258)):

-   No `--lang` flag provided
-   Language detected from file extension for each file
-   Pattern recompiled for each detected language
-   Supports language injection (e.g., JavaScript in HTML)

**Language Inference Decision Tree**: Shows how the `run` command selects between `RunWithSpecificLang` and `RunWithInferredLang` based on arguments, and how each handles file filtering and pattern compilation.

Sources: [crates/cli/src/run.rs151-185](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L151-L185) [crates/cli/src/run.rs187-258](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L187-L258) [crates/cli/src/run.rs260-344](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L260-L344)

## Scan Command Architecture

The `scan` command implements rule-based code analysis with support for multiple rules, severity filtering, and unused suppression detection.

**Scan Command Execution Flow**: Shows how `ScanArg` loads rules from different sources, filters by severity, and executes the `CombinedScan` engine against filtered files.

Sources: [crates/cli/src/scan.rs26-286](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs#L26-L286) [crates/cli/src/utils/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/utils/mod.rs)

### Rule Source Priority

The scan command accepts rules from three mutually exclusive sources ([scan.rs140-157](https://github.com/ast-grep/ast-grep/blob/3ae01aec/scan.rs#L140-L157)):

1.  **Single Rule File** (`--rule`/`-r`): Loads one YAML file via `read_rule_file()`, useful for testing individual rules
2.  **Inline Rules** (`--inline-rules`): Parses YAML directly from command line argument, supports multiple rules separated by `---`
3.  **Project Rules** (default): Loads all rules from directories specified in `sgconfig.yml` via `ProjectConfig::find_rules()`

### Severity Filtering

The `RuleOverwrite` system ([utils/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/mod.rs)) allows command-line severity overrides:

-   `--filter`: Comma-separated list of rule IDs to include
-   `--error`: Override rules to error severity
-   `--warning`: Override rules to warning severity
-   `--info`: Override rules to info severity
-   `--hint`: Override rules to hint severity
-   `--off`: Disable specific rules

### Unused Suppression Detection

The scan command includes special logic for detecting unused inline suppression comments ([scan.rs196-213](https://github.com/ast-grep/ast-grep/blob/3ae01aec/scan.rs#L196-L213)):

-   **Auto-enabled** when `include_all_rules()` returns true (no `--rule`, `--inline-rules`, or severity filters)
-   **Default severity** is `Hint` when enabled, `Off` otherwise
-   **Can be overridden** via `RuleOverwrite` for the special `unused-suppression` rule ID

This heuristic prevents false positives when scanning with a subset of rules, where suppressions might be for rules not currently active.

### Max Results Limiting

Both `scan` and `run` commands support `--max-results` ([scan.rs75](https://github.com/ast-grep/ast-grep/blob/3ae01aec/scan.rs#L75-L75) [run.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/run.rs)) for early termination:

**ScanWithConfig** ([scan.rs163-172](https://github.com/ast-grep/ast-grep/blob/3ae01aec/scan.rs#L163-L172)):

The `MaxItemCounter` uses atomic operations to track match count across parallel workers ([scan.rs253-264](https://github.com/ast-grep/ast-grep/blob/3ae01aec/scan.rs#L253-L264)):

-   Workers atomically claim slots via `counter.claim(wanted)`
-   Truncates match list when limit reached
-   Workers check `should_stop()` to terminate file walking early

## Printer Selection and Output Formatting

The CLI uses a trait-based printer system that separates output formatting from matching logic. Each command selects a printer implementation based on output flags.

**Printer Selection Flow**: Shows how output arguments determine which `Printer` implementation is used, and how all printers implement the common `Printer` trait.

Sources: [crates/cli/src/print/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/print/mod.rs) [crates/cli/src/run.rs151-174](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L151-L174) [crates/cli/src/scan.rs85-112](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs#L85-L112)

### Printer Trait Interface

Each `Printer` implementation must provide ([print/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/print/mod.rs)):

-   `type Processor`: Associated type implementing `PrintProcessor<Self::Processed>`
-   `type Processed`: The processed item type returned by workers
-   `before_print()`: Called once before any output
-   `process(item: Self::Processed)`: Called for each item
-   `after_print()`: Called once after all output

The `PrintProcessor` associated type handles the actual formatting logic:

-   `print_matches()`: Format pattern matches without fixes
-   `print_diffs()`: Format diffs when `--rewrite` is provided
-   `print_rule()`: Format rule matches (scan command)
-   `print_rule_diffs()`: Format rule diffs with auto-fixes

### Output Mode Decision Logic

**Run Command** ([run.rs151-174](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L151-L174)):

1.  Check `--files-with-matches` → `FileNamePrinter`
2.  Check `--json` → `JSONPrinter` with optional context
3.  Check `needs_interactive()` (i.e., `--interactive` or `--update-all`) → `InteractivePrinter` wrapping `ColoredPrinter`
4.  Default → `ColoredPrinter` with heading and context settings

**Scan Command** ([scan.rs85-112](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs#L85-L112)):

1.  Check `--files-with-matches` → `FileNamePrinter`
2.  Check `--format` → `CloudPrinter` (GitHub Actions or SARIF)
3.  Check `--json` → `JSONPrinter` with optional `--include-metadata`
4.  Check `needs_interactive()` → `InteractivePrinter` wrapping `ColoredPrinter`
5.  Default → `ColoredPrinter` with `--report-style` and context

### Special Output Features

**Context Lines** ([utils/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/mod.rs)):

-   `--before N`: Lines before match
-   `--after N`: Lines after match
-   `--context N`: Equivalent to `--before N --after N`

**JSON Output** ([print/json.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/print/json.rs)):

-   `--json compact`: Single-line JSON per match
-   `--json pretty`: Pretty-printed JSON (default when flag used without argument)
-   `--include-metadata`: Adds rule metadata to scan output (requires `--json`)

**Report Styles** ([print/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/print/mod.rs)):

-   `rich`: Full diff output with colors (default)
-   `short`: Condensed output
-   `medium`: Intermediate detail level

For detailed documentation on output formats and interactive mode, see [Output Formatting](/ast-grep/ast-grep/5.5-output-formatting) and [Interactive Mode](/ast-grep/ast-grep/5.6-interactive-mode).

Sources: [crates/cli/src/print/mod.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/print/mod.rs) [crates/cli/src/run.rs151-174](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L151-L174) [crates/cli/src/scan.rs85-112](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs#L85-L112)

## File Traversal and Filtering

Both `run` and `scan` commands use the `ignore` crate's `WalkParallel` for efficient multi-threaded file traversal with gitignore support.

**File Traversal Configuration**: Shows how `InputArgs` configures `WalkBuilder` with language-specific filters, gitignore rules, and parallel execution settings.

Sources: [crates/cli/src/utils/input.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/utils/input.rs)

### Language-Based Filtering

The `InputArgs::walk_lang()` and `walk_langs()` methods ([utils/input.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/input.rs)) use language metadata to filter files:

1.  Each `SgLang` provides `file_types()` method returning `ignore::types::Types`
2.  Types include file extensions and glob patterns for the language
3.  `WalkBuilder::types()` accepts the types configuration
4.  Files not matching any type are skipped during traversal

### Gitignore Integration

By default, the walker respects:

-   `.gitignore` files
-   `.ignore` files
-   `.git/info/exclude`
-   Global git ignore configuration

The `--no-ignore` flag family ([utils/input.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/input.rs)) allows selective disabling:

-   `--no-ignore`: Ignore all ignore files
-   `--no-ignore dot`: Ignore `.ignore` files only
-   `--no-ignore vcs`: Ignore VCS files only (`.gitignore`, etc.)
-   `--no-ignore global`: Ignore global git config only
-   `--no-ignore parent`: Don't read parent directory ignore files

### Custom Ignore Rules

The CLI supports `.ast-grep.ignore` files for ast-grep-specific patterns, loaded via `add_custom_ignore_filename()` ([utils/input.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/input.rs)).

### Parallel Execution

The `--threads`/`-j` flag ([utils/input.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/input.rs)) controls worker count:

-   `0` (default): Auto-detect based on CPU count
-   `1`: Single-threaded execution (useful for debugging)
-   `N > 1`: Specific worker count

Sources: [crates/cli/src/utils/input.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/utils/input.rs)

## Exit Codes and Error Handling

The CLI uses Rust's `ExitCode` type to communicate success/failure:

| Exit Code | Condition | Commands |
| --- | --- | --- |
| `0` | Success, matches found | `run`, `scan` |
| `1` | No matches found | `run` |
| `1` | Errors in matched rules | `scan` (when rules have `severity: error`) |
| `1` | Pattern parse error without matches | `run` (when pattern has syntax error) |
| Non-zero | Other errors | All commands |

**Run Command** ([run.rs206](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L206-L206) [run.rs304](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L304-L304)):

-   Returns `ExitCode(0)` if any matches found
-   Returns `ExitCode(1)` if no matches and pattern is valid
-   Returns error if pattern has syntax error and no matches (indicates pattern may be malformed)

**Scan Command** ([scan.rs186-192](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs#L186-L192)):

-   Tracks error count atomically across workers
-   Returns error via `anyhow::anyhow!(EC::DiagnosticError(error_count))` if any `severity: error` rules matched
-   This causes the command to exit with non-zero code, useful for CI/CD pipelines

The `EC` (ErrorContext) enum ([utils/error.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/utils/error.rs)) provides structured error messages for common failure modes.

Sources: [crates/cli/src/run.rs191-207](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L191-L207) [crates/cli/src/run.rs287-306](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/run.rs#L287-L306) [crates/cli/src/scan.rs175-193](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/scan.rs#L175-L193) [crates/cli/src/utils/error.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/cli/src/utils/error.rs)
