# Rendering, CSS, and interaction

Know which pipeline stage you are fixing before you change anything: **style recalc → layout →
layerize → paint → composite**. The DevTools Performance panel names them; a fix aimed at the wrong
stage is noise. [Guide to the Chrome performance tab](https://blog.jiayihu.net/comprenhensive-guide-chrome-performance/).

## Layout and paint

- **Separate DOM reads from writes.** A read after a write forces synchronous layout. Batch reads in
  one `requestAnimationFrame` and writes in the next.
  [High-performance input handling](https://nolanlawson.com/2019/08/11/high-performance-input-handling-on-the-web/),
  [Wikipedia page previews](https://techblog.wikimedia.org/2020/11/23/web-performance-case-study-wikipedia-page-previews/).
- **`content-visibility: auto`** with `contain-intrinsic-size` skips rendering offscreen subtrees —
  one of the largest cheap wins available. [emoji-picker-element#445](https://github.com/nolanlawson/emoji-picker-element/pull/445),
  [write-up](https://nolanlawson.com/2024/09/18/improving-rendering-performance-with-css-content-visibility/).
- **Write CSS variables at the narrowest scope that needs them.** Setting a custom property on an
  ancestor invalidates the whole subtree's style recalc.
  [mui-x#12019](https://github.com/mui/mui-x/pull/12019).
- **Watch layer count.** Stray `z-index`, `will-change`, and transforms create composited layers that
  cost GPU memory and lengthen "layerize".
  [nuka-carousel#796](https://github.com/FormidableLabs/nuka-carousel/pull/796),
  [mui-x#11924](https://github.com/mui/mui-x/pull/11924).
- **Uniform `border-radius`.** Mismatched radii on large elements can cost tens of MB of GPU memory —
  [report](https://x.com/penzington/status/2098513892460708142).
- **Don't attach tooltips and popovers to `document.body`** when a closer containing block will do —
  [write-up](https://atfzl.com/articles/don-t-attach-tooltips-to-document-body/).
- **Cap DOM size.** Virtualize long lists; the state of the art is in
  [On rendering diffs](https://pierre.computer/writing/on-rendering-diffs) and
  [Sentry's code renderer](https://sentry.engineering/blog/better-code-rendering-through-virtualization)
  (virtualize but keep a `textarea` so Ctrl+F still works; disable pointer events while scrolling).
  [Lighthouse guidance](https://developer.chrome.com/docs/lighthouse/performance/dom-size/).
- **Swap thousands of elements for one canvas** when the content is pixels, not semantics —
  [athena-crisis#32](https://github.com/nkzw-tech/athena-crisis/pull/32).
- **Event delegation over per-node listeners** —
  [ariakit#4860](https://github.com/ariakit/ariakit/pull/4860),
  [Confluence Whiteboards](https://www.atlassian.com/engineering/rendering-like-butter-a-confluence-whiteboards-story).

## Selectors and style recalc

- Selector cost is mostly a myth; the real costs are sheet size and invalidation scope.
  [The truth about CSS selector performance](https://blogs.windows.com/msedgedev/2023/01/17/the-truth-about-css-selector-performance/).
- Frequent runtime `<style>` insertion is a measurable cost in concurrent-rendering apps —
  [Style performance and concurrent rendering](https://nolanlawson.com/2022/10/22/style-performance-and-concurrent-rendering/).
- Shadow DOM scoping is slightly faster than class-prefix scoping —
  [comparison](https://nolanlawson.com/2022/06/22/style-scoping-versus-shadow-dom-which-is-fastest/),
  [talk summary](https://nolanlawson.com/2023/01/17/my-talk-on-css-runtime-performance/).
- Registering `@property` has a one-time cost, then outperforms unregistered custom properties —
  [benchmark](https://web.dev/blog/at-property-performance).

## Animations

- **Animate `transform`, `opacity`, and `filter` only.** For motion that changes layout, use
  [FLIP](https://aerotwist.com/blog/flip-your-animations/). General guidance:
  [motion.dev performance guide](https://motion.dev/guides/performance).
- Animated blur needs a layered/downsampled approach —
  [Chrome](https://developer.chrome.com/blog/animated-blur/).
- Animate a pseudo-element's opacity instead of `box-shadow` —
  [SitePoint](https://www.sitepoint.com/css-box-shadow-animation-performance/).
- Text shimmer: `translate` + `mask`, not `background-position` —
  [devongovett](https://x.com/devongovett/status/2092991811157463500).
- View transitions: precompute the animation delta for `::view-transition-group(*)` —
  [bram.us](https://www.bram.us/2025/02/07/view-transitions-applied-more-performant-view-transition-group-animations/).
- `getComputedStyle().opacity` is a cheaper forced paint than a double `rAF` —
  [write-up](https://webventures.rejh.nl/blog/2022/getcomputedstyle-element-opacity/).
- If a decorative CSS animation still burns CPU, a prerendered APNG can win —
  [opencode#42952](https://github.com/anomalyco/opencode/pull/42952).

## INP and interaction responsiveness

- **Yield to the main thread** between chunks of work (`scheduler.yield()`, or `await` a macrotask).
  Split long tasks; run background work at low priority.
  [Optimizing INP deep dive](https://www.youtube.com/watch?v=cmtfM4emG5k).
- **Let the browser paint before the expensive write.** Deferring an inline style on `<body>` by one
  frame, or moving `autofocus` into the next task, fixes INP without changing behaviour.
  [radix-ui#2855](https://github.com/radix-ui/primitives/pull/2855),
  [sentry#64165](https://github.com/getsentry/sentry/pull/64165).
- **Lazy de-render.** Set `display: none` on the interaction, remove the nodes later in
  `requestIdleCallback` — [PubTech cut INP by 64%](https://web.dev/case-studies/pubconsent-inp).
- **Defer offscreen components** to protect early-load INP —
  [HTTPArchive data](https://twitter.com/rick_viscomi/status/1754882706951864731).
- **Prefer the CSS primitive over the JS polyfill.** `text-wrap: balance` replaces a JS text balancer
  outright — [write-up](https://www.erwinhofman.com/blog/use-text-wrap-balance-to-improve-inp/).
- **Throttle store subscriptions.** Excessive `useSyncExternalStore` notifications are an INP source —
  [ariakit#4212](https://github.com/ariakit/ariakit/pull/4212).
- **Avoid frequent `DOMParser` calls** on the interaction path —
  [Uber Eats](https://webperf.tips/tip/uber-eats-dom-parsing/).
- Framework-specific case study: [how Preply improved INP on Next.js](https://medium.com/preply-engineering/how-preply-improved-inp-on-a-next-js-application-without-react-server-components-and-app-router-491713149875).

## Loading and delivery

- **Remove redirects at the edge** — [Redirect liquidation](https://calendar.perfplanet.com/2021/redirect-liquidation/).
- **Pass server state as `JSON.parse` of a non-executing `<script>` payload**, not an inline JS object
  literal — [write-up](https://kurtextrem.de/posts/state-revisited).
- **Keep requests "simple"** so they skip the CORS `OPTIONS` preflight —
  [webperf.tips](https://webperf.tips/tip/optimizing-cors/).
- **Choose an SVG embedding strategy deliberately** at icon-set scale —
  [SVG icon stress test](https://cloudfour.com/thinks/svg-icon-stress-test/).
- Worked audits worth reading end to end: [Notion](https://3perf.com/blog/notion/),
  [Walmart](https://iamakulov.com/notes/walmart/), [Causal](https://3perf.com/blog/causal/).

## Memory in UI code

- Closures captured by long-lived caches are the usual leak. In React, extract to a custom hook so
  the closure's lifetime matches the component's —
  [Sneaky React memory leaks II](https://schiener.io/2024-05-29/react-query-leaks).
