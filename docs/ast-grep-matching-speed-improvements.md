# ast-grep Matching Engine: Speed Improvement Architecture

> A comprehensive analysis of the current matching pipeline and concrete proposals to improve matching speed in ast-grep's core engine.

---

## 1. Current Architecture Summary

### 1.1 Matching Pipeline

ast-grep's matching pipeline flows through these stages:

```
Source File → tree-sitter parse → Root<D> → DFS traversal → kind filter → pattern match → constraint check → transform → result
```

**Key components:**

| Component | Location | Role |
|-----------|----------|------|
| `Pattern` | `crates/core/src/matcher/pattern.rs` | Compiles pattern strings into `PatternNode` trees |
| `match_node_impl()` | `crates/core/src/match_tree/match_node.rs` | Recursive tree-to-tree structural matching |
| `MetaVarEnv` | `crates/core/src/meta_var.rs` | Stores captured meta-variables during matching |
| `CombinedScan` | `crates/config/src/combined.rs` | Maps AST node kinds to rules for multi-rule scans |
| `find_all()` | `crates/core/src/node.rs:323-336` | DFS traversal with `potential_kinds()` pre-filter |
| `RuleCore::do_match()` | `crates/config/src/rule_core.rs:218-244` | Full matching pipeline: kind → pattern → constraints → transforms |
| Worker/WalkParallel | `crates/cli/src/utils/worker.rs` | Parallel file traversal using `ignore` crate |

### 1.2 Existing Optimizations

1. **`potential_kinds()` BitSet filter**: Before expensive structural matching, `find_all()` and `CombinedScan` skip nodes whose `kind_id` isn't in the matcher's `potential_kinds()` BitSet.
2. **`Cow<MetaVarEnv>` copy-on-write**: The environment is passed as `Cow`, avoiding cloning when no meta-variables are captured.
3. **`kind_rule_mapping` in CombinedScan**: A `Vec<Vec<usize>>` maps each node kind to the list of applicable rules, enabling O(1) lookup per node during multi-rule scans.
4. **Parallel file walking**: The CLI uses `ignore::WalkParallel` with MPSC channels for concurrent file processing.
5. **LTO in release builds**: `profile.release.lto = true` enables link-time optimization.
6. **Zero-alloc pattern preprocessing**: `Cow<str>` avoids allocation for stub languages where no `$`-replacement is needed.

### 1.3 Current Bottlenecks Identified

| Bottleneck | Where | Impact |
|------------|-------|--------|
| **Full DFS for every scan** | `node.dfs()` in `find_all()` and `CombinedScan::scan()` | Every node is visited even if entire subtrees cannot match |
| **Per-node `Cow::Owned(MetaVarEnv::new())` allocation** | `MatcherExt::match_node()` | Allocates a fresh HashMap for every candidate node |
| **Recursive tree matching** | `match_nodes_impl_recursive()` | Deep recursion on large ASTs; stack overhead |
| **Re-parsing patterns from `&str` matcher** | `impl Matcher for str` | Creates a new `Pattern` on every call |
| **Single DFS per `CombinedScan::scan()`** | `combined.rs:252` | All rules are checked in the same DFS pass — efficient, but no subtree pruning |
| **`does_node_match_exactly()` full-tree comparison** | `match_tree/mod.rs:123-144` | Recursive structural equality check without short-circuit hashing |
| **HashMap-based `MetaVarEnv`** | `meta_var.rs:17-21` | HashMap overhead for typically 1-5 variables |

---

## 2. Proposed Speed Improvements

### 2.1 Subtree Pruning in DFS Traversal

**Problem:** Currently, `find_all()` and `CombinedScan::scan()` visit every node in the AST via `dfs()`, even when entire subtrees provably cannot contain matches.

**Proposal:** Implement a *pruning DFS iterator* that skips subtrees when no node kind within the subtree could match any rule.

**Mechanism:**
1. During pattern compilation, compute a `BitSet` of all *ancestor kinds* that could transitively contain a matching node (call this `potential_ancestor_kinds`).
2. In `CombinedScan::new()`, precompute the union of all rules' potential ancestor kinds.
3. During DFS traversal, when visiting a node whose kind is NOT in `potential_ancestor_kinds` AND NOT in `potential_kinds`, skip its entire subtree.

```rust
// Proposed: pruning DFS in CombinedScan::scan()
fn scan_with_pruning<D: Doc>(&self, root: &AstGrep<D>) {
    let mut stack = vec![root.root()];
    while let Some(node) = stack.pop() {
        let kind = node.kind_id() as usize;

        // Check if this node itself matches any rule
        if let Some(rule_idx) = self.kind_rule_mapping.get(kind) {
            for &idx in rule_idx {
                // ... match logic
            }
        }

        // Only descend if this subtree could contain matches
        if self.ancestor_kinds.contains(kind) {
            // Push children in reverse order for correct DFS ordering
            for child in node.children().rev() {
                stack.push(child);
            }
        }
        // else: prune entire subtree
    }
}
```

**Expected impact:** 20-60% reduction in visited nodes for specific patterns (e.g., searching for `return` statements skips nodes that cannot contain `return_statement` as descendants).

---

### 2.2 MetaVarEnv Allocation Pooling with SmallVec

**Problem:** `MatcherExt::match_node()` creates a `Cow::Owned(MetaVarEnv::new())` for every candidate node. `MetaVarEnv` uses three `HashMap`s internally. Most patterns capture 0-5 variables.

**Proposal:** Replace `HashMap<String, Node>` in `MetaVarEnv` with a `SmallVec<[(String, Node); N]>` for the common case, falling back to `HashMap` only when N is exceeded.

```rust
use smallvec::SmallVec;

pub struct MetaVarEnv<'tree, D: Doc> {
    // Inline storage for up to 4 single matches (covers 95%+ of patterns)
    single_matched: SmallVec<[(String, Node<'tree, D>); 4]>,
    multi_matched: SmallVec<[(String, Vec<Node<'tree, D>>); 2]>,
    transformed_var: SmallVec<[(String, Underlying<D>); 2]>,
}
```

**Alternative:** Use an arena allocator or object pool to reuse `MetaVarEnv` instances across match attempts:

```rust
// Pool-based approach
struct EnvPool<D: Doc> {
    pool: Vec<MetaVarEnv<'static, D>>,  // reuse cleared envs
}
impl<D: Doc> EnvPool<D> {
    fn acquire(&mut self) -> MetaVarEnv<D> {
        self.pool.pop().unwrap_or_default()
    }
    fn release(&mut self, mut env: MetaVarEnv<D>) {
        env.clear();
        self.pool.push(env);
    }
}
```

**Expected impact:** 15-30% reduction in allocation overhead during hot matching loops. The SmallVec approach is simpler and avoids lifetime complexity.

---

### 2.3 Pattern Fingerprinting for Fast Rejection

**Problem:** Even after `potential_kinds()` filtering, many nodes pass the kind check but fail the full structural match. The `match_node_impl()` recursive comparison is expensive for nodes that don't match.

**Proposal:** Compute a lightweight *fingerprint* of each pattern during compilation — a fixed-size hash of the pattern's structural skeleton (kind sequence + terminal text). During matching, compute the candidate's fingerprint and reject mismatches before entering recursive tree comparison.

```rust
struct PatternFingerprint {
    // Hash of the top-level children's kind sequence
    children_kind_hash: u64,
    // Count of named children
    named_child_count: u8,
    // Number of meta-variables (affects minimum child count)
    meta_var_count: u8,
}

impl Pattern {
    fn fingerprint(&self) -> PatternFingerprint { /* ... */ }
}

// In match_node_impl, before recursive descent:
fn match_node_impl(...) -> MatchOneNode {
    // Fast rejection: check fingerprint before recursive matching
    if !fingerprint_compatible(&goal.fingerprint, candidate) {
        return MatchOneNode::NoMatch;
    }
    // ... proceed with full recursive match
}
```

**Expected impact:** 10-25% reduction in time spent on non-matching nodes, especially for complex patterns with many children.

---

### 2.4 Iterative Tree Matching (Eliminate Recursion)

**Problem:** `match_nodes_impl_recursive()` uses recursive function calls to compare pattern trees against candidate trees. For deeply nested ASTs (e.g., deeply nested JSX, chained method calls), this creates stack overhead and prevents tail-call optimization.

**Proposal:** Convert the recursive tree matcher to an explicit-stack iterative algorithm.

```rust
struct MatchFrame<'p, 't, D: Doc> {
    goal_children: &'p [PatternNode],
    cand_children: Vec<Node<'t, D>>,
    goal_idx: usize,
    cand_idx: usize,
}

fn match_node_iterative<'tree, D: Doc>(
    goal: &PatternNode,
    candidate: &Node<'tree, D>,
    env: &mut Cow<MetaVarEnv<'tree, D>>,
    strictness: &MatchStrictness,
) -> Option<Node<'tree, D>> {
    let mut stack: SmallVec<[MatchFrame; 16]> = SmallVec::new();
    stack.push(MatchFrame::new(goal, candidate));

    while let Some(frame) = stack.last_mut() {
        match frame.step(env, strictness) {
            StepResult::Matched => { stack.pop(); }
            StepResult::Descend(child_goal, child_cand) => {
                stack.push(MatchFrame::new(child_goal, child_cand));
            }
            StepResult::NoMatch => return None,
        }
    }
    Some(candidate.clone())
}
```

**Expected impact:** 5-15% improvement for deeply nested structures; eliminates stack overflow risk for pathological inputs.

**Note:** The existing `match_node_non_recursive` function name is misleading — it's non-recursive at the top level but calls `match_node_impl` which IS recursive. This proposal makes the actual tree comparison iterative.

---

### 2.5 Node Text Hash Cache for `does_node_match_exactly()`

**Problem:** `does_node_match_exactly()` in `match_tree/mod.rs:123-144` performs full recursive structural comparison. This is called during meta-variable consistency checks (when `$A` appears multiple times in a pattern). For large captured subtrees, this is expensive.

**Proposal:** Cache a structural hash on AST nodes (lazily computed, stored in a concurrent hash map keyed by `node_id`). Use hash comparison as a fast-path before recursive structural comparison.

```rust
use dashmap::DashMap;
use std::hash::{Hash, Hasher};
use std::sync::LazyLock;

static NODE_HASH_CACHE: LazyLock<DashMap<usize, u64>> = LazyLock::new(DashMap::new);

fn structural_hash<D: Doc>(node: &Node<D>) -> u64 {
    if let Some(cached) = NODE_HASH_CACHE.get(&node.node_id()) {
        return *cached;
    }
    let mut hasher = FxHasher::default();
    node.kind_id().hash(&mut hasher);
    if node.is_named_leaf() {
        node.text().hash(&mut hasher);
    } else {
        for child in node.children() {
            structural_hash(&child).hash(&mut hasher);
        }
    }
    let hash = hasher.finish();
    NODE_HASH_CACHE.insert(node.node_id(), hash);
    hash
}

pub fn does_node_match_exactly<D: Doc>(goal: &Node<D>, candidate: &Node<D>) -> bool {
    if goal.node_id() == candidate.node_id() {
        return true;
    }
    // Fast-path: hash mismatch means definitely not equal
    if structural_hash(goal) != structural_hash(candidate) {
        return false;
    }
    // Slow-path: full structural comparison (hash collision possible)
    does_node_match_exactly_inner(goal, candidate)
}
```

**Expected impact:** 30-50% speedup for patterns with repeated meta-variables (e.g., `$A($A)`, `if ($X) { $X }`).

---

### 2.6 Rule Ordering by Selectivity in CombinedScan

**Problem:** In `CombinedScan::scan()`, when multiple rules map to the same node kind, they are tried in a fixed order (sorted by fixable + id). Rules with low selectivity (match many nodes) waste time before rules with high selectivity (match few nodes).

**Proposal:** Reorder rules within each kind bucket by estimated selectivity — computed from the pattern's structural depth and number of fixed terminal nodes.

```rust
impl<'r, L: Language> CombinedScan<'r, L> {
    pub fn new(mut rules: Vec<&'r RuleConfig<L>>) -> Self {
        // ... existing kind_rule_mapping construction ...

        // Sort each kind's rule list by selectivity (most selective first)
        for bucket in &mut mapping {
            bucket.sort_by_cached_key(|&idx| {
                let rule = &rules[idx];
                // Higher selectivity score = tried first
                // Deep patterns with many terminals are more selective
                std::cmp::Reverse(rule.matcher.selectivity_score())
            });
        }
        // ...
    }
}

// Add to Matcher trait (with default impl)
trait Matcher {
    fn selectivity_score(&self) -> u32 { 0 }
}
```

**Expected impact:** 5-15% improvement for multi-rule scans where rules share the same kind bucket. Most selective rules reject quickly, reducing wasted work.

