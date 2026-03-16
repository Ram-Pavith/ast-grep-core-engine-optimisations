# Matching Speed Improvements — Phased Implementation Plan

> Implementation plan for the 10 optimisations proposed in `ast-grep-matching-speed-improvements.md`, validated against the current codebase state.

---

## Implementation Status Summary

| § | Optimisation | Status |
|---|---|---|
| 2.1 | Subtree Pruning DFS | ✅ Implemented (`combined.rs` — `collect_with_ancestors()`, `KindMask`-based `ancestor_kinds`) |
| 2.2 | SmallVec MetaVarEnv | ✅ Implemented (`meta_var.rs` — `SmallVec<[…; 4]>`, `SmallVec<[…; 2]>`) |
| 2.3 | Pattern Fingerprinting | ✅ Implemented (`pattern.rs` — `PatternFingerprint` struct with `is_compatible()`) |
| 2.5 | Node Hash Cache | ✅ Implemented (`match_tree/mod.rs` — inline `structural_hash()` bloom-filter fast-path) |
| 2.6 | Rule Selectivity Ordering | ✅ Implemented (`combined.rs` — `match_cost_hint()` sorting per kind bucket) |
| 2.7 | Inline Suppression Collection | ✅ Implemented (`combined.rs` — `collect_with_ancestors()` merges suppression + ancestor DFS) |
| 2.4 | Iterative Tree Matching | 🔶 Partial (SmallVec children pre-collection done; full iterative rewrite deferred) |
| 2.8 | Parallel Rule Matching | ✗ Not feasible (`tree_sitter::Node` lacks `Send`/`Sync`) |
| 2.9 | Pattern Compilation Cache | ❌ Not started |
| 2.10 | SIMD Kind Filtering | 🔶 Partial (`KindMask` with inline `[u64; 8]` done; explicit SIMD intrinsics not added) |

---

## Phase 1 — Remaining Quick Wins (1–2 weeks)

### 1.1 Pattern Compilation Cache for `impl Matcher for str`

**Source:** §2.9 · **Impact:** High for NAPI/PyO3 users · **Effort:** Small · **Risk:** Low

The `impl Matcher for str` in `crates/core/src/matcher.rs` creates a new `Pattern` on every call. Add a thread-local LRU cache to avoid re-parsing identical pattern strings.

**Files to modify:**

| File | Change |
|---|---|
| `crates/core/Cargo.toml` | Add `lru = "0.12"` dependency |
| `crates/core/src/matcher.rs` | Add `thread_local!` LRU cache keyed by `(pattern_str: String, lang_id: u32)` in `impl Matcher for str`; look up before calling `Pattern::new()` |

**Implementation steps:**
1. Add `lru` to `crates/core/Cargo.toml`
2. In `matcher.rs`, add a `thread_local!` static:
   ```rust
   thread_local! {
       static PATTERN_CACHE: RefCell<LruCache<(String, u32), Pattern>> =
           RefCell::new(LruCache::new(NonZero::new(64).unwrap()));
   }
   ```
3. In `impl Matcher for str::match_node_with_env()`, wrap the existing `Pattern::new()` call with a cache lookup
4. Ensure cache entries are evicted correctly (LRU handles this)

**Verification:**
- All existing tests pass (no semantic change)
- Write a micro-benchmark calling `node.find("pattern")` in a loop — expect near-zero parse time on 2nd+ calls
- Verify with `cargo test -p ast-grep-core`

---

### 1.2 SIMD-Accelerated Kind Set Intersection

**Source:** §2.10 · **Impact:** Low–Medium (5–10%) · **Effort:** Small · **Risk:** Low

`KindMask::Inline([u64; 8])` is already in place. Add SIMD intrinsics for `intersects()` and `is_disjoint()` operations used in `CombinedScan` when combining multiple rules' kind sets.

**Files to modify:**

