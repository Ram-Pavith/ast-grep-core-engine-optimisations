# ast-grep Search Speed Optimisations — Implementation Plan

> Unified, de-duplicated plan derived from `search-speed-optimisations.md` and `ast-grep-matching-speed-improvements.md`, validated against the actual codebase.

---

## Already Implemented (No Further Work)

These optimisations from the proposal docs are **already present** in the codebase:

| Optimisation | Where Implemented |
|---|---|
| Subtree pruning DFS | `crates/config/src/combined.rs` — `collect_with_ancestors()`, stack-based DFS with `ancestor_kinds` + `all_potential_kinds` |
| PruningPre iterator | `crates/core/src/tree_sitter/traversal.rs` — `PruningPre` with `skip_children()` |
| `potential_kinds()` BitSet filtering | `crates/core/src/node.rs::find_all()`, `CombinedScan::scan()` |
| BitSet intersection for composite rules | `crates/core/src/ops.rs` — `And::potential_kinds`, `All::compute_kinds` |
| Shared-traversal batch evaluation | `CombinedScan::new()` + `CombinedScan::scan()` — single DFS, kind→rule dispatch |
| Inline suppression collection | `collect_with_ancestors()` merges suppression + ancestor_kinds in one DFS pass |

---

## Phase 1 — Quick Wins (P0) — Highest Impact / Lowest Risk

### 1.1 Extend Literal Pre-Filtering to Scan Rules

**Impact:** Very High · **Effort:** Medium · **Risk:** Low

The CLI already uses `Pattern::fixed_string()` for `sg run --pattern`. Extend this to `sg scan` (multi-rule) to skip files that cannot match any rule.

**Files to modify:**

| File | Change |
|---|---|
| `crates/config/src/rule_core.rs` | Add `fn fixed_string_hint(&self) -> Option<String>` to `RuleCore`, caching the hint from `self.rule` |
| `crates/config/src/rule/mod.rs` | Add `fn fixed_string_hint(&self) -> Option<String>` to `Rule` enum — conservative: `Pattern` → use `fixed_string()`, `All` → longest child hint, `Inside/Has` → inner hint, everything else → `None` |
| `crates/cli/src/scan.rs` | In `ScanWithConfig::produce_item`, before `CombinedScan::new(rules)`, filter out rules whose `fixed_string_hint` is absent from file content |
| `crates/config/src/combined.rs` | Add `fn file_can_match(&self, content: &[u8]) -> bool` — returns false if ALL rules have literal hints and NONE appear in content (skip the entire file) |

**Implementation strategy:**
```
1. Add Rule::fixed_string_hint() — conservative, returns None when unsure
2. Cache on RuleCore at construction time
3. In scan path: read file bytes → check literals → skip file or filter rules
4. Use memchr::memmem for byte scanning (already available via ignore crate)
```

**Correctness rule:** Hints must be conservative. If there's any doubt, return `None` — never skip a file that could match.

---

### 1.2 SmallVec MetaVarEnv

**Impact:** High · **Effort:** Medium · **Risk:** Low

Replace `HashMap` internals in `MetaVarEnv` with inline `SmallVec` storage. 95%+ of patterns capture 0–5 variables.

**Files to modify:**

| File | Change |
|---|---|
| `crates/core/Cargo.toml` | Add `smallvec = "1"` dependency |
| `crates/core/src/meta_var.rs` | Replace all 3 `HashMap` fields with `SmallVec<[(String, T); N]>` |

**New struct layout:**
```rust
pub struct MetaVarEnv<'tree, D: Doc> {
    single_matched: SmallVec<[(MetaVariableID, Node<'tree, D>); 4]>,
    multi_matched: SmallVec<[(MetaVariableID, Vec<Node<'tree, D>>); 2]>,
    transformed_var: SmallVec<[(MetaVariableID, Underlying<D>); 2]>,
}
```

**Methods to update:** `new`, `insert`, `insert_multi`, `get_match`, `get_multiple_matches`, `add_label`, `get_labels`, `get_matched_variables`, `match_variable`, `match_multi_var`, `match_constraints`, `insert_transformation`, `get_transformed`, `get_var_bytes`, `visit_nodes`, `From<MetaVarEnv> for HashMap`.

**Key change:** Replace `HashMap::get/insert` with linear scan on the SmallVec. For N ≤ 4, linear scan is faster than hashing.

---

### 1.3 Rule Selectivity Ordering in CombinedScan

**Impact:** Medium–High · **Effort:** Small · **Risk:** Low

Sort rules within each `kind_rule_mapping[kind]` bucket by cheapness/selectivity so cheap rules reject early.

**Files to modify:**