---

### 2.7 Batch Suppression Pre-computation

**Problem:** In `CombinedScan::scan()`, suppression checking (`file_sup.suppressed_id()` and `line_sup.suppressed_id()`) happens for every match. The `Suppressions::collect_all()` does a full DFS just to find comment nodes.

**Proposal:** Integrate suppression collection into the main scan DFS pass instead of doing a separate pre-pass.

```rust
fn scan<D: Doc>(&self, root: &AstGrep<D>, separate_fix: bool) -> ScanResult<D, L> {
    let mut suppression_lines: HashMap<usize, Suppression> = HashMap::new();
    let mut file_suppression: Option<Suppression> = None;

    for node in root.root().dfs() {
        // Inline suppression detection (currently a separate pass)
        if node.kind().contains("comment") && node.text().contains(IGNORE_TEXT) {
            // ... register suppression inline ...
        }

        let kind = node.kind_id() as usize;
        let Some(rule_idx) = self.kind_rule_mapping.get(kind) else {
            continue;
        };
        // ... match against rules with inline suppression check ...
    }
}
```

**Expected impact:** ~5-10% improvement by eliminating the separate suppression DFS pass. Particularly impactful for large files with many comments.

---

### 2.8 Parallel Rule Matching Within a File

**Problem:** Currently, within a single file, rules are evaluated sequentially in the DFS loop. For files with thousands of nodes and dozens of rules, this is a serial bottleneck.

**Proposal:** For large files with many rules, partition rules into groups and evaluate them in parallel using `rayon::scope`.

```rust
use rayon::prelude::*;

fn scan_parallel_rules<D: Doc + Sync>(
    &self,
    root: &AstGrep<D>,
) -> Vec<(usize, NodeMatch<D>)> {
    let nodes: Vec<_> = root.root().dfs().collect();

    // Only parallelize if both node count and rule count exceed thresholds
    if nodes.len() < 5000 || self.rules.len() < 10 {
        return self.scan_sequential(root);
    }

    nodes.par_iter()
        .filter_map(|node| {
            let kind = node.kind_id() as usize;
            let rule_idx = self.kind_rule_mapping.get(kind)?;
            let mut results = vec![];
            for &idx in rule_idx {
                if let Some(ret) = self.rules[idx].matcher.match_node(node.clone()) {
                    results.push((idx, ret));
                }
            }
            Some(results)
        })
        .flatten()
        .collect()
}
```

**Expected impact:** 2-4x speedup on large files with many rules; negligible overhead for small files due to threshold gating.

**Caveat:** `Node`'s lifetime ties to `Root`, so nodes must be collected first. The `pinned` module's `DetachNode` mechanism already supports cross-thread node transfer.

---

### 2.9 Lazy Pattern Compilation with Caching

**Problem:** The `impl Matcher for str` creates a new `Pattern` on every call to `match_node_with_env()` (line 79 of `matcher.rs`). This is used by convenience APIs like `node.find("let $A = $B")`.

**Proposal:** Introduce a thread-local LRU cache for recently compiled patterns.

```rust
use std::cell::RefCell;
use lru::LruCache;
use std::num::NonZero;

thread_local! {
    static PATTERN_CACHE: RefCell<LruCache<(String, u32), Pattern>> =
        RefCell::new(LruCache::new(NonZero::new(64).unwrap()));
}

impl Matcher for str {
    fn match_node_with_env<'tree, D: Doc>(
        &self,
        node: Node<'tree, D>,
        env: &mut Cow<MetaVarEnv<'tree, D>>,
    ) -> Option<Node<'tree, D>> {
        let lang_id = node.lang().lang_id();
        PATTERN_CACHE.with(|cache| {
            let mut cache = cache.borrow_mut();
            let key = (self.to_string(), lang_id);
            let pattern = cache.get_or_insert(key, || {
                Pattern::new(self, node.lang().clone())
            });
            pattern.match_node_with_env(node, env)
        })
    }
}
```

**Expected impact:** Significant for NAPI/PyO3 users who call `find("pattern")` in loops. Minimal impact for CLI (patterns are pre-compiled).

---

### 2.10 SIMD-Accelerated Kind Filtering

**Problem:** `BitSet::contains()` checks during `find_all()` are called for every node in the DFS. While individually cheap, at millions of nodes per scan, this adds up.