| File | Change |
|---|---|
| `crates/core/src/kind_mask.rs` | Add `#[cfg(target_arch = …)]` SIMD paths for `intersects()`, `union_with()`, and `is_disjoint()` using `_mm256_*` (x86_64) or NEON (aarch64). Keep scalar fallback for other targets. |

**Implementation steps:**
1. Add `intersects(&self, other: &KindMask) -> bool` with AVX2 path:
   ```rust
   #[cfg(target_arch = "x86_64")]
   fn intersects_simd(a: &[u64; 8], b: &[u64; 8]) -> bool {
       unsafe {
           let va = _mm256_loadu_si256(a.as_ptr() as *const __m256i);
           let vb = _mm256_loadu_si256(b.as_ptr() as *const __m256i);
           let and = _mm256_and_si256(va, vb);
           _mm256_testz_si256(and, and) == 0
       }
   }
   ```
2. Add `#[cfg(target_arch = "aarch64")]` NEON equivalent
3. Scalar fallback: `a.iter().zip(b).any(|(x, y)| x & y != 0)`
4. Add `#[repr(align(32))]` to the `Inline` variant's array for aligned SIMD loads

**Verification:**
- `cargo test -p ast-grep-core` on x86_64 and aarch64
- Benchmark `CombinedScan::scan()` on a project with 50+ rules to measure kind-filtering speedup

---

## Phase 2 — Engine Internals (2–4 weeks)

### 2.1 Full Iterative Tree Matching Rewrite

**Source:** §2.4 · **Impact:** Medium (5–15%) · **Effort:** High · **Risk:** Medium

The recursive `match_node_impl()` / `match_nodes_impl_recursive()` in `match_tree/match_node.rs` should be converted to an explicit-stack iterative algorithm to eliminate stack overhead and stack overflow risk on deeply nested ASTs.

**Files to modify:**

| File | Change |
|---|---|
| `crates/core/src/match_tree/match_node.rs` | Add `MatchFrame` struct; rewrite `match_node_impl()` as iterative loop with `SmallVec<[MatchFrame; 16]>` stack |
| `crates/core/src/match_tree/mod.rs` | Add `Checkpoint` associated type + `checkpoint()`/`rollback()` to `Aggregator` trait (for backtracking during ellipsis matching) |

**Implementation steps:**
1. Define `MatchFrame` holding goal children slice, candidate children vec, and index positions
2. Define `StepResult` enum: `Matched`, `Descend(goal, cand)`, `NoMatch`
3. Implement `MatchFrame::step()` that advances one level of matching
4. Handle ellipsis matching (`…`) via checkpoint/rollback on the aggregator
5. Replace `match_node_impl()` body with iterative loop
6. Keep the old recursive version behind `#[cfg(test)]` for differential testing

**Verification:**
- Differential test: run both recursive and iterative on the full test suite, compare results
- Fuzz test with deeply nested inputs (>500 levels) to verify no stack overflow
- `cargo test -p ast-grep-core`
- Benchmark `match_node_impl` on deeply nested JSX/chained method call ASTs

**Prerequisites:** Phase 1 complete (SmallVec children pre-collection is already done)

---

### 2.2 Structural Hash Cache with `DashMap`

**Source:** §2.5 (enhancement) · **Impact:** Medium · **Effort:** Medium · **Risk:** Low

The current `structural_hash()` is computed inline every time. For repeated meta-variable consistency checks on the same nodes (e.g., `$A($A)` patterns), add a per-file `DashMap<NodeId, u64>` cache.

**Files to modify:**

| File | Change |
|---|---|
| `crates/core/Cargo.toml` | Add `dashmap = "6"` dependency (if not already present) |
| `crates/core/src/match_tree/mod.rs` | Replace inline `structural_hash()` with cached version using thread-local `HashMap<usize, u64>` (cheaper than `DashMap` since matching is single-threaded per file) |

**Implementation steps:**
1. Add a `thread_local!` `RefCell<HashMap<usize, u64>>` for per-file hash caching
2. Wrap `structural_hash()` to check cache first
3. Clear the cache at the start of each file's scan (in `find_all()` or `CombinedScan::scan()`)
4. Alternative: pass the cache as a parameter to avoid global state

