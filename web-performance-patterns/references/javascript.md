# JavaScript hot paths

Ordered by the priority ladder in `SKILL.md`. Each entry links the patch or write-up it came from —
read the original before applying it, because most wins are conditional on the workload.

## 1. Do less work

- **Cache expensive derived results** — regex matches, parses, resolved paths. Bound the cache, or
  you have traded CPU for a leak. [emoji-toolkit#57](https://github.com/joypixels/emoji-toolkit/pull/57),
  [pnpm#6317](https://github.com/pnpm/pnpm/pull/6317),
  [ckeditor5#17296](https://github.com/ckeditor/ckeditor5/pull/17296) and
  [#17586](https://github.com/ckeditor/ckeditor5/pull/17586).
- **Skip work that cannot affect output.** Non-reactive hooks do not need to run during SSR.
  [TanStack Start#6497](https://github.com/TanStack/router/pull/6497).
- **Replace measurement with math.** An early-exit calculation avoids a forced DOM read entirely.
  [recharts#3953](https://github.com/recharts/recharts/pull/3953).
- **Throttle progress and logging.** npm's install progress bar made `npm i` measurably slower.
  [npm#11283](https://github.com/npm/npm/issues/11283#issuecomment-175246823).
- **`os.availableParallelism()`**, not `os.cpus().length` — the latter does real work per call.

## 2. Change the complexity

- Frequency-counter objects turn `O(n²)` array comparison into `O(n)`.
  [write-up](https://dev.to/doabledanny/how-to-compare-arrays-in-javascript-efficiently-1p0).
- Binary search over a sorted index instead of a linear scan.
  [How CKEditor made its editor load faster](https://ckeditor.com/blog/how-we-made-our-rich-text-editor-load-faster-part-2/).
- A large `switch` over opcodes is slower than an indexed dispatch table.
  [10x'ing a TI-84 emulator](https://artemis.sh/2022/08/07/emulating-calculators-fast-in-js.html).
- Hand-written finite state machines beat regexes for simple grammars.
  [FSM parsing](https://hackernoon.com/high-performance-text-parsing-using-finite-state-machines-fsm-6d3m33j9),
  [mrale.ph](https://mrale.ph/blog/2016/11/23/making-less-dart-faster.html).
- Convert recursion to an explicit stack when the depth is data-driven.
  [parcel#9266](https://github.com/parcel-bundler/parcel/pull/9266).

## 3. Allocate less

- **`Promise`s are not free.** Avoid creating them per-node in SSR and template hot paths.
  [vuejs/core#11340](https://github.com/vuejs/core/pull/11340),
  [astro#13195](https://github.com/withastro/astro/pull/13195),
  [streaming templates](https://lorenzofox.dev/posts/html-streaming-part-2/).
  Long promise chains also cannot be collected until they settle —
  [Cribl](https://cribl.io/blog/promise-chaining-memory-leak/).
- **`new URL()` is expensive.** Gate it behind a cheap string check.
  [TanStack Router#6447](https://github.com/TanStack/router/pull/6447). Build query strings with
  `new URLSearchParams()` rather than writing through `new URL().searchParams` —
  [react-router#14084](https://github.com/remix-run/react-router/pull/14084).
- **Hoist regexes, closures, and constants** out of the function that runs per item; an inline arrow
  allocates on every call. [slow-json-stringify#31](https://github.com/lucagez/slow-json-stringify/pull/31),
  [esquery#134](https://github.com/estools/esquery/pull/134),
  [Bluebird](https://www.reaktor.com/articles/javascript-performance-fundamentals-make-bluebird-fast).
- **Avoid spread in hot paths** — [astro#10765](https://github.com/withastro/astro/pull/10765/changes).
  But above roughly 1020 properties in V8, `{...base}` beats an `Object.keys()` loop —
  [immer#1188](https://github.com/immerjs/immer/pull/1188). Measure at *your* `N`.
- **Lazy construction.** `Object.create(Foo.prototype)` plus getters skips initializing fields nobody
  reads — [joist-orm](https://joist-orm.io/blog/lazy-fields/).
- **Batch encoder calls.** One `TextEncoder#encode` on a large string beats `N` small ones —
  [astro#15123](https://github.com/withastro/astro/pull/15123/). Same shape for string building:
  `Buffer#toString` over `str += char` — [nanoid#602](https://github.com/ai/nanoid/pull/602).
- **Cap `Error.stackTraceLimit`** when code constructs many `Error`s — capturing the stack, not the
  `throw`, is the cost. [react#37086](https://github.com/facebook/react/pull/37086) caps it at 10.
- **Bit flags instead of N booleans or string surgery.**
  [node-semver#536](https://github.com/npm/node-semver/pull/536/files),
  [nodejs/node#49745](https://github.com/nodejs/node/pull/49745).
- **`Object.freeze`** lowered memory at roughly equal speed in one case —
  [node-semver#528](https://github.com/npm/node-semver/pull/528). Verify; freezing can also deopt.

## 4. Keep object shapes stable

V8 gives each object a [hidden class](https://v8.dev/docs/hidden-classes). A call site that sees one
shape inlines; a few shapes go polymorphic; many go megamorphic and fall back to dictionary lookup.

- **Initialize every property, in the same order, in one place.** Never add fields conditionally.
  [TypeScript#58045](https://github.com/microsoft/TypeScript/pull/58045/files),
  [#57977](https://github.com/microsoft/TypeScript/pull/57977/files),
  [Monomorphic AST Nodes](https://github.com/microsoft/TypeScript/issues/59198).
- **Never `delete`.** Assign `undefined`, or rebuild the object.
  [ansicolor#20](https://github.com/xpl/ansicolor/pull/20),
  [TanStack Router#6456](https://github.com/TanStack/router/pull/6456).
- **Split grab-bag objects** into a hot object with the common fields plus a side object for the rest
  — [TypeScript#58928](https://github.com/microsoft/TypeScript/pull/58928).
- **Prefer a monomorphic tuple array to a polymorphic object union** —
  [rocicorp/mono#5801](https://github.com/rocicorp/mono/pull/5801).
- **Use the same key names across code paths**, even when they mean the same thing —
  [react#28569](https://github.com/facebook/react/pull/28569/).
- **Field representation matters.** A field initialized to `null` and later assigned a double forces
  a representation change; initialize numeric fields with `0` and doubles with `NaN` —
  [The story of a V8 performance cliff in React](https://v8.dev/blog/react-cliff).
- **Pass positional arguments, not an options object,** in hot internal APIs — the object is both an
  allocation and a polymorphism source. [undici#3302](https://github.com/nodejs/undici/pull/3302).
- Further reading: [Typia's hidden-class deep dive](https://dev.to/samchon/secret-of-typia-how-it-could-be-20000x-faster-validator-hidden-class-optimization-in-v8-engine-1mfb),
  [Maybe you don't need Rust and WASM](https://mrale.ph/blog/2018/02/03/maybe-you-dont-need-rust-to-speed-up-your-js.html).

## 5. Data structures

The corpus does **not** say "`Set` is slow". It says *pick from the measured access pattern* — the
same swap appears in both directions.

| Change | When it wins | Evidence |
|---|---|---|
| `Set` → array | uniqueness already guaranteed; small or short-lived; iteration-dominant | [Chrome Performance panel, 400% faster](https://developer.chrome.com/blog/perf-panel-4x-faster), [valibot#68](https://github.com/fabian-hiller/valibot/pull/68) |
| object → `Set`/`Map` | repeated membership tests over many keys | [pnpm#6749](https://github.com/pnpm/pnpm/pull/6749) |
| `Set` → linked list | ordered traversal with O(1) insert/remove and no hashing | [preact/signals#136](https://github.com/preactjs/signals/pull/136) |
| array → linked list | `shift`/`unshift` dominates | [superlock#7](https://github.com/Kikobeats/superlock/pull/7) |
| `Map` → typed / parallel arrays | large fixed-size datasets; CPU cache locality | [Data-oriented design](https://docs.google.com/presentation/d/1yn87uVuB7oXmRX5uMsdu9_DQP7M5a2jO2xKuRl1ERIo/edit) |
| `Set` → bitset | dense integer ID sets | [rollup#4862](https://github.com/rollup/rollup/pull/4862), [parcel#9266](https://github.com/parcel-bundler/parcel/pull/9266) |
| immutable spread → mutable array | mutation stays inside one scope | [TanStack Table#4495](https://github.com/TanStack/table/pull/4495) |

- **`Map` rehashing** is the hidden cost on large, frequently grown collections — pre-size, or use an
  array keyed by a dense index. [Chrome perf panel write-up](https://developer.chrome.com/blog/perf-panel-4x-faster).
- **Pre-allocate**: `new Array(len)` with index assignment beats `push` when the length is known;
  plain `for..in` beats `Object.entries`.
  [opentelemetry-js#6514](https://github.com/open-telemetry/opentelemetry-js/pull/6514),
  [#6287](https://github.com/open-telemetry/opentelemetry-js/pull/6287).
- **Replace generic iterators with hand-written loops** in the hottest paths —
  [rocicorp/mono#5834](https://github.com/rocicorp/mono/pull/5834).
- **Encode compactly when the bottleneck is bytes**, not CPU: three `Float16`s → 8 base64 chars —
  [tldraw#7364](https://github.com/tldraw/tldraw/pull/7364).

## 6. Strings and regex

- Cache `str.length` in `while` loops and hoist repeatedly compared character codes.
  [jshttp/cookie#144](https://github.com/jshttp/cookie/pull/144),
  [node-jsonc-parser#81](https://github.com/microsoft/node-jsonc-parser/pull/81).
- Avoid negative lookahead/lookbehind — it blocks regex engine optimizations.
  [valibot#180](https://github.com/fabian-hiller/valibot/pull/180#issuecomment-1751250891).
- Do not run a regex twice (`test` then `match`); one `exec` and branch on the result.
  [svgo#1717](https://github.com/svg/svgo/pull/1717); avoid expensive regexes entirely where a
  character scan will do — [postcss-plugins#737](https://github.com/csstools/postcss-plugins/pull/737).
- Replace `split` with `indexOf` + `substring` in parsers.
  [unhead#368](https://github.com/unjs/unhead/pull/368), [esquery#134](https://github.com/estools/esquery/pull/134).
- Prefer `slice` over `indexOf` scans where worst-case runtime matters —
  [fastify#5400](https://github.com/fastify/fastify/pull/5400).
- **Sliced strings leak.** `slice`/`substring`/`match` on a huge string keeps the parent alive in V8
  ([v8:2869](https://bugs.chromium.org/p/v8/issues/detail?id=2869)). Force a flat copy when you retain
  the result. [three.js#9680](https://github.com/mrdoob/three.js/pull/9680),
  [A tale of a JS memory leak](https://www.just-bi.nl/a-tale-of-a-javascript-memory-leak/),
  [V8 string internals](https://iliazeus.github.io/articles/js-string-optimizations-en/).
- Compression is a performance budget too: reordering an alphabet shrank brotli output —
  [nanoid#310](https://github.com/ai/nanoid/pull/310/files).
- `Intl.DateTimeFormat` (constructed once, reused) beats `Date.toLocaleDateString` —
  [webperf.tips](https://webperf.tips/tip/date-formatting/).

## 7. Syntax and transpilation cost

What you write is not what runs. Check the bundler output before blaming the source.

- Down-transpiled `for..of` and destructuring allocate iterators and temporaries — raise the target.
  [graphql-js#3687](https://github.com/graphql/graphql-js/pull/3687),
  [Speeding up the JS ecosystem](https://marvinh.dev/blog/speeding-up-javascript-ecosystem/).
- Class transpilation output differs sharply between targets; measure the emitted form.
  [preact/signals#160](https://github.com/preactjs/signals/pull/160).
- Drop pass-through wrappers: `const foo = (s) => bar(s)` → `const foo = bar`.
  [TypeScript#61822](https://github.com/microsoft/TypeScript/pull/61822/).
- `?.` instead of `|| {}` avoids an allocation per call.
- `Symbol.isConcatSpreadable` deopts `Array#concat` — [Webpack 5, 15x slower](https://www.tines.com/blog/understanding-why-our-build-got-15x-slower-with-webpack-5).
- `var` beat `let`/`const` in one specific `tsc` loop — [TypeScript#52656](https://github.com/microsoft/TypeScript/pull/52656).
  This is the most cargo-culted entry in the corpus. It is a local result. Benchmark before repeating it.

## Further reading

[MythBusters JS](https://mythbusters.js.org/) ·
[v8-perf](https://github.com/thlorenz/v8-perf) ·
[Optimizing JavaScript for fun and for profit](https://romgrk.com/posts/optimizing-javascript) ·
[The fastest JS color library](https://romgrk.com/posts/color-bits/) ·
[nodejs-bench-operations](https://github.com/RafaelGSS/nodejs-bench-operations)