**Proposal:** Replace `bit-set` crate's `BitSet` with a SIMD-friendly fixed-size bitset when the language's kind count fits in 512 bits (which it does for all tree-sitter grammars — typically 200-500 kinds).

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

#[repr(align(64))]
struct SimdBitSet {
    words: [u64; 8],  // 512 bits, covers all tree-sitter grammars
}

impl SimdBitSet {
    #[inline(always)]
    fn contains(&self, bit: usize) -> bool {
        let word = bit / 64;
        let bit_in_word = bit % 64;
        self.words[word] & (1u64 << bit_in_word) != 0
    }

    // SIMD intersection for combining multiple rules' kind sets
    #[cfg(target_arch = "x86_64")]
    fn intersects(&self, other: &Self) -> bool {
        unsafe {
            let a = _mm256_load_si256(self.words.as_ptr() as *const __m256i);
            let b = _mm256_load_si256(other.words.as_ptr() as *const __m256i);
            let result = _mm256_and_si256(a, b);
            !_mm256_testz_si256(result, result) != 0
        }
    }
}
```

**Expected impact:** 5-10% improvement in the kind-filtering hot path; larger benefit when intersecting kind sets for combined scans.

---

## 3. Improvement Priority Matrix

| # | Improvement | Complexity | Impact | Risk | Priority |
|---|-------------|-----------|--------|------|----------|
| 2.1 | Subtree Pruning DFS | Medium | High (20-60%) | Low | **P0** |
| 2.2 | SmallVec MetaVarEnv | Low | Medium (15-30%) | Low | **P0** |
| 2.5 | Node Hash Cache | Medium | High (30-50%) | Low | **P1** |
| 2.3 | Pattern Fingerprinting | Medium | Medium (10-25%) | Medium | **P1** |
| 2.7 | Inline Suppression Collection | Low | Low-Medium (5-10%) | Low | **P1** |
| 2.9 | Pattern Compilation Cache | Low | Variable | Low | **P1** |
| 2.4 | Iterative Tree Matching | High | Low-Medium (5-15%) | Medium | **P2** |
| 2.6 | Rule Selectivity Ordering | Low | Low-Medium (5-15%) | Low | **P2** |
| 2.8 | Parallel Rule Matching | High | High (2-4x) | High | **P2** |
| 2.10 | SIMD Kind Filtering | Medium | Low (5-10%) | Medium | **P3** |

---

## 4. Measurement Strategy

Before implementing any optimization, establish baselines:

```bash
# Benchmark single-pattern matching
hyperfine 'sg run -p "console.log(\$A)" --lang js .' --warmup 3

# Benchmark multi-rule scanning
hyperfine 'sg scan' --warmup 3

# Profile with perf
perf record -g sg scan
perf report

# Flame graph
cargo flamegraph -- scan
```

**Key metrics to track:**
- Nodes visited per file (to validate pruning)
- Allocations per match attempt (to validate SmallVec/pool)
- Time spent in `match_node_impl` vs. total scan time
- Rule-level match/reject ratio (to validate selectivity ordering)

---

## 5. Compatibility Considerations

All proposed changes maintain:
- **API compatibility**: The `Matcher` trait interface is unchanged (new methods have defaults).
- **Behavioral correctness**: Optimizations are purely performance — no change in match semantics.
- **Cross-platform support**: SmallVec, hash caching, and iterative matching work everywhere. SIMD proposal uses `cfg` gates.
- **Thread safety**: All caching proposals use thread-local or concurrent data structures (`DashMap`, `thread_local!`).

---

## 6. Architecture Diagram: Optimized Matching Pipeline

```
Source File
    │
    ▼
tree-sitter parse → Root<D>
    │
    ▼
Pruning DFS Iterator (§2.1)
    │ skip subtrees where ancestor_kinds ∉ any rule
    ▼
Kind Filter (§2.10 SIMD BitSet)
    │ kind_id ∈ potential_kinds?
    ▼
Fingerprint Fast-Reject (§2.3)
    │ children_kind_hash match?
    ▼
Iterative Tree Match (§2.4)
    │ PatternNode vs Node comparison
    │ MetaVarEnv with SmallVec (§2.2)
    ▼
Meta-Var Consistency (§2.5 Hash Cache)
    │ structural_hash(goal) == structural_hash(candidate)?
    ▼
Constraint Check → Transform → Result
```
