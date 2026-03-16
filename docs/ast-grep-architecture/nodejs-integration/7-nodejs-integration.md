# Node.js Integration

> **Source:** https://deepwiki.com/ast-grep/ast-grep/7-node.js-integration

Relevant source files

-   [crates/napi/__test__/index.spec.ts](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts)
-   [crates/napi/index.d.ts](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.d.ts)
-   [crates/napi/index.js](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js)
-   [crates/napi/package.json](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/package.json)
-   [crates/napi/src/find_files.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs)
-   [crates/napi/src/lib.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs)
-   [crates/napi/src/sg_node.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs)
-   [crates/napi/yarn.lock](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/yarn.lock)

This page documents the `@ast-grep/napi` package, which provides Node.js bindings for ast-grep through native Node-API (NAPI) modules. This allows JavaScript and TypeScript applications to leverage ast-grep's pattern matching and AST manipulation capabilities with near-native performance.

For information about the CLI wrapper distributed via npm, see [npm CLI Wrapper](/ast-grep/ast-grep/9.1-npm-cli-wrapper). For TypeScript type definitions and API usage in detail, see [NAPI API Reference](/ast-grep/ast-grep/7.1-napi-api-reference).

## Overview

The Node.js integration consists of Rust code in [crates/napi/](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/) that exposes ast-grep functionality through NAPI bindings. The package is distributed as `@ast-grep/napi` on npm with platform-specific native binaries for 9 different architectures.

**Architecture Diagram: NAPI Module Structure**

```
┌─────────────────────────────────────────────────────────────────────┐
│                         JavaScript Layer                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │  index.js       │  │  index.d.ts     │  │ Language Modules│    │
│  │  (loader)       │  │  (types)        │  │ (js, ts, etc.)  │    │
│  └────────┬────────┘  └─────────────────┘  └────────┬────────┘    │
│           │                                          │              │
│           └─────────────────┬────────────────────────┘              │
│                             ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Native Binary                              │  │
│  │  @ast-grep/napi-{platform}-{arch}                             │  │
│  │  (e.g., @ast-grep/napi-darwin-arm64)                         │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
└─────────────────────────────┼───────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Rust Layer                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │  lib.rs         │  │  sg_node.rs     │  │  find_files.rs  │    │
│  │  (exports)      │  │  (SgNode/SgRoot)│  │  (async tasks)  │    │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │
│           │                    │                    │              │
│           └────────────────────┼────────────────────┘              │
│                                ▼                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    ast-grep-core                              │  │
│  │  (pattern matching, AST traversal, editing)                  │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

Sources: [crates/napi/src/lib.rs1-128](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs#L1-L128) [crates/napi/index.js1-412](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L1-L412) [crates/napi/index.d.ts1-15](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.d.ts#L1-L15)

## Core API Functions

The main API surface is exposed through [crates/napi/src/lib.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs) which defines the primary functions available to JavaScript consumers.

### Parse Functions

**`parse(lang: string, src: string): SgRoot`**

Synchronously parses source code into an AST. The `lang` parameter accepts language names like `"javascript"`, `"typescript"`, `"python"`, etc.

**`parseAsync(lang: string, src: string): Promise<SgRoot>`**

Asynchronously parses source code in a separate thread pool. This is useful for processing multiple files concurrently without blocking the main Node.js event loop. The implementation uses NAPI's `AsyncTask` mechanism defined in [crates/napi/src/find_files.rs17-34](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L17-L34)

Sources: [crates/napi/src/lib.rs68-83](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs#L68-L83) [crates/napi/src/find_files.rs17-34](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L17-L34)

### Pattern and Kind Functions

**`pattern(lang: string, pattern: string): NapiConfig`**

Compiles a pattern string into a reusable configuration object. This is useful for creating matcher objects that can be used with `find` and `findAll` methods.

**`kind(lang: string, kindName: string): number`**

Converts a string representation of a syntax node kind (e.g., `"member_expression"`) into its numeric identifier used by the tree-sitter parser.

Sources: [crates/napi/src/lib.rs86-105](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs#L86-L105)

### File Discovery Functions

**`findInFiles(lang: string, config: FindConfig, callback: Function): Promise<number>`**

Discovers files matching the specified paths and applies pattern matching across them. The callback receives arrays of `SgNode` objects for each file with matches. Returns the total number of files processed.

**`parseFiles(paths: string[] | FileOption, callback: Function): Promise<number>`**

Parses multiple files in parallel and invokes the callback with each parsed `SgRoot`. The `FileOption` allows specifying custom language glob patterns.

Sources: [crates/napi/src/lib.rs111-119](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs#L111-L119) [crates/napi/src/find_files.rs88-111](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L88-L111) [crates/napi/src/find_files.rs164-186](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L164-L186)

### Dynamic Language Registration

**`registerDynamicLanguage(langs: Object): void`**

Registers custom tree-sitter parsers at runtime. This allows adding support for languages not built into the package.

Sources: [crates/napi/src/lib.rs123-127](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs#L123-L127)

## Data Structures

**Data Flow: Rust to JavaScript Type Mapping**

```
Rust Type                    JavaScript Type
────────────────────────────────────────────────
SgRoot                       SgRoot class
  └─ AstGrep<JsDoc>            └─ root(): SgNode
                                └─ filename(): string

