---
name: web-performance-patterns
description: Use when making JavaScript, TypeScript, CSS, or DOM code faster — profiling a hot path, cutting allocations, picking a data structure, fixing layout thrashing or INP, speeding up `tsc` or a CLI, writing a benchmark, or reviewing a PR that claims a performance win.
version: 1.0.0
license: MIT
metadata:
  hermes:
    tags: [performance, v8, benchmarking, profiling, inp, core-web-vitals, css, typescript]
---

# Web Performance Patterns

A measure-first playbook distilled from ~150 merged performance PRs and write-ups catalogued in
[kurtextrem/awesome-performance-patches](https://github.com/kurtextrem/awesome-performance-patches).
Every pattern below was a real, measured win in a real codebase — which is exactly why none of them
is a rule you may apply unmeasured.

## The iron rule: measure, then patch

**A performance change without a number is a refactor.** Before editing anything:

1. **Profile first.** Find the hot path; do not guess it. Chrome DevTools Performance panel for the
   browser, `--cpu-prof` / `--prof` for Node.
2. **Distrust the profiler's attribution.** Self-time can be misassigned — the widely-cited
   "`setTimeout` is expensive" finding was a DevTools artifact, and the
   [TanStack Query patch built on it was reverted](https://github.com/TanStack/query/pull/9827#issuecomment-3530614157).
   Confirm a hypothesis with a benchmark, not a flame chart alone.
3. **Benchmark with a real harness**, never `console.time` in a `for` loop:
   [bench-node](https://github.com/RafaelGSS/bench-node), [Tachometer](https://nolanlawson.com/2024/08/05/reliable-javascript-benchmarking-with-tachometer/),
   [mitata](https://github.com/evanwashere/mitata), or [Attest](https://github.com/arktypeio/arktype/tree/main/ark/attest#benches) for type-checking cost.
   Watch for dead-code elimination, missing warmup, and an `N` that never reaches the tier you ship in.
4. **Report engine, version, `N`, and delta.** Then state what got worse — readability, memory,
   bundle size. Most of these patterns trade one for the other.
5. **Assume every claim here has an expiry date.** Engines change. `Set` vs. array, `Intl.Collator`,
   `for..of` cost and transpilation trade-offs have all flipped across V8 versions. Re-validate
   against the runtime you actually deploy.

## Priority ladder

Work top-down. Step 5 is where most people start and where the least value is.

1. **Do less work.** Cache, memoize, early-exit, skip work that cannot affect the output (e.g. skip
   non-reactive hooks during SSR). The largest wins in the corpus are all here.
2. **Change the complexity.** `O(n²)` → `O(n)` with a frequency map; linear scan → binary search;
   array `shift` → linked list; regex → finite state machine.
3. **Allocate less.** Objects, closures, `Promise`s, `new URL()`, streams, and intermediate arrays in
   a hot loop dominate everything below this line.
4. **Keep object shapes stable.** Monomorphic call sites, no `delete`, consistent property
   initialization order. This is the single most repeated win across the whole corpus.
5. **Micro-tune.** Hoisted regexes, cached `.length`, bit flags, syntax choices. Real, but small, and
   easy to get backwards — benchmark each one.

## Reference

| File | Covers |
|---|---|
| [references/javascript.md](references/javascript.md) | Hidden classes and monomorphism, allocation, data structures, strings and regex, transpilation cost, caching |
| [references/rendering.md](references/rendering.md) | Layout/paint/composite, selectors and style recalc, animations, INP, loading |
| [references/tooling.md](references/tooling.md) | TypeScript type-check performance, Node/CLI/build, test-suite speed, benchmark harnesses |

## Reviewing a performance PR

- Is there a benchmark, with a harness named and an `N` stated? Is the harness measuring the thing
  the PR changed?
- Does the profile confirm this path is hot in a realistic workload, or only in the microbenchmark?
- Is the win engine-specific? Say so in the PR, so the next person knows when to re-check it.
- Did readability or memory regress? A `Uint32Array` bitset is fine in a graph traversal, hostile in
  application code.
- Is there a correctness risk? Caching without invalidation, mutation escaping its scope, and
  hand-rolled parsers replacing regexes are the three that bite.
- Could a cheaper step on the ladder have produced the same win?
