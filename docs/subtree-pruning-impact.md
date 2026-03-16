# Subtree Pruning DFS: Implementation & Impact Analysis

## Overview

This document describes the subtree pruning optimization implemented in `CombinedScan::scan()` — the core multi-rule scanning engine in ast-grep. The optimization reduces the number of AST nodes visited during rule matching by skipping entire subtrees that provably cannot contain matches.

---

## What Changed

### 1. `PruningPre` Iterator (`crates/core/src/tree_sitter/traversal.rs`)

A new pre-order DFS iterator that supports `skip_children()` — after yielding a node, the caller can prevent descending into that node's children. This is a general-purpose building block for any pruning strategy.

### 2. Ancestor Kind Computation (`crates/config/src/combined.rs`)

Added `Suppressions::collect_with_ancestors()` which, during the existing suppression-collection DFS pass, also builds an `ancestor_kinds` BitSet. This set contains every node kind that appears as an **ancestor** of a node whose kind is in any rule's `potential_kinds`.

**How it works:**
- Maintains a `path_stack` of `(end_byte, kind_id)` entries tracking the current DFS ancestry path
- When a node's kind matches `potential_kinds`, all kinds on `path_stack` are added to `ancestor_kinds`
- Entries are popped from `path_stack` when the DFS moves past their byte range

### 3. Pruning DFS in `CombinedScan::scan()` (`crates/config/src/combined.rs`)

The main scan loop was changed from `root.root().dfs()` (visits every node) to a stack-based DFS that skips subtrees based on two BitSets:

- `all_potential_kinds`: union of all rules' `potential_kinds` — can this node directly match?
- `ancestor_kinds`: computed from the AST — can this node's descendants match?

**Pruning rule:** If a node's kind is NOT in `all_potential_kinds` AND NOT in `ancestor_kinds`, skip its entire subtree.

### 4. Precomputed `all_potential_kinds` in `CombinedScan`

`CombinedScan::new()` now computes the union of all rules' `potential_kinds` into a single `BitSet`, enabling O(1) checks during the pruning DFS.

---

## Why It Works

### Tree-sitter ASTs have structured kind hierarchies

In tree-sitter grammars, node kinds follow a hierarchical pattern:

```
program
  ├── function_declaration (kind=50)
  │   ├── identifier (kind=1)
  │   └── statement_block (kind=80)
  │       └── return_statement (kind=120)
  │           └── call_expression (kind=42)
  ├── class_declaration (kind=60)
  │   └── class_body (kind=61)
  │       └── method_definition (kind=62)
  │           └── ...
  └── lexical_declaration (kind=30)
      └── variable_declarator (kind=31)
```

If a rule matches `return_statement` (kind=120), only subtrees rooted at kinds that are ancestors of `return_statement` (i.e., `program`, `function_declaration`, `statement_block`, etc.) need to be traversed. Subtrees like `lexical_declaration → variable_declarator → identifier` can be entirely skipped.

### File-specific optimization

Unlike static grammar analysis, our approach builds `ancestor_kinds` from the **actual AST** of each file. This means:

- It adapts to the specific file's structure
- It's always correct (no false negatives)
- It captures only the ancestor kinds present in this file, not all theoretically possible ones

---

## Expected Impact

### Reduction in nodes visited

| Scenario | Estimated Reduction |
|----------|-------------------|
| Specific pattern (e.g., `return $A`) in large files | **30-60%** fewer nodes visited |
| Multiple rules targeting few kinds | **20-50%** fewer nodes visited |
| Rules targeting common kinds (e.g., `identifier`) | **5-10%** fewer nodes visited |
| Wildcard rules (`potential_kinds = None`) | **0%** (no pruning possible) |

### When pruning is most effective

1. **Deeply nested ASTs with specific targets**: Searching for `return_statement` in a file with many top-level declarations — all non-function subtrees are skipped.

2. **Large files with many rule kinds**: When rules target 5-10 specific kinds out of 300+ possible kinds, most subtrees are prunable.

3. **Multi-rule scans (`sg scan`)**: The union of all rules' kinds determines pruning — even with many rules, if they target a small subset of kinds, pruning is effective.

### When pruning has minimal impact

1. **Rules matching very common kinds** (e.g., `identifier`, `string`): These appear at every level of the AST, so `ancestor_kinds` includes most kinds.

2. **Small files**: The overhead of building `ancestor_kinds` (O(N) during the suppression pass) is comparable to the savings from pruning.

3. **Wildcard matchers**: Rules without `potential_kinds` (returning `None`) cannot benefit from pruning.

### Overhead

- **Memory**: One additional `BitSet` (~64 bytes for typical grammars with <512 kinds) plus the `path_stack` Vec during the suppression pass.
- **Time**: The ancestor kind computation adds O(depth) work per matching-kind node during the suppression pass. For typical ASTs (depth 5-15), this is negligible.
- **Stack-based DFS**: The scan loop now uses `Vec<Node>` instead of the cursor-based `dfs()` iterator. This allocates a Vec but avoids re-traversing nodes that would be skipped.

---

## Correctness Guarantees

1. **No false negatives**: A subtree is only skipped if its root kind is NOT in `potential_kinds` AND NOT in `ancestor_kinds`. Since `ancestor_kinds` is built from the actual AST, any node that could transitively contain a match has its ancestor kinds recorded.

2. **Suppression ordering preserved**: Suppressions are still collected in a pre-pass before rule matching, maintaining correct ordering for same-line and file-level suppressions.

3. **Behavioral equivalence**: All existing tests pass without modification, confirming identical match results.

---

## Measurement

To measure the impact on a real codebase:

```bash
# Before optimization (on the previous commit):
hyperfine 'sg scan' --warmup 3

# After optimization:
hyperfine 'sg scan' --warmup 3

# Profile node visitation:
RUST_LOG=debug sg scan 2>&1 | grep "nodes visited"
```

To verify pruning effectiveness, add temporary counters: 
```rust
// In CombinedScan::scan(), add:
let mut visited = 0u64;
let mut pruned = 0u64;
// In the pruning check:
if !is_potential && !is_ancestor {
    pruned += 1;  // Count pruned subtree roots
    continue;
}
visited += 1;
// At the end:
eprintln!("visited: {visited}, pruned: {pruned}, ratio: {:.1}%", pruned as f64 / (visited + pruned) as f64 * 100.0);
```

---

## Files Modified

| File | Change |
|------|--------|
| `crates/core/src/tree_sitter/traversal.rs` | Added `PruningPre` iterator with `skip_children()` support |
| `crates/core/src/tree_sitter/mod.rs` | Exported `PruningPre` |
| `crates/config/src/combined.rs` | Added `all_potential_kinds` field, `collect_with_ancestors()` method, pruning DFS in `scan()` |