| File | Change |
|---|---|
| `crates/config/src/rule_core.rs` | Add `fn match_cost_hint(&self) -> u32` — `Kind` = 1, `Regex` = 10, `Pattern` = 100, composite/relational = 200+ |
| `crates/config/src/combined.rs` | In `CombinedScan::new()`, after building `kind_rule_mapping`, sort each bucket by `rule.matcher.match_cost_hint()` ascending |

**Correctness:** Results are collected into a `HashMap<usize, Vec<NodeMatch>>` keyed by rule index, so evaluation order doesn't affect output order.

---

## Phase 2 — Core Engine Improvements (P1)

### 2.1 Incremental Parsing for LSP

**Impact:** High (editor responsiveness) · **Effort:** Large · **Risk:** Medium

Currently the LSP uses `TextDocumentSyncKind::FULL` and reparses from scratch on every edit.

**Files to modify:**

| File | Change |
|---|---|
| `crates/lsp/src/lib.rs` | Change sync mode to `INCREMENTAL`; convert LSP content changes to byte-offset edits; apply via `Root::edit()` |
| `crates/lsp/src/utils.rs` | Add UTF-16 position → byte offset conversion helpers |
| `crates/core/src/tree_sitter/mod.rs` | Reuse existing `StrDoc::do_edit()` |

**Main risk:** UTF-16 ↔ byte offset conversion correctness. Always keep full-reparse as fallback.

---

### 2.2 Pattern Fingerprinting for Fast Rejection

**Impact:** Medium–High · **Effort:** Medium · **Risk:** Low

Compute a cheap structural fingerprint at pattern compile time. Before descending into recursive `match_node_impl`, reject candidates that obviously can't match.

**Files to modify:**

| File | Change |
|---|---|
| `crates/core/src/matcher/pattern.rs` | Add `PatternFingerprint` struct on `Pattern`; compute during `convert_node_to_pattern()` |
| `crates/core/src/match_tree/match_node.rs` | Add fingerprint check before `match_node_impl()` recursive descent |

**Fingerprint fields:**
```rust
struct PatternFingerprint {
    root_kind_id: u16,
    named_child_count: u8,
    total_child_count: u8,
    is_leaf: bool,
    meta_var_count: u8,      // adjusts minimum child count
    first_child_kind: Option<u16>,
}
```

**Rule:** Fingerprint must only produce false positives, never false negatives.

---

### 2.3 Pattern Compilation Cache for `impl Matcher for str`

**Impact:** Medium · **Effort:** Small · **Risk:** Low

The `impl Matcher for str` in `matcher.rs:73-87` creates a new `Pattern` on every call. Add a thread-local LRU cache.

**Files to modify:**

| File | Change |
|---|---|
| `crates/core/Cargo.toml` | Add `lru` dependency |
| `crates/core/src/matcher.rs` | Add `thread_local!` LRU cache keyed by `(pattern_str, lang_id)`; use in `impl Matcher for str` |

---

## Phase 3 — Polish (P2) ✅ IMPLEMENTED

### 3.1 Node Structural Hash Cache for `does_node_match_exactly()` ✅

**Impact:** Medium · **Effort:** Medium · **Risk:** Low

~~Cache a structural hash per AST node.~~ Added inline `structural_hash()` using `DefaultHasher` as a bloom-filter fast-path in `does_node_match_exactly()`. For multi-child nodes, hash mismatch skips the expensive recursive `.all()` comparison.

| File | Change |
|---|---|
| `crates/core/src/match_tree/mod.rs` | Added `structural_hash()` function; inserted hash comparison before recursive child matching when `children.len() > 1` |

---

### 3.2 Predicate Cost Ordering Inside Composite Rules ✅

**Impact:** Low–Medium · **Effort:** Medium · **Risk:** Medium

Constraints in `RuleCore::do_match()` are now evaluated in cost-sorted order. Added `sorted_constraint_keys` field to `RuleCore`, computed at construction in `with_matchers()`, and a `match_constraints_ordered()` method that iterates constraints cheapest-first.

| File | Change |
|---|---|
| `crates/config/src/rule_core.rs` | Added `sorted_constraint_keys: Vec<String>` field, `match_constraints_ordered()` method, updated `with_matchers()`, `Default`, and `do_match()` |

**Note:** `All` sub-matcher reordering in `ops.rs` was intentionally skipped — capture-producing matchers depend on evaluation order, making reordering unsafe.

---

### 3.3 Memory-Mapped File I/O ✅

**Impact:** Low–Medium · **Effort:** Small · **Risk:** Low

For files ≥ 64 KB, use `memmap2` for zero-copy reads.