SgNode                       SgNode class
  └─ SharedReference<...>      └─ text(): string
                                └─ kind(): string
                                └─ range(): Range
                                └─ children(): SgNode[]
                                └─ find(pattern): SgNode | null
                                └─ findAll(pattern): SgNode[]
                                └─ replace(text): Edit

NapiConfig                   Object
  └─ RuleConfig<JsDoc>         └─ { rule, constraints, language, ... }

Edit                         Edit object
  └─ sg_core::Edit             └─ { startPos, endPos, insertedText }

Range                        Range object
  └─ (Pos, Pos)                └─ { start: Pos, end: Pos }

Pos                          Pos object
  └─ (line, col, byte)         └─ { line, column, index }
```

Sources: [crates/napi/src/sg_node.rs12-418](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L12-L418)

### SgRoot Class

Represents a parsed syntax tree. Contains the root `AstGrep<JsDoc>` instance and the filename.

**Methods:**

-   `root(): SgNode` - Returns the root node of the tree
-   `filename(): string` - Returns the file path or `"anonymous"` for `parse()` results

Defined in [crates/napi/src/sg_node.rs417-434](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L417-L434)

### SgNode Class

Represents a single node in the AST with pattern matching and traversal capabilities.

**Core Properties:**

-   `range(): Range` - Returns the source location
-   `text(): string` - Returns the node's source text
-   `kind(): string` - Returns the node kind name
-   `id(): number` - Returns unique node identifier
-   `isLeaf(): boolean`, `isNamed(): boolean`, `isNamedLeaf(): boolean` - Node property checks

**Pattern Matching:**

-   `matches(pattern: string | number | NapiConfig): boolean` - Tests if node matches a pattern
-   `inside(matcher): boolean`, `has(matcher): boolean`, `precedes(matcher): boolean`, `follows(matcher): boolean` - Relational matchers

**Traversal:**

-   `find(matcher): SgNode | null` - Finds first matching descendant
-   `findAll(matcher): SgNode[]` - Finds all matching descendants
-   `children(): SgNode[]` - Returns all child nodes
-   `parent(): SgNode | null`, `next(): SgNode | null`, `prev(): SgNode | null` - Tree navigation
-   `field(name: string): SgNode | null` - Accesses named field
-   `fieldChildren(name: string): SgNode[]` - Returns all children in a field

**Meta-variables:**

-   `getMatch(name: string): SgNode | null` - Retrieves single meta-variable match
-   `getMultipleMatches(name: string): SgNode[]` - Retrieves multi-matches (e.g., `$$$VAR`)
-   `getTransformed(name: string): string | null` - Gets transformed meta-variable

**Editing:**

-   `replace(text: string): Edit` - Creates replacement edit
-   `commitEdits(edits: Edit[]): string` - Applies edits and returns new text

Defined in [crates/napi/src/sg_node.rs40-414](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L40-L414)

Sources: [crates/napi/src/sg_node.rs40-440](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L40-L440)

### Configuration Types

| Type | Purpose | Key Fields |
| --- | --- | --- |
| `NapiConfig` | Pattern matching configuration | `rule`, `constraints`, `language`, `utils`, `transform` |
| `FindConfig` | File discovery configuration | `paths`, `matcher`, `languageGlobs` |
| `FileOption` | Parse files configuration | `paths`, `languageGlobs` |
| `Edit` | Code modification descriptor | `startPos`, `endPos`, `insertedText` |
| `Range` | Source location span | `start: Pos`, `end: Pos` |
| `Pos` | Source position | `line`, `column`, `index` |

Sources: [crates/napi/src/doc.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/doc.rs) [crates/napi/src/find_files.rs82-162](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L82-L162) [crates/napi/src/sg_node.rs12-38](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L12-L38)

## Language-Specific Modules

The package provides language-specific modules that pre-configure functions for common languages. Each module exports the same API but with the language parameter already bound.

**Language Module Implementation**

```rust
// Rust side - language enum mapping
fn lang_to_module(lang: SupportLang) -> &'static str {
    match lang {
        SupportLang::JavaScript => "js",
        SupportLang::TypeScript => "ts",
        SupportLang::Tsx => "tsx",
        SupportLang::Html => "html",
        SupportLang::Css => "css",
        // ...
    }
}
```

Sources: [crates/napi/src/lib.rs21-65](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs#L21-L65)

**Available Modules:**

-   `js` / `jsx` - JavaScript (uses `SupportLang::JavaScript`)
-   `ts` - TypeScript (uses `SupportLang::TypeScript`)
-   `tsx` - TypeScript JSX (uses `SupportLang::Tsx`)
-   `html` - HTML (uses `SupportLang::Html`)
-   `css` - CSS (uses `SupportLang::Css`)

Each module provides:

-   `parse(src: string): SgRoot`
-   `parseAsync(src: string): Promise<SgRoot>`
-   `kind(kindName: string): number`
-   `pattern(pattern: string): NapiConfig`
-   `findInFiles(config: FindConfig, callback: Function): Promise<number>`

Example usage from tests: [crates/napi/__test__/index.spec.ts4-5](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L4-L5)

Sources: [crates/napi/src/lib.rs60-65](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/lib.rs#L60-L65)

## Platform Support and Binary Loading

The package supports 9 platform/architecture combinations through separate native binary packages.

**Supported Platform Matrix:**

| Platform | Architectures | Package Names |
| --- | --- | --- |
| macOS | x64, arm64, universal | `@ast-grep/napi-darwin-x64`, `@ast-grep/napi-darwin-arm64` |
| Windows | x64, ia32, arm64 | `@ast-grep/napi-win32-x64-msvc`, `@ast-grep/napi-win32-ia32-msvc`, `@ast-grep/napi-win32-arm64-msvc` |
| Linux (GNU) | x64, arm64 | `@ast-grep/napi-linux-x64-gnu`, `@ast-grep/napi-linux-arm64-gnu` |
| Linux (musl) | x64, arm64 | `@ast-grep/napi-linux-x64-musl`, `@ast-grep/napi-linux-arm64-musl` |

Configuration in [crates/napi/package.json22-35](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/package.json#L22-L35)

**Native Binding Loading Flow**

```
require('@ast-grep/napi')
       │
       ▼
┌──────────────────────────┐
│    index.js loader       │
│    (platform detection)  │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│    Environment override? │
│    NAPI_RS_NATIVE_...    │
└───────────┬──────────────┘
            │ No
            ▼
┌──────────────────────────┐
│    Platform detection    │
│    (win32/darwin/linux)  │
└───────────┬──────────────┘
            │
            ▼
┌──────────────────────────┐
│    Load native binary    │
│    from platform package │
└───────────┬──────────────┘
            │ Failed
            ▼
┌──────────────────────────┐
│    WASI fallback         │
│    (if available)        │
└──────────────────────────┘
```

Sources: [crates/napi/index.js66-395](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L66-L395)

### Platform Detection Details

The loader implements sophisticated platform detection:

1.  **Environment Variable Override**: If `NAPI_RS_NATIVE_LIBRARY_PATH` is set, loads from that path [crates/napi/index.js67-72](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L67-L72)
2.  **Android Detection**: Checks for `process.platform === 'android'` and `process.arch` [crates/napi/index.js73-98](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L73-L98)
3.  **Windows Detection**: Checks for x64, ia32, and arm64 architectures [crates/napi/index.js99-135](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L99-L135)
4.  **macOS Detection**: Attempts universal binary first, then falls back to architecture-specific [crates/napi/index.js136-171](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L136-L171)
5.  **Linux Detection**: Implements musl detection via three methods:
    -   Filesystem check: reads `/usr/bin/ldd` [crates/napi/index.js29-35](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L29-L35)
    -   Process report: checks `process.report.getReport()` [crates/napi/index.js37-55](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L37-L55)
    -   Child process: executes `ldd --version` [crates/napi/index.js57-64](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L57-L64)
6.  **WASI Fallback**: Attempts WebAssembly System Interface version if native loading fails [crates/napi/index.js364-381](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L364-L381)
7.  **Error Handling**: Provides helpful error message about npm's optional dependencies bug [crates/napi/index.js383-393](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L383-L393)

Sources: [crates/napi/index.js13-395](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/index.js#L13-L395)

## Usage Patterns

### Basic Pattern Matching

```typescript
import { js } from '@ast-grep/napi';

const sg = js.parse(`
  function hello() {
    console.log('world');
  }
`);

const matches = sg.root().findAll('console.log($MSG)');
console.log(matches.length); // 1
```

Test example: [crates/napi/__test__/index.spec.ts7-18](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L7-L18)

### Asynchronous Parsing

For concurrent file processing, use `parseAsync` to avoid blocking the event loop:

```typescript
import { js } from '@ast-grep/napi';

const results = await Promise.all([
  js.parseAsync('const a = 1'),
  js.parseAsync('const b = 2'),
  js.parseAsync('const c = 3'),
]);
```

Test example: [crates/napi/__test__/index.spec.ts21-33](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L21-L33)

### Multi-File Discovery

The `findInFiles` function discovers and processes files in parallel:

```typescript
import { js } from '@ast-grep/napi';

await js.findInFiles(
  {
    paths: ['src/'],
    matcher: { rule: { pattern: 'console.log($MSG)' } },
  },
  (err, nodes) => {
    console.log(`Found ${nodes.length} matches`);
  }
);
```

Test examples: [crates/napi/__test__/index.spec.ts230-248](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L230-L248) [crates/napi/__test__/index.spec.ts272-288](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L272-L288)

### Code Editing

Apply multiple edits to transform code:

```typescript
import { js } from '@ast-grep/napi';

const sg = js.parse('var x = 1; var y = 2;');
const root = sg.root();

const edits = root.findAll('var $VAR = $VAL').map(n => n.replace('let $VAR = $VAL'));
const newText = root.commitEdits(edits);
console.log(newText); // 'let x = 1; let y = 2;'
```

Test example: [crates/napi/__test__/index.spec.ts98-107](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L98-L107)

### Relational Matching

Use relational methods for contextual queries:

```typescript
import { js } from '@ast-grep/napi';

const sg = js.parse('if (x) { console.log(x); }');
const root = sg.root();

const ifNode = root.find('if ($COND) { $$$BODY }');
const consoleLog = root.find('console.log($X)');

// Check if console.log is inside the if statement
console.log(consoleLog.inside(ifNode)); // true
```

Test examples: [crates/napi/__test__/index.spec.ts435-447](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L435-L447) [crates/napi/__test__/index.spec.ts449-461](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L449-L461)

### Language-Specific Globs

Override file extension detection with custom globs:

```typescript
import { parseFiles } from '@ast-grep/napi';

await parseFiles(
  {
    paths: ['src/'],
    languageGlobs: {
      javascript: ['*.js', '*.jsx', '*.cjs', '*.mjs'],
    },
  },
  (err, sgRoot) => {
    console.log(`Parsed: ${sgRoot.filename()}`);
  }
);
```

Test example: [crates/napi/__test__/index.spec.ts323-341](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L323-L341)

Sources: [crates/napi/__test__/index.spec.ts1-462](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/__test__/index.spec.ts#L1-L462)

## Implementation Details

### JsDoc and UTF-16 Encoding

The NAPI bindings use a custom `JsDoc` wrapper [crates/napi/src/doc.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/doc.rs) that handles JavaScript's UTF-16 string encoding. Tree-sitter works with bytes, but JavaScript uses UTF-16 code units, requiring conversion.

Byte positions are divided by 2 when converting to JavaScript positions: [crates/napi/src/sg_node.rs383-388](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L383-L388)

### Thread Safety and SharedReference

`SgNode` uses NAPI's `SharedReference` to maintain references to the parent `SgRoot` across JavaScript calls. This ensures the underlying AST remains valid while nodes are being accessed. The implementation is in [crates/napi/src/sg_node.rs40-43](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L40-L43)

### Async Task Execution

Both `ParseAsync` and `FindInFiles` implement the `Task` trait from NAPI, allowing them to execute in libuv's thread pool:

-   `compute()` runs in a worker thread
-   `resolve()` marshals results back to the main thread

Implementation: [crates/napi/src/find_files.rs22-34](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L22-L34) [crates/napi/src/find_files.rs45-73](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L45-L73)

### Pinned Node Data

For `findInFiles`, matches are collected using `PinnedNodeData` which prevents node invalidation when transferring between threads. The `PinnedNodes` wrapper [crates/napi/src/find_files.rs145-150](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L145-L150) is marked as `Send` and `Sync` to allow thread-safe transfer.

Sources: [crates/napi/src/doc.rs](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/doc.rs) [crates/napi/src/sg_node.rs40-440](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/sg_node.rs#L40-L440) [crates/napi/src/find_files.rs17-231](https://github.com/ast-grep/ast-grep/blob/3ae01aec/crates/napi/src/find_files.rs#L17-L231)
