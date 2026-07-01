# Performance Optimization Learning Track (optional, ship-first friendly)

> **Status:** an optional *learning track*, not a rewrite of the plan. Unlike microservices (which would slow shipping), performance work **optimizes what you have already shipped** — so it never blocks the mainline. Rule: **ship the slice first, then optimize it.**
>
> Sibling to [realtime-service.md](./realtime-service.md). Everything here is "code written by you" — this is a study guide, not an implementation.

---

## The one principle that beats every trick: **Measure First**

Most people optimize the wrong thing because they guess. The professional loop is:

1. **Baseline** — measure current latency/throughput/size with numbers, not vibes.
2. **Profile** — use a profiler to find the *real* bottleneck (it's rarely where you think).
3. **Optimize** — change one thing.
4. **Re-measure** — confirm it actually helped; keep or revert.

Learn the tools that make this possible: **flame graphs** (CPU), **k6 / wrk** (load testing), **p50 / p95 / p99 latency** (tail matters more than average), memory/allocation profiles. Avoid **premature optimization** — don't micro-optimize a path that isn't hot.

---

## ⭐ Star topic — E2EE performance: envelope (hybrid) encryption

The single highest-value optimization in this project, because it's specific to ghostline *and* teaches deep concepts.

**Problem:** `tweetnacl` is pure JS. Encrypting a 20–50MB DSLR photo (Preserve-quality mode) with `nacl.box` is slow enough to jank the UI.

**Fix — envelope encryption (what real systems do):**
- Generate a random **symmetric key** per message/file.
- Encrypt the **big payload** with **AES-GCM via WebCrypto** (native, **hardware-accelerated** — often 10–50× faster than JS) — or a native crypto module on RN.
- Encrypt only the small **symmetric key** with `nacl.box` (recipient's public key). Fan out the tiny encrypted key per recipient; upload the single AES-GCM blob once.
- Run the crypto in a **Web Worker / native thread** so the main thread stays at 60fps.

**What you learn:** symmetric vs asymmetric crypto, the hybrid/envelope pattern, hardware acceleration, AEAD (authenticated encryption), off-main-thread work, and throughput benchmarking. Pairs directly with the compression + Preserve-quality media work (ROADMAP clarification 10).

> Security note: keep AEAD nonces unique (never reuse an AES-GCM nonce under the same key); benchmark, but don't hand-roll the primitives — use WebCrypto / libsodium.

---

## Area-by-area mini-curriculum

Each area is worth a focused study + a measured before/after.

### 1. Client rendering (React Native + Web)
- **Long message lists:** virtualization with `FlashList` (or `FlatList` tuned) — don't render 10k rows.
- **Kill needless re-renders:** `React.memo`, `useMemo`/`useCallback`, narrow Zustand selectors, stable keys.
- **Animations at 60fps:** keep Reanimated work on the **UI thread** (worklets); verify Letter Mode open/burn stays smooth on a low-end phone and in the browser.
- **Startup & bundle:** Hermes engine (native), code-splitting / lazy routes (web), measure **Core Web Vitals** (LCP, CLS, **INP**) with Lighthouse.

### 2. Media pipeline
- Compression (clarification 10) — measure size/time per codec/level.
- **Thumbnails + progressive load** so a 20MB original doesn't block the timeline; cache decoded images.
- Lazy-load off-screen media; cancel in-flight loads on scroll.

### 3. Network / realtime
- **Payload compression** and **message batching**; measure bytes-on-wire.
- **Backpressure** on the socket; efficient **reconnect + backfill** (don't refetch everything).
- **HTTP/2**, **TLS session resumption**, keep-alive / connection pooling.
- Track end-to-end **latency percentiles**, not averages.

### 4. Backend — Node (main API)
- Watch the **event loop** (don't block it); move CPU work to **`worker_threads`**.
- **Stream** large responses instead of buffering.
- Profile with **clinic.js / --prof / flame graphs**.
- **DB connection pooling**; reuse prepared statements.

### 5. Backend — Go (realtime fan-out)
- **Goroutine pools**, buffered **channels**, explicit **backpressure**.
- Reduce allocations (**`sync.Pool`**, preallocated buffers); profile with **pprof**; understand **GC** pauses.
- Benchmark fan-out throughput (messages/sec to N connections) as group size grows toward the 30 cap.

### 6. Database (PostgreSQL)
- **Indexes** on hot query columns; read plans with **`EXPLAIN ANALYZE`**.
- Eliminate **N+1** queries; batch.
- **Connection pooling** (PgBouncer if needed), prepared statements.
- Measure query p95 under load.

### 7. Load testing + observability
- **Load test** with **k6 / wrk** — find the throughput knee and the failure mode.
- **Prometheus + Grafana** for latency/throughput/error-rate dashboards (lightweight — this is the *good* part of the microservices observability stack, without the microservices).
- Define **latency targets / SLIs** (e.g. "message send p95 < 150ms") so "fast enough" is a number, not a feeling.

---

## Lightweight tooling on one VPS

k6 (load), Prometheus + Grafana (metrics), pprof (Go), clinic.js/0x (Node), Lighthouse/React DevTools Profiler (client). All fit a 6GB box; run load tests against staging, not prod.

## How it maps to the mainline (optimize after shipping, not before)

| Ships in | Optimize afterward |
| --- | --- |
| Slice 1 (realtime chat) | render perf, re-render elimination, latency baseline |
| Slice 3 (E2EE) | ⭐ envelope encryption, off-thread crypto |
| Slice 4 (media) | compression, thumbnails, AES-GCM bulk path, caching |
| Slice 6 (Go fan-out) | goroutine pools, backpressure, pprof, throughput at 30-cap |
| Slice 7 (release) | load testing, Prometheus/Grafana, Core Web Vitals, SLIs |

## Anti-patterns to avoid

- Optimizing **without measuring** (guessing the bottleneck).
- **Premature** optimization of cold paths.
- Chasing micro-benchmarks that don't move real user-facing latency.
- Adding caches/threads that add complexity for no measured win.