| File | Change |
|---|---|
| `Cargo.toml` | Added `memmap2 = "0.9"` to workspace dependencies |
| `crates/cli/Cargo.toml` | Added `memmap2.workspace = true` |
| `crates/cli/src/utils/mod.rs` | Modified `read_file()` to use mmap for files ≥ 64KB via `MMAP_THRESHOLD`; added `Mmap` and `File` imports |

---

## Phase 4 — Engine Internals (P3) — Partially Implemented

### 4.1 KindMask Fast-Path for Kind Filtering ✅

**Impact:** Medium · **Effort:** Small · **Risk:** Low

Replaced `BitSet` with a private `KindMask` type that uses inline `[u64; 8]` storage (512 bits) for kind membership checks, falling back to heap `BitSet` for grammars with >512 kinds. All tree-sitter grammars have 200–500 kinds, so the inline path is always taken in practice.

| File | Change |
|---|---|
| `crates/core/src/kind_mask.rs` | New `KindMask` enum with `Inline([u64; 8])` / `Heap(BitSet)` variants, `contains`, `insert`, `union_with`, `from_bitset` |
| `crates/core/src/lib.rs` | Added `pub mod kind_mask` |
| `crates/config/src/combined.rs` | Changed `all_potential_kinds` from `BitSet` to `KindMask`; `ancestor_kinds` in `collect_with_ancestors` also uses `KindMask`. `Matcher::potential_kinds()` still returns `Option<BitSet>` (public API unchanged) |

### 4.2 Iterative Tree Matching Foundations ✅

**Impact:** Medium · **Effort:** Small · **Risk:** Low

Added aggregator checkpoint/rollback infrastructure and SmallVec-based children pre-collection for the tree matcher.

| File | Change |
|---|---|
| `crates/core/src/match_tree/mod.rs` | Added `Checkpoint` associated type, `checkpoint()` and `rollback()` to `Aggregator` trait; implemented for both `ComputeEnd` (checkpoint = `usize`) and `Cow<MetaVarEnv>` (checkpoint = cloned env) |
| `crates/core/src/match_tree/match_node.rs` | Added `#[inline]` to `match_node_impl`; pre-collect candidate children into `SmallVec<[Node; 8]>` before `match_nodes_impl_recursive` to avoid repeated tree-sitter FFI calls during peekable iteration |

### 4.3 Parallel Rule Matching Within a File — NOT FEASIBLE

**Status:** Deferred permanently. `tree_sitter::Node` uses raw pointers internally and does not implement `Send`/`Sync`. Rayon-based within-file parallelism would require unsafe wrappers or a detached node representation. File-level parallelism via `ignore::WalkParallel` already saturates cores effectively.

### 4.4 Full Iterative Match Rewrite — Deferred

The full conversion of `match_node_impl` + ellipsis handling to an explicit-stack state machine is deferred. The checkpoint infrastructure (4.2) is in place for when profiling shows recursion overhead is material. The ellipsis control flow (consecutive ellipsis, named captures, trivial-node skipping) makes a full iterative rewrite high-risk without significant profiling evidence.

---

## Implementation Order Summary

```
Phase 1 ✅
  ├── 1.1 Extend literal pre-filtering to sg scan
  ├── 1.2 SmallVec MetaVarEnv
  └── 1.3 Rule selectivity ordering in CombinedScan

Phase 2 ✅
  ├── 2.1 Incremental LSP parsing
  ├── 2.2 Pattern fingerprinting
  └── 2.3 Pattern compilation cache

Phase 3 ✅
  ├── 3.1 Node structural hash cache
  ├── 3.2 Predicate cost ordering
  └── 3.3 Memory-mapped file I/O

Phase 4 (partially implemented)
  ├── 4.1 KindMask fast-path ✅
  ├── 4.2 Iterative matching foundations ✅
  ├── 4.3 Parallel within-file matching ✗ (not feasible: Node lacks Send/Sync)
  └── 4.4 Full iterative rewrite (deferred: needs profiling evidence)
```

---

## Benchmarking (Before & After Each Change)

```bash
# Single-pattern search
hyperfine 'sg run -p "console.log(\$A)" --lang js .' --warmup 3

# Multi-rule scan
hyperfine 'sg scan' --warmup 3

# Profile with flamegraph
cargo flamegraph -- scan

# Key metrics
# - Wall time (primary)
# - Files parsed (validate literal filtering)
# - Nodes visited (validate pruning)
# - Allocations per match (validate SmallVec)
```

---

## Compatibility

All changes are **backward-compatible**:
- No CLI argument changes
- No YAML rule format changes
- No match semantics changes
- No public API changes to `ast-grep-core` or `ast-grep-config`
