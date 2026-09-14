# TypeScript, tooling, and benchmarking

## TypeScript type-checking performance

`tsc` slowness is its own discipline — it has almost nothing to do with runtime performance.

- **Measure first**: `tsc --extendedDiagnostics` for totals, `--generateTrace` for a flame chart, and
  [Attest](https://github.com/arktypeio/arktype/tree/main/ark/attest#benches) to pin per-type cost in
  tests. Start from the [TypeScript performance wiki](https://github.com/microsoft/TypeScript/wiki/Performance).
- **Prefer `interface` to large unions and intersection aliases.** Interfaces are cached and compared
  by identity; conditional-intersection aliases are re-expanded.
  [sentry#30847](https://github.com/getsentry/sentry/pull/30847),
  [TanStack Table v9](https://tanstack.com/blog/tanstack-table-v9-typescript-performance).
- **Do not defeat lazy type evaluation.** A mapped type or explicit annotation that forces eager
  expansion can dominate check time — [tRPC](https://twitter.com/s4chinraja/status/1570658634039984128).
- **Annotate invariant type parameters `in out`** to skip variance probing —
  [TanStack Table v9](https://tanstack.com/blog/tanstack-table-v9-typescript-performance).
- Broader treatment: [An approach to optimizing TypeScript type checking performance](https://www.edgedb.com/blog/an-approach-to-optimizing-typescript-type-checking-performance),
  [TanStack Router#1453](https://github.com/TanStack/router/pull/1453).

## Node, CLIs, and build tooling

- **Lazy-load modules in short-lived processes.** Module resolution and evaluation dominate CLI
  startup — [npm scripts](https://marvinh.dev/blog/speeding-up-javascript-ecosystem-part-4/),
  [reducing `npm run` overhead](https://viniciusl.com.br/posts/2024/06/29-reducing-overhead-of-npm-run/).
- **Cache repeated dynamic `import()`** — [vite#12721](https://github.com/vitejs/vite/pull/12721),
  [node#52369](https://github.com/nodejs/node/issues/52369#issuecomment-2071643229).
- **Spawning processes is expensive** and the cost differs sharply across Node, Deno, and Bun —
  [val.town](https://blog.val.town/blog/node-spawn-performance/).
- **Stream splitting** is a common hidden hot spot in protocol code —
  [vscode-js-debug#2002](https://github.com/microsoft/vscode-js-debug/pull/2002/).
- **Reach for native last.** A Rust/napi rewrite is a real option once JS-level work is exhausted —
  [My Node.js is a bit Rusty](https://gal.hagever.com/posts/my-node-js-is-a-bit-rusty). Most entries in
  this corpus got their win without leaving JavaScript.
- Editor extensions have their own startup budget —
  [Speeding up VSCode extensions](https://jason-williams.co.uk/posts/speeding-up-vscode-extensions-in-2022/).

## Test suite speed

- [Making React Testing Library tests 43% faster](https://sigh.dev/posts/making-react-testing-library-faster/).
- Playwright: [general speedups](https://argos-ci.com/blog/speed-up-playwright) and
  [request interception](https://www.checklyhq.com/blog/speed-up-playwright-scripts-request-interception/)
  to cancel requests irrelevant to the test.

## Benchmark harnesses

| Tool | Use for |
|---|---|
| [bench-node](https://github.com/RafaelGSS/bench-node) | Node microbenchmarks with proper warmup and statistics |
| [Tachometer](https://nolanlawson.com/2024/08/05/reliable-javascript-benchmarking-with-tachometer/) | Browser benchmarks with confidence intervals across builds |
| [mitata](https://github.com/evanwashere/mitata) | Quick cross-runtime microbenchmarks |
| [Attest](https://github.com/arktypeio/arktype/tree/main/ark/attest#benches) | TypeScript type instantiation counts |
| [nodejs-bench-operations](https://github.com/RafaelGSS/nodejs-bench-operations) | Prior art on what is fast in current Node |

Pitfalls that invalidate a microbenchmark:

- **Dead-code elimination.** If nothing consumes the result, the engine may delete the work. Return
  or accumulate it.
- **No warmup.** V8 runs interpreted, then optimizes. Measuring tier-up noise measures nothing.
- **Unrealistic `N`.** Several entries in this corpus reverse direction with input size — the
  `Object.keys()` vs. spread crossover sits near 1020 properties.
- **Wrong engine.** JSC, SpiderMonkey, Hermes, and V8 disagree. Benchmark where you ship.
- **Profiler artifacts.** Confirm with a benchmark before patching what a flame chart implied;
  [this one cost a merged-then-reverted PR](https://github.com/TanStack/query/pull/9827#issuecomment-3530614157).