**Verification:**
- `cargo test -p ast-grep-core`
- Benchmark patterns with repeated meta-variables (`$A($A)`, `if ($X) { $X }`)

---

## Phase 3 — Advanced Optimisations (4–6 weeks)

### 3.1 Parallel Rule Matching — Alternative Approach

**Source:** §2.8 · **Original status:** Not feasible due to `Node` lacking `Send`/`Sync`

**Alternative approach:** Instead of parallelising within a single DFS pass, partition rules into independent `CombinedScan` groups and run each group's scan in parallel using `rayon`, collecting DFS nodes into detached representations first.

**Files to modify:**

| File | Change |
|---|---|
| `crates/config/src/combined.rs` | Add `scan_partitioned()` that groups rules by kind disjointness, runs groups in parallel via `rayon::scope`, merges results |
| `crates/core/src/pinned.rs` | Verify `DetachNode` supports the required operations for cross-thread use |

**Implementation steps:**
1. Profile to determine if within-file rule matching is actually a bottleneck (file-level parallelism may already saturate cores)
2. If justified: partition rules into groups with disjoint `potential_kinds()` sets
3. Collect all nodes into a `Vec<DetachNode>` (already supported by `pinned` module)
4. `par_iter()` over nodes, matching each against the relevant rule group
5. Gate behind threshold: only parallelise when `nodes.len() > 5000 && rules.len() > 10`

**Verification:**
- `cargo test -p ast-grep-config`
- Benchmark on a large project (VS Code, ~30K files) with 50+ rules
- Verify no regressions on small projects

**Decision gate:** Only proceed if profiling shows >20% of scan time in rule matching (vs. file I/O / parsing). If file-level parallelism already saturates cores, skip this entirely.

---

## Phase Dependency Graph

```
Phase 1 (no dependencies, can run in parallel)
  ├── 1.1 Pattern Compilation Cache
  └── 1.2 SIMD Kind Filtering

Phase 2 (depends on Phase 1 completion for testing baseline)
  ├── 2.1 Full Iterative Tree Matching
  └── 2.2 Structural Hash Cache (independent of 2.1)

Phase 3 (depends on Phase 2 for profiling data)
  └── 3.1 Parallel Rule Matching (conditional on profiling)
```

---

## Benchmarking Protocol (Before & After Each Phase)

Run before starting each phase and after completing each item:

```bash
# 1. Single-pattern search (validates core matching speed)
hyperfine 'sg run -p "console.log(\$A)" --lang js .' --warmup 3

# 2. Multi-rule scan (validates CombinedScan optimisations)
hyperfine 'sg scan' --warmup 3

# 3. Repeated meta-variable pattern (validates hash cache)
hyperfine 'sg run -p "\$A(\$A)" --lang js .' --warmup 3

# 4. Deeply nested pattern (validates iterative matching)
hyperfine 'sg run -p "\$A.\$B.\$C.\$D(\$E)" --lang js .' --warmup 3

# 5. Flamegraph for detailed profiling
cargo flamegraph -- scan

# 6. Allocation profiling (validates SmallVec / cache effectiveness)
# macOS:
cargo instruments -t Allocations -- scan
# Linux:
valgrind --tool=massif target/release/sg scan
```

**Key metrics per phase:**

| Metric | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|
| Pattern recompilation count | ✓ | | |
| Kind-filter time (ns/node) | ✓ | | |
| `match_node_impl` stack depth | | ✓ | |
| Hash cache hit rate | | ✓ | |
| Per-file rule match time | | | ✓ |
| Wall time (primary) | ✓ | ✓ | ✓ |

---

## Compatibility

All changes maintain backward compatibility:
- No CLI argument changes
- No YAML rule format changes  
- No match semantics changes
- No public API changes to `ast-grep-core` or `ast-grep-config`
- Thread-local caches are invisible to callers
- SIMD paths use `#[cfg]` gates with scalar fallbacks
