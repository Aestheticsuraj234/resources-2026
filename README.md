# Advanced Frontend & React — Deep Dive Notes

> 30 topics covering React internals, rendering, performance, architecture, network, and browser fundamentals.

---

## Table of Contents

### React Internals
1. [Diffing Algorithm](#1-diffing-algorithm)
2. [React Fiber Architecture](#2-react-fiber-architecture)
3. [Concurrent Mode](#3-concurrent-mode)
4. [Time Slicing](#4-time-slicing)
5. [Memoization & Render Optimization](#5-memoization--render-optimization)
6. [Server Components (RSC)](#6-server-components-rsc)
7. [Suspense & Error Boundaries](#7-suspense--error-boundaries)

### Rendering
8. [Streaming SSR HTML](#8-streaming-ssr-html)
9. [Hydration](#9-hydration)
10. [ISR – Incremental Static Regeneration](#10-isr--incremental-static-regeneration)
11. [Critical Rendering Path](#11-critical-rendering-path)
12. [Browser Rendering Pipeline](#12-browser-rendering-pipeline)

### Performance
13. [Virtual List & Infinite Scrolling](#13-virtual-list--infinite-scrolling)
14. [Web Vitals & Core Metrics](#14-web-vitals--core-metrics)
15. [Code Splitting & Lazy Loading](#15-code-splitting--lazy-loading)
16. [Image Optimization](#16-image-optimization)
17. [Bundle Optimization & Tree Shaking](#17-bundle-optimization--tree-shaking)
18. [CSS Performance & Containment](#18-css-performance--containment)

### Architecture
19. [Island Architecture](#19-island-architecture)
20. [Micro-frontends](#20-micro-frontends)
21. [State Management Patterns](#21-state-management-patterns)
22. [Edge Computing & Edge Functions](#22-edge-computing--edge-functions)
23. [Design Tokens & CSS Architecture](#23-design-tokens--css-architecture)

### Network
24. [HTTP/2 & HTTP/3 (QUIC)](#24-http2--http3-quic)
25. [Caching Strategies](#25-caching-strategies)
26. [WebSockets & Real-Time Patterns](#26-websockets--real-time-patterns)
27. [GraphQL & Data Fetching Patterns](#27-graphql--data-fetching-patterns)

### Browser
28. [Web Workers & Offthread Work](#28-web-workers--offthread-work)
29. [Web Assembly (WASM)](#29-web-assembly-wasm)
30. [Security: XSS, CSP & CSRF](#30-security-xss-csp--csrf)

---

## React Internals

### 1. Diffing Algorithm

React's reconciliation engine compares the previous and next virtual DOM trees to compute the minimal set of real DOM mutations needed. Instead of using a general O(n³) tree-diff algorithm, React applies two heuristics to achieve O(n) complexity: elements of different types produce different trees, and developers can hint stable identity with keys.

**Key Pointers**

- Two nodes of different types (e.g. `<div>` → `<span>`) cause a full subtree teardown and rebuild — never swap container types unnecessarily.
- Sibling elements without keys force React to diff by position — adding an item at the top causes every sibling to update. Keys short-circuit this.
- Keys must be stable and unique among siblings — using array index as key causes subtle bugs when items are reordered or filtered.
- React bails out of a subtree early when a component returns the same element reference (`memo`, `PureComponent`) — the diff never descends.
- The diffing happens in the render/reconcile phase; DOM mutations are batched and flushed in the commit phase — these are always separate.
- In React 18 concurrent mode, reconciliation can be interrupted mid-tree and restarted — assumptions about synchronous diffing no longer hold.

---

### 2. React Fiber Architecture

Fiber is React's reimplemented reconciler, introduced in React 16. It replaces the old recursive call-stack reconciler with a linked-list of "fiber" nodes — one per component instance. Each fiber is a plain JS object capturing type, props, state, effect list, and pointers to parent/child/sibling fibers. This structure allows React to pause, resume, and prioritize work.

**Key Pointers**

- Each fiber has two copies: `current` (rendered) and `workInProgress` (being computed). Completing a render swaps the two — this is double buffering.
- The fiber tree is traversed depth-first using `child`, `sibling`, and `return` pointers rather than the call stack — work can be paused at any fiber boundary.
- Each fiber carries an `effectTag` (Placement, Update, Deletion) accumulated during render; these are batched into an effect list and applied in commit.
- Priority is encoded as lanes (a bitmask) on each fiber — multiple updates with different priorities can be in flight simultaneously.
- The two phases are "render" (pure, interruptible, may run multiple times) and "commit" (synchronous, imperative DOM mutations, fires effects).
- `useTransition` and `Suspense` are only possible because Fiber can hold a partially-rendered tree in memory without committing it to the DOM.

---

### 3. Concurrent Mode

Concurrent Mode (stable in React 18) allows React to prepare multiple versions of the UI simultaneously without blocking the main thread. It introduces interruptible rendering — React can yield back to the browser between fibers, handle user input mid-render, and discard stale renders when newer updates arrive.

**Key Pointers**

- Concurrent features are opt-in — using `createRoot()` enables them; legacy `render()` keeps old synchronous behavior.
- `startTransition` marks an update as non-urgent; React renders it in the background and only commits when no higher-priority work is pending.
- `useDeferredValue` returns a lagged version of a value — React keeps showing the old value while computing the new one in the background.
- Tearing (showing inconsistent state from an external store mid-render) is solved by `useSyncExternalStore` — required for third-party state libraries in React 18.
- Strict Mode in React 18 deliberately double-invokes render functions in development to surface side effects in the render phase.
- Suspense boundaries become more powerful — they can show fallbacks while async data fetches, not just lazy-loaded components.

---

### 4. Time Slicing

Time slicing is the mechanism inside Concurrent Mode that breaks long rendering work into small chunks and schedules them across multiple frames. React uses the Scheduler package internally, which relies on `MessageChannel` (or `setTimeout`) to yield back to the browser after each ~5ms slice, ensuring frames stay near 60fps even during heavy re-renders.

**Key Pointers**

- The Scheduler uses `postMessage` / `MessageChannel` rather than `requestAnimationFrame` — rAF fires once per frame, but Scheduler may process multiple tasks per frame if time allows.
- React's scheduler assigns five priority levels: Immediate, UserBlocking, Normal, Low, Idle — each has a timeout after which it escalates.
- A render that is time-sliced may run across dozens of MessageChannel turns; refs and state must be consistent across those turns — another reason render must be pure.
- Yielding is heuristic — React checks elapsed time after each fiber unit of work and yields only if over budget (≈5ms by default).
- Time slicing alone doesn't speed up total work; it improves responsiveness by preventing any single frame from being monopolized.
- Long third-party JS still blocks the main thread — time slicing only helps with React's own reconciliation, not synchronous event handlers or external libs.

---

### 5. Memoization & Render Optimization

React re-renders a component whenever its parent re-renders or its state/props change. Memoization with `React.memo`, `useMemo`, and `useCallback` short-circuits unnecessary renders by preserving referential equality across renders.

**Key Pointers**

- `React.memo` does a shallow prop comparison by default — it's useless if you pass new object/array literals or inline functions on every render.
- `useCallback` stabilizes function references between renders — essential when passing callbacks as props to memoized children.
- `useMemo` is for expensive computations, not just object identity — profile first; memoization itself has overhead that can outweigh savings for cheap ops.
- The React Compiler (React Forget, now in beta) automatically infers and inserts memoization — manual memo usage may become obsolete.
- Context causes every consumer to re-render when the value object changes — split contexts by concern, or use `useMemo` on the value object.
- Avoid state colocation anti-pattern: state lifted too high causes wide re-render trees. Keep state as close to where it's used as possible.

---

### 6. Server Components (RSC)

React Server Components run exclusively on the server (or at build time) and never ship their component code to the client. They can directly access databases, file systems, and secrets. Their output is a serialized component tree (RSC payload) streamed to the client — not HTML, not JSON, but a React-specific wire format.

**Key Pointers**

- Server Components have zero client-side JS footprint — the component code itself doesn't ship. Only the rendered output is sent.
- They cannot use `useState`, `useEffect`, or browser APIs — they are fundamentally async functions that return JSX.
- The RSC payload is separate from HTML — it enables React to re-fetch individual Server Components without a full page reload (Next.js router cache).
- Client Components are still needed for interactivity — the split is "where does this component run", not "is this component dynamic".
- Props crossing the server-client boundary must be serializable — functions, class instances, and Symbols cannot cross the boundary.
- Server Components can be async: `async function Page() { const data = await db.query(); return <div>{data}</div>; }` — no `useEffect` needed.

---

### 7. Suspense & Error Boundaries

Suspense lets components declare a loading state by throwing a Promise — React catches it, renders the nearest fallback, and re-renders when the Promise resolves. Error Boundaries catch JS errors anywhere in the child tree, preventing a blank screen.

**Key Pointers**

- Throwing a Promise in render is the mechanism — Suspense-compatible data fetching libs (React Query, Relay, SWR with `suspense: true`) do this automatically.
- Nested Suspense boundaries allow granular loading states — each boundary independently shows/hides its fallback as data arrives.
- Error Boundaries must be class components — there is no hook equivalent. Libraries like `react-error-boundary` wrap this pattern cleanly.
- In React 18, an error thrown during streaming SSR is caught by the nearest server-side error boundary and replaced with an error UI.
- Suspense + Server Components: the server suspends on awaited data, streams the result — the client never sees a loading state for server-fetched data.
- Do not use Suspense for purely client-side effects (loading images, manual promises) — it creates invisible waterfall dependencies.

---

## Rendering

### 8. Streaming SSR HTML

Traditional SSR renders the full HTML string on the server before sending anything — the browser waits. Streaming SSR (React 18 + `renderToPipeableStream`) sends HTML chunks to the browser as they become ready, using HTTP chunked transfer encoding. Above-the-fold content arrives and becomes interactive before slower data-dependent sections finish rendering.

**Key Pointers**

- `renderToPipeableStream` replaces `renderToString` — it accepts `onShellReady` (first meaningful content) and `onAllReady` (full document) callbacks.
- Suspense boundaries act as streaming cut points — React flushes everything outside Suspense immediately, then streams each boundary's content when its data resolves.
- The browser's streaming HTML parser processes chunks incrementally — it can start downloading images and CSS referenced in the first chunk before the last chunk arrives.
- React inlines a small script with each streamed Suspense chunk to move the content from a hidden template into the placeholder — this is done without a full re-render.
- Headers (including `Set-Cookie`) must be sent before the body stream starts — you cannot set response headers after streaming begins.
- Next.js App Router uses streaming SSR by default for every async Server Component — the `loading.tsx` file defines the Suspense fallback for the route.

---

### 9. Hydration

Hydration is the process of attaching React's event system and state to server-rendered HTML that already exists in the DOM. React 18 introduced selective hydration — it can prioritize hydrating components the user interacts with first, even before the rest of the page finishes hydrating.

**Key Pointers**

- `hydrateRoot` (React 18) replaces `ReactDOM.hydrate` — it enables concurrent hydration and selective prioritization.
- A hydration mismatch (server HTML ≠ client render) causes React to discard the server HTML and re-render from scratch — very expensive and logged as a warning.
- Common mismatch causes: `Date.now()` / `Math.random()` in render, browser-only globals (`window`), locale-sensitive formatting, browser extensions mutating the DOM.
- Selective hydration: clicking a Suspense boundary before it hydrates causes React to synchronously hydrate just that boundary first.
- Islands architecture and Astro's partial hydration are motivated by the observation that hydration cost is proportional to JS bundle size, not HTML size.
- `suppressHydrationWarning` on a DOM element tells React to skip mismatch checking for that element — useful for timestamps that legitimately differ.

---

### 10. ISR – Incremental Static Regeneration

ISR (Next.js) allows statically generated pages to be updated after build time without a full rebuild. Pages are cached at the CDN edge and regenerated in the background when their revalidation window expires — visitors always get the cached version instantly, and the next request after expiry triggers a background rebuild.

**Key Pointers**

- `revalidate: 60` means the page may be stale for up to 60 seconds — after expiry, the next request serves the stale page but triggers a background regeneration.
- On-demand ISR (`revalidatePath` / `revalidateTag`) allows external webhooks (CMS updates, e-commerce inventory) to purge specific pages immediately.
- ISR is stale-while-revalidate at the infrastructure level — identical semantics to the `Cache-Control: stale-while-revalidate` HTTP header.
- ISR pages are stored in the Next.js file system cache (or distributed CDN) — they're real HTML files, not server-rendered on each request.
- `fallback: 'blocking'` causes the first request for an un-generated page to SSR synchronously and then cache; `fallback: true` shows a skeleton while SSR runs.
- ISR doesn't help for highly personalized pages — those should use SSR or client-side fetching. ISR shines for shared, cacheable content.

---

### 11. Critical Rendering Path

The Critical Rendering Path is the sequence of steps a browser must complete before it can show any content: download HTML, parse it (blocked by sync scripts), download CSS (render-blocking), build CSSOM, build Render Tree, Layout, Paint. Optimizing CRP directly reduces First Contentful Paint.

**Key Pointers**

- Render-blocking resources (CSS in `<head>`, synchronous `<script>`) delay the first paint. Inline critical CSS and defer non-critical CSS with `media='print'` + onload swap.
- `<script async>` downloads in parallel but still blocks parsing when it executes. `<script defer>` executes after HTML parsing — prefer `defer` for non-critical scripts.
- The HTTP `103 Early Hints` status code lets the server hint preloads before the full HTML response is ready — the browser starts fetching assets during server think time.
- Font render-blocking: `@font-face` without `font-display` causes FOIT (Flash of Invisible Text). `font-display: swap` shows fallback text immediately.
- Resource hints: `<link rel='preconnect'>` for third-party origins, `<link rel='preload' as='...'>` for critical assets, `<link rel='prefetch'>` for next-page assets.
- Chrome's rendering engine (Blink) parallelizes many parsing tasks — speculative preload scanner fetches `img`/`script`/`link` hrefs before full parse is complete.

---

### 12. Browser Rendering Pipeline

The browser rendering pipeline takes HTML/CSS/JS and produces pixels. The stages are: Parse HTML → Build DOM, Parse CSS → Build CSSOM, Merge → Render Tree, Layout (Reflow), Paint, Composite. Understanding which stages each CSS property triggers is key to jank-free animations.

**Key Pointers**

- `transform` and `opacity` only trigger Composite — they're the only CSS properties that can animate at 60fps without involving Layout or Paint.
- Changing `width`, `height`, `margin`, or `top` triggers Layout → Paint → Composite — the full pipeline. Avoid animating these properties.
- `will-change: transform` promotes an element to its own compositor layer, enabling GPU-accelerated compositing. Use sparingly — layers consume GPU memory.
- Forced synchronous layouts (FSL) occur when you read layout properties (`offsetHeight`, `getBoundingClientRect`) after a DOM write in the same frame. Batch reads before writes.
- CSS containment (`contain: layout paint`) limits the scope of reflow — a change inside a contained element doesn't trigger layout on the rest of the page.
- `content-visibility: auto` defers rendering of off-screen content, functioning as a declarative virtual list for static content.

---

## Performance

### 13. Virtual List & Infinite Scrolling

Rendering thousands of DOM nodes simultaneously is expensive in both memory and layout time. Virtualization (windowing) solves this by only rendering the items visible in the viewport, plus a small overscan buffer. Libraries like `react-window` and `react-virtual` manage this automatically. Infinite scrolling extends this by fetching more data as the user approaches the list end.

**Key Pointers**

- `react-window`'s `FixedSizeList` is fastest — it assumes all rows are equal height, enabling O(1) index-to-offset math. `VariableSizeList` requires a size cache.
- Intersection Observer is the modern primitive for detecting scroll-end — vastly cheaper than scroll event listeners with debounce.
- Items in a virtual list are absolutely positioned inside a "scroll container" of total height — the scroll container height gives correct scrollbar proportions.
- Overscan (rendering extra rows beyond the viewport) prevents blank flashes during fast scrolling — typical values are 3–5 rows.
- Virtualization breaks native browser find-in-page (Ctrl+F) — un-rendered rows are invisible to search. Mitigate with `aria-setsize` / `aria-posinset`.
- React Query / SWR's `useSuspenseInfiniteQuery` + `fetchNextPage` pattern is the recommended data-fetching approach for infinite lists in modern React.

---

### 14. Web Vitals & Core Metrics

Core Web Vitals are Google's standardized metrics for real-world user experience. The three core metrics are Largest Contentful Paint (LCP), Interaction to Next Paint (INP, replaced FID in 2024), and Cumulative Layout Shift (CLS). These directly influence Google search ranking and are measurable via the `web-vitals` JS library or CrUX field data.

**Key Pointers**

- **LCP** measures when the largest above-the-fold image or text block finishes rendering — target < 2.5s. Optimize with preload, early server response, and no render-blocking resources.
- **INP** (Interaction to Next Paint) replaced FID in March 2024 — it measures the worst interaction latency across a full page visit, not just first input. Target < 200ms.
- **CLS** measures unexpected layout shifts — each shift is scored as (impact fraction × distance fraction). Target < 0.1. Avoid inserting content above existing content without reserved space.
- TTFB (Time to First Byte) is not a Core Web Vital but affects LCP — slow servers, no CDN, and uncached SSR are top causes.
- Long Tasks (>50ms on the main thread) are the root cause of poor INP — identify them with Chrome DevTools Performance panel or the `PerformanceObserver` API.
- Use the `web-vitals` npm package to report real user metrics (RUM) to your analytics — lab tools (Lighthouse) simulate conditions, field data reflects actual users.

---

### 15. Code Splitting & Lazy Loading

Code splitting divides a JS bundle into smaller chunks loaded on demand. `React.lazy` + `Suspense` enable component-level splitting. Route-based splitting (Next.js App Router does this automatically) is the highest-leverage split — each route only loads its own code.

**Key Pointers**

- `React.lazy(() => import('./Heavy'))` wraps a dynamic import — React defers loading until the component is first rendered.
- Suspense boundaries are required for lazy components — they show a fallback while the chunk downloads and evaluates.
- Preload critical chunks with `<link rel='modulepreload'>` or webpack's `/* webpackPrefetch */` magic comment to avoid waterfall delays on navigation.
- Bundle analyzer (`webpack-bundle-analyzer`, Vite's `rollup-plugin-visualizer`) is essential for identifying what's inside each chunk.
- Third-party scripts (analytics, chat widgets) should use dynamic import with `{ ssr: false }` in Next.js to avoid SSR overhead and hydration cost.
- Granular splitting can backfire — too many tiny chunks create HTTP round-trip overhead. Aim for chunks of 50–150 KB gzipped.

---

### 16. Image Optimization

Images account for 50–75% of page weight on most sites. Modern optimization involves using next-gen formats (WebP, AVIF), correct sizing (`srcset`/`sizes`), lazy loading, and blur-up placeholder patterns. Next.js Image component and Cloudinary automate most of this.

**Key Pointers**

- AVIF achieves ~50% smaller files than JPEG at equal quality — use `<picture>` with AVIF → WebP → JPEG fallback for broad compatibility.
- `srcset` and `sizes` let the browser pick the optimal image for the device pixel ratio and viewport — ship 1x, 1.5x, 2x variants.
- `loading='lazy'` defers images below the fold — built into all major browsers. Combine with `decoding='async'` to prevent layout jank on decode.
- The Largest Contentful Paint image should be preloaded with `<link rel='preload' as='image'>` and must NOT be lazy-loaded.
- LQIP (Low Quality Image Placeholder) — a tiny blurred base64 image shown while the full image loads — eliminates layout shift and perceived blank space.
- Content-Aware Image CDNs (Cloudinary, Imgix, Fastly) handle format negotiation, on-the-fly resizing, and smart cropping via URL parameters.

---

### 17. Bundle Optimization & Tree Shaking

Tree shaking eliminates dead code from ES module bundles — only exported symbols that are actually imported survive. Effective tree shaking requires ES modules (not CommonJS), named exports, and side-effect-free module declarations.

**Key Pointers**

- CommonJS (`require`) cannot be statically analyzed — tree shaking doesn't work. Always prefer ESM. Many older libs ship both; set `moduleResolution: bundler` in `tsconfig`.
- `sideEffects: false` in `package.json` signals to Webpack/Rollup that the package is safe to tree-shake — without it, the bundler may keep entire files.
- Dynamic imports (`import()`) split bundles at natural boundaries — combine with `React.lazy` for route-based splitting with minimal boilerplate.
- Bundle analysis: use `source-map-explorer` or `bundlephobia.com` to inspect what's inside your production bundle before and after changes.
- Vite uses Rollup for production builds and esbuild for development — Rollup's scope hoisting produces smaller bundles than Webpack's module wrapping.
- Lodash is a notorious tree-shaking failure in legacy setups — `import { debounce } from 'lodash-es'` (ESM build) instead of `import _ from 'lodash'`.

---

### 18. CSS Performance & Containment

CSS selectors, property changes, and layout interactions all have performance costs. Modern CSS features like containment, `content-visibility`, and `@layer` provide new performance and specificity tools.

**Key Pointers**

- CSS selector performance is rarely a bottleneck today — browser matching is right-to-left and highly optimized. Don't obsess over selector depth.
- `contain: strict` (= `layout paint style size`) fully isolates an element — changes inside cannot affect anything outside. Use on widget/card components.
- `content-visibility: auto` skips rendering and layout of off-screen elements — measured 7× rendering improvement by Una Kravets on long document pages.
- `@layer` gives explicit cascade layer control — third-party resets, base styles, components, utilities, overrides can each occupy a named layer, eliminating specificity wars.
- `@property` (Houdini) registers typed CSS custom properties with a defined syntax, initial value, and inheritance — enables animating custom property values with transitions.
- Avoid scroll-linked animations with JS — use CSS `@scroll-timeline` or the Animation Worklet API to run them off the main thread.

---

## Architecture

### 19. Island Architecture

Island Architecture (coined by Jason Miller, popularized by Astro) treats a page as mostly static HTML with isolated interactive "islands" of JavaScript. Each island hydrates independently with its own bundle. The vast majority of the page ships zero JS, dramatically reducing Time to Interactive.

**Key Pointers**

- Astro's `client:` directives control island hydration strategy — `client:load` (immediate), `client:idle` (`requestIdleCallback`), `client:visible` (IntersectionObserver), `client:media` (`matchMedia`).
- Islands are isolated — they don't share a React root or virtual DOM. Communication between islands requires a shared store (nanostores, signals) or custom events.
- The pattern is ideal for content-heavy pages (marketing, blogs, docs) with a few interactive widgets — not ideal for highly interactive apps like dashboards.
- Compared to Next.js, islands produce much smaller JS payloads because only island code ships — the framework runtime is not needed for static sections.
- Fresh (Deno) and Marko are other frameworks built around island architecture, each with different hydration primitives.
- Progressive enhancement is a natural fit — islands can render meaningful server HTML as a fallback if the JS fails to load or hydrate.

---

### 20. Micro-frontends

Micro-frontends extend microservice principles to the UI layer — different teams own, build, and deploy independent pieces of the frontend. Integration strategies include: build-time npm packages, iframe embedding, server-side composition (ESI, SSI), or runtime module federation.

**Key Pointers**

- Webpack 5 Module Federation is the dominant runtime approach — host apps consume remote apps' components as if they were local, with shared dependency deduplication.
- Shared singleton state (Redux store, React context) doesn't work across separate bundles — use custom events, `BroadcastChannel`, or shared URL state for cross-MFE communication.
- Each MFE brings its own framework version risk — mismatched React versions on one page require isolated rendering roots and careful singleton avoidance.
- CSS isolation is critical — use CSS Modules, Shadow DOM, or scoped BEM conventions to prevent style bleed between MFEs.
- The biggest operational win: independent deployability. Teams can ship without coordinating releases. The biggest cost: duplicated dependencies increase bundle size.
- Single-SPA is a meta-framework for orchestrating multiple MFEs — it manages lifecycle methods (bootstrap, mount, unmount) for each app on route change.

---

### 21. State Management Patterns

State management in React has evolved from centralized stores (Redux) to distributed atomic stores (Zustand, Jotai) to server-state libraries (React Query, SWR). Choosing the right tool depends on whether the state is client-only, server-synced, or global UI state.

**Key Pointers**

- Server state (fetched data) and client state (UI flags, form state) have fundamentally different lifecycles — mixing them in Redux is the root of most over-engineering.
- React Query / TanStack Query handles server state: caching, deduplication, background refetch, optimistic updates, pagination — without a custom Redux slice.
- Zustand uses a single store with hooks — no Provider required, no boilerplate reducers. It uses immer-style mutations under the hood if you use the immer middleware.
- Jotai and Recoil are atom-based — state is split into fine-grained atoms, so updates only re-render components that subscribe to the changed atom.
- URL state (search params, hash) is underutilized — it's free state that persists across refreshes and is shareable. `nuqs` / `use-query-params` wraps this cleanly.
- Redux is still appropriate for complex, cross-cutting state with time-travel debugging needs — Redux Toolkit reduces boilerplate significantly.

---

### 22. Edge Computing & Edge Functions

Edge Functions run server-side code in PoPs (Points of Presence) geographically close to users — typically 50–300+ locations worldwide. They're ideal for request-time personalization, A/B testing, auth checks, and geolocation logic with sub-10ms added latency.

**Key Pointers**

- Edge runtimes are V8 isolates (Vercel Edge, Cloudflare Workers) — not Node.js. Node-specific APIs (`fs`, `net`, some `process.env`) are unavailable.
- Cold starts in V8 isolates are ~0ms (isolates reuse the same process) — far better than Lambda/container cold starts of 100–500ms.
- Middleware in Next.js runs on the edge by default — perfect for JWT validation, redirect logic, and setting personalization cookies before SSR.
- KV stores (Cloudflare KV, Vercel KV) provide eventually-consistent key-value storage from the edge — eventual consistency means stale reads are possible.
- Durable Objects (Cloudflare) provide strongly-consistent, stateful edge computation — each Object has a single-threaded location, enabling WebSocket servers at the edge.
- Regional deployments (Vercel Regional Edge, Fly.io) offer stronger consistency than global edge by limiting the number of possible replica locations.

---

### 23. Design Tokens & CSS Architecture

Design tokens are named values that represent design decisions (color, spacing, typography) — they're the single source of truth between design tools (Figma) and code. CSS Custom Properties are the native platform primitive for tokens.

**Key Pointers**

- Token taxonomy: Global tokens (`--color-blue-500`) → Semantic tokens (`--color-primary`) → Component tokens (`--button-bg`). Never skip the semantic layer.
- Style Dictionary (Amazon) transforms token JSON into CSS vars, SCSS, iOS Swift, Android XML, and JS objects from one source — the industry standard token pipeline.
- Tokens in Figma Variables sync to code via plugins (Tokens Studio, Figma's own REST API) — design and code stay in sync without manual copy-paste.
- Dark mode: swap semantic token values at the `:root [data-theme='dark']` selector — components never need to know about light/dark; they just use semantic tokens.
- CSS `@layer` + tokens = clean architecture: define token custom properties in a base layer, component styles in a components layer, utilities on top.
- Avoid magic numbers in component CSS — every spacing, radius, and color value should trace back to a token. Code reviews should flag raw hex values.

---

## Network

### 24. HTTP/2 & HTTP/3 (QUIC)

HTTP/2 introduced multiplexing (multiple requests over one TCP connection), header compression (HPACK), and server push. HTTP/3 replaces TCP with QUIC (UDP-based), eliminating head-of-line blocking at the transport level and improving performance on lossy networks.

**Key Pointers**

- HTTP/2 multiplexing makes domain sharding and asset concatenation (spriting) anti-patterns — spreading assets across domains defeats connection reuse.
- HTTP/2 server push was deprecated in Chrome 106 — use `<link rel='preload'>` or `103 Early Hints` instead to push critical resources.
- QUIC (HTTP/3) uses connection IDs rather than IP:port tuples — connections survive network changes (Wi-Fi → cellular) without renegotiation.
- 0-RTT resumption in QUIC allows data to be sent on the very first packet after reconnect — but is vulnerable to replay attacks; use only for idempotent requests.
- HTTP/3 adoption: most CDNs (Cloudflare, Fastly, CloudFront) support it — enable it in your CDN config; browsers negotiate via `Alt-Svc` or HTTPS DNS records.
- TLS 1.3 is required for HTTP/2 in all major browsers — it reduces handshake round trips from 2 to 1, improving TTFB on fresh connections.

---

### 25. Caching Strategies

Effective caching is the single highest-leverage performance optimization. The main layers are: browser cache (Cache-Control headers), CDN edge cache, service worker cache (offline), and application-level cache (React Query, SWR). Each layer has different invalidation characteristics.

**Key Pointers**

- `Cache-Control: max-age=31536000, immutable` is the ideal header for content-hashed assets — permanent cache, never revalidated.
- `Cache-Control: no-cache` means "always revalidate with server" (not "don't cache") — the browser sends a conditional request (ETag/Last-Modified) and gets `304` if unchanged.
- `stale-while-revalidate` serves cached content immediately while fetching a fresh copy in the background — excellent for non-critical data with acceptable staleness.
- Service workers intercept all fetch requests and can implement any caching strategy: Cache First, Network First, Stale-While-Revalidate, or Cache Only.
- CDN cache keys default to URL — vary the cache on `Accept-Encoding` for compression and `Accept` for content negotiation, but avoid `Vary: Cookie` (defeats CDN caching).
- Cache poisoning: never include user-controlled input in cached responses without sanitization — a poisoned CDN response affects all users sharing that cache key.

---

### 26. WebSockets & Real-Time Patterns

WebSockets provide a persistent, bidirectional TCP connection between browser and server — enabling real-time data without polling. Modern alternatives include Server-Sent Events (SSE) for unidirectional server push, and WebTransport (QUIC-based) for high-performance bidirectional streams.

**Key Pointers**

- WebSockets use the `ws://` or `wss://` protocol — the connection starts as HTTP and upgrades via the `Upgrade` header. TLS is mandatory in production (`wss://`).
- SSE (`EventSource` API) is simpler for server-push-only use cases — uses regular HTTP/2, supports auto-reconnect and `Last-Event-ID` for resumption.
- Socket.io adds rooms, namespaces, auto-reconnect, and a fallback to polling — but it's a heavyweight wrapper; native WebSocket is preferred when you control both sides.
- Fan-out at scale: a single WebSocket server can't broadcast to millions of connections. Redis Pub/Sub or Kafka decouples message production from delivery across server instances.
- Presence and collaborative editing (like Figma or Notion) use CRDTs (Conflict-free Replicated Data Types) — Yjs and Automerge are the main JS implementations.
- WebTransport (QUIC) supports multiple streams and datagrams over a single connection — sub-frame latency for gaming and real-time media with no head-of-line blocking.

---

### 27. GraphQL & Data Fetching Patterns

GraphQL is a query language and runtime where clients specify exactly the data they need, preventing over-fetching and under-fetching. Modern patterns include fragments, persisted queries, and normalized caching with Apollo Client or URQL.

**Key Pointers**

- Fragments colocate data requirements with the component that needs them — Relay enforces this pattern strictly, Apollo/URQL allow it optionally.
- N+1 problem: a list query that fetches child data per item makes N extra requests. DataLoader batches and deduplicates these into a single query.
- Persisted queries (APQ) send a hash of the query instead of the full query string — reduces request size, enables GET requests, and allows CDN caching.
- Apollo Client's normalized cache stores each entity by `__typename + id` — subsequent queries that touch the same entity update all components referencing it.
- Subscriptions use WebSockets (`graphql-ws`) for real-time data — defer to SSE or polling when WebSocket infrastructure isn't available.
- REST vs GraphQL: REST has better HTTP caching, simpler mental model, and no client library needed. GraphQL wins when clients have diverse data needs or when avoiding over-fetching matters.

---

## Browser

### 28. Web Workers & Offthread Work

Web Workers run JavaScript on a separate OS thread, keeping the main thread free for UI. They have no DOM access but can use `fetch`, IndexedDB, WebSockets, and most APIs. Shared Workers and Service Workers are specialized variants. Comlink provides a clean RPC wrapper over `postMessage`.

**Key Pointers**

- `postMessage` serializes data using structured clone — it's asynchronous and copies data (except Transferable objects like `ArrayBuffer`).
- Transferable objects (`ArrayBuffer`, `ImageBitmap`, `OffscreenCanvas`) are moved (not copied) to the worker, making large data transfers O(1).
- `OffscreenCanvas` allows canvas rendering in a worker — useful for chart libraries and WebGL — the compositing still happens on the main thread.
- Comlink (by Surma/Google) wraps workers in a Proxy — you call worker methods like async functions and Comlink handles `postMessage`/response matching.
- WASM execution in workers is extremely common for compute-heavy tasks (video encoding, cryptography, image processing) — keeps the main thread free.
- Service Workers are a special type that persist across pages, intercept network requests, and enable push notifications and background sync.

---

### 29. Web Assembly (WASM)

WebAssembly is a binary instruction format that runs in browsers at near-native speed. It's not a replacement for JavaScript — JS calls into WASM for compute-intensive tasks. Languages like C, C++, Rust, Go, and AssemblyScript compile to WASM.

**Key Pointers**

- WASM runs in the same sandbox as JS — it has no direct DOM access and must call JS functions imported as WASM imports for DOM interaction.
- Linear memory is a contiguous `ArrayBuffer` shared between JS and WASM — avoid excessive copying; pass pointers, not values, for large data.
- WASM Threads (via `SharedArrayBuffer` + `Atomics`) enable true parallel execution — requires COOP/COEP headers (cross-origin isolation) on the page.
- WASI (WebAssembly System Interface) allows WASM modules to run outside browsers (on servers, edge) with controlled access to OS resources.
- `ffmpeg.wasm`, `SQLite.wasm`, and TensorFlow.js WASM backend are real-world examples delivering C-level performance in browsers.
- WASM binary size: Rust + `wasm-pack` produces very small binaries; Emscripten-compiled C++ can produce larger runtimes. Use `wasm-opt` to minimize.

---

### 30. Security: XSS, CSP & CSRF

Cross-Site Scripting (XSS) allows attackers to inject malicious scripts into pages viewed by other users. Content Security Policy (CSP) is the browser's enforcement mechanism. CSRF tricks authenticated users into making unintended requests. Both are solved with a combination of framework defaults and HTTP headers.

**Key Pointers**

- React escapes all JSX expressions by default — `dangerouslySetInnerHTML` bypasses this and must be sanitized with DOMPurify before rendering user content.
- CSP via `Content-Security-Policy` header restricts which origins can load scripts, styles, images, and frames — nonce-based CSP is the most effective approach.
- CSRF tokens (synchronizer pattern) or `SameSite=Strict` cookies mitigate CSRF — modern browsers honor `SameSite` by default, but explicit headers are still required for older clients.
- Subresource Integrity (SRI) — `integrity` attribute on `<script>` and `<link>` — verifies that CDN-served files haven't been tampered with.
- HTTP security headers: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Permissions-Policy`. Run `securityheaders.com` to audit.
- Never store sensitive tokens in `localStorage` — XSS on any page can read it. Use `httpOnly`, `Secure`, `SameSite=Strict` cookies for auth tokens.

---

*30 topics · React Internals · Rendering · Performance · Architecture · Network · Browser*