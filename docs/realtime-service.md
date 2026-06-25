# Ghostline Realtime Fan-out Service — Production Build Guide (Go)

> **What this is.** A from-scratch build guide for the standalone Go service introduced in
> [ROADMAP](../ROADMAP.md) slice 6: a concurrent WebSocket hub that owns live connections,
> presence, room (group) fan-out, and direct messages.
>
> **How to use it.** This guide gives you the *architecture, the type contracts, the concurrency
> patterns, and the production checklist* — **you write every line of the implementation yourself.**
> Where you see `// TODO(you)`, that body is yours to fill. The shapes around it (struct fields,
> function signatures, the `select` skeletons) are the scaffolding you reason against, not code to
> copy blindly.
>
> **The one invariant you must never break.** This service is a *ciphertext courier*. It routes
> opaque bytes between devices. It never decrypts, never inspects message bodies, never persists
> plaintext. Encryption stays on the phones (per-recipient fan-out, see ROADMAP clarification 4).
> Every design decision below bends to keep this true.

---

## Table of contents

1. [Why Go, why a hub](#1-why-go-why-a-hub)
2. [The mental model: one owner, many workers](#2-the-mental-model-one-owner-many-workers)
3. [Architecture](#3-architecture)
4. [The wire protocol](#4-the-wire-protocol)
5. [Core types — the contracts you implement](#5-core-types--the-contracts-you-implement)
6. [The connection lifecycle: readPump & writePump](#6-the-connection-lifecycle-readpump--writepump)
7. [The Hub loop: select multiplexing](#7-the-hub-loop-select-multiplexing)
8. [Rooms and direct messages](#8-rooms-and-direct-messages)
9. [Presence and the RWMutex question](#9-presence-and-the-rwmutex-question)
10. [Production hardening](#10-production-hardening)
11. [Horizontal scaling with Redis pub/sub](#11-horizontal-scaling-with-redis-pubsub)
12. [Integrating with the Node backend](#12-integrating-with-the-node-backend)
13. [Observability](#13-observability)
14. [Testing strategy](#14-testing-strategy)
15. [Project layout](#15-project-layout)
16. [Learning path & checkpoints](#16-learning-path--checkpoints)
17. [Pitfalls checklist](#17-pitfalls-checklist)
18. [Glossary](#18-glossary)

---

## 1. Why Go, why a hub

Real-time chat is a *fan-out* problem: one inbound message must reach N outbound connections, and
those N connections live and die independently and unpredictably. Two properties make this hard:

- **Concurrency.** Hundreds or thousands of sockets are readable/writable at unpredictable moments.
- **Shared mutable state.** "Who is online", "who is in room X" is read and written constantly.

Go is a strong fit because a goroutine is cheap enough (a few KB of stack) to dedicate **one per
direction per connection** — you write straight-line blocking code (`conn.Read`, `conn.Write`) and
the runtime scheduler multiplexes thousands of them onto a handful of OS threads. You never touch an
event loop or a callback. The cost you pay instead is **coordination**: those goroutines share state,
and getting that coordination right (without data races or deadlocks) is the entire skill this slice
teaches.

The **hub pattern** is the canonical answer. Read it carefully — it is the spine of everything below.

---

## 2. The mental model: one owner, many workers

There are two ways to protect shared state in Go. You will use **both**, deliberately, for different
state — knowing *which* and *why* is the core lesson.

**Pattern A — Communicating Sequential Processes (CSP): "share memory by communicating."**
One goroutine *owns* a piece of state. Nobody else touches it. Other goroutines send it *requests*
over channels; it processes them one at a time, in its own `for { select { ... } }` loop. Because
exactly one goroutine ever reads or writes that state, **there is no race — no mutex needed.**
This is how the **Hub owns the rooms/clients registry.**

**Pattern B — shared memory + mutex.**
Multiple goroutines read/write the same map directly, guarded by a `sync.RWMutex`. Cheaper for
*read-mostly* state that many goroutines need to query synchronously without a round-trip through a
channel. This is how a **presence snapshot** (read by many, written rarely) is best served.

> **The trap to avoid:** reaching for a mutex *everywhere*. A `select`-driven owner goroutine is
> often simpler and deadlock-resistant than a web of locks. Use the mutex only where a synchronous,
> read-mostly query genuinely beats message-passing. We mark exactly where, in §9.

```
        Pattern A (Hub)                         Pattern B (Presence)
   ┌───────────────────────┐              ┌───────────────────────────┐
   │  single Hub goroutine │              │  shared map[UserID]Status │
   │  owns rooms + clients │              │  guarded by sync.RWMutex  │
   └──────────▲────────────┘              └──────▲──────────▲─────────┘
              │ channels                          │ RLock     │ Lock
      register│unregister│broadcast        many readers   rare writer
```

---

## 3. Architecture

```
                          ┌──────────────────────────────────────────┐
   phone  ──WebSocket──►  │  GO REALTIME SERVICE  (this guide)         │
   phone  ──WebSocket──►  │                                            │
   phone  ──WebSocket──►  │   Client (readPump goroutine)              │
                          │      └─ inbound ─► Hub.inbound chan        │
                          │   Client (writePump goroutine)             │
                          │      ◄─ send chan ─┘                       │
                          │                                            │
                          │   Hub goroutine (select loop, owns state): │
                          │      rooms:   map[RoomID]map[*Client]bool  │
                          │      clients: map[UserID]*Client           │
                          │                                            │
                          │   Presence (sync.RWMutex map)              │
                          └───────────▲─────────────────▲─────────────┘
                                      │ Redis pub/sub    │ HTTP (internal)
                          ┌───────────┴─────────┐   ┌────┴──────────────┐
                          │  Redis (Upstash)    │   │  Node main backend │
                          │  cross-instance bus │   │  auth, storage,    │
                          │  + presence shared  │   │  letters, history  │
                          └─────────────────────┘   └───────────────────┘
```

**Boundaries (memorize these):**

- The Go service holds the **transport**. It never owns the database and never sees plaintext.
- The Node backend holds **truth**: who exists, what the invite code was, message history (as
  ciphertext), letter scheduling. It hands the Go service a *signed token* per connection.
- **Redis** is the only thing the Go and Node processes share — both for cross-instance fan-out
  (§11) and as the decoupling bus from Node (§12). This is your single, deliberate "one cut".
- **Language-agnostic.** This service routes ciphertext and emits only fixed `error` *codes* (e.g.
  `RATE_LIMITED`), never human-facing copy. The app's i18n (English-first, ja/zh later) lives entirely
  on the phone; clients map codes to localized text. So this service needs **no** i18n.

---

## 4. The wire protocol

Define the envelope **before** writing a single handler. A clear protocol is what lets you grow the
service without rewrites. Use a tagged JSON envelope — one outer shape, a `type` discriminator, and a
`body` that is **opaque to this service** (it is ciphertext; you route it, you never parse it).

```jsonc
// Client -> Server
{
  "type": "room.send" | "dm.send" | "room.join" | "room.leave" | "ping",
  "id":   "<client-generated message id, ULID/UUID>",  // for dedupe + ack correlation
  "to":   "<RoomID or UserID>",                          // routing target
  "body": "<base64 ciphertext — NEVER inspected here>",  // opaque
  "ts":   1719300000000
}

// Server -> Client
{
  "type": "room.msg" | "dm.msg" | "presence" | "ack" | "error" | "pong",
  "id":   "<echoed message id for ack, or server id for relayed msg>",
  "from": "<UserID>",
  "room": "<RoomID, when applicable>",
  "body": "<base64 ciphertext, untouched>",
  "ts":   1719300000000,
  "code": "RATE_LIMITED" | "UNAUTHORIZED" | ...   // only on type=error
}
```

**Design rules:**

- **Validate the envelope, never the body.** Reject unknown `type`, missing `to`, oversize frames —
  but treat `body` as bytes you forward verbatim.
- **Every client message carries a client-generated `id`.** You echo it in an `ack`. This is how the
  phone implements reliable send + dedupe (Ghostline GOALS §五).
- **Version it.** Add a `"v": 1` field now, or reserve a path, so you can evolve the protocol.

> **Teaching note.** `body` is ciphertext end to end. This service reads only the envelope
> (`type`/`to`/`id`) to decide where to route, and never parses `body`. That is the
> ciphertext-courier invariant expressed at the protocol layer.

---

## 5. Core types — the contracts you implement

These are the **signatures and field lists**. The bodies are yours. Resist the urge to add state to
`Client` that the Hub should own, or to let anything but the Hub touch the registries.

```go
package hub

// Client is one device's live connection. It owns exactly two goroutines:
// readPump (reads socket -> hub) and writePump (send chan -> socket).
type Client struct {
    UserID UserID
    conn   *websocket.Conn       // the underlying socket
    send   chan []byte           // BUFFERED outbound queue (see §6 on why buffered)
    hub    *Hub
    rooms  map[RoomID]struct{}   // rooms this client is in; touched ONLY by the hub goroutine
    // TODO(you): add only what a single connection legitimately needs.
}

// Hub owns ALL shared routing state. Only the hub goroutine reads/writes these maps.
type Hub struct {
    clients    map[UserID]*Client            // online clients (one device per user, for now)
    rooms      map[RoomID]map[*Client]bool   // room membership
    register   chan *Client
    unregister chan *Client
    inbound    chan Envelope                 // decoded messages from any readPump
    // TODO(you): shutdown signaling (context or a done chan).
}

// Envelope is the decoded wire message (§4). body stays []byte — opaque.
type Envelope struct {
    Type string
    ID   string
    To   string
    From UserID
    Body []byte
    TS   int64
}

// The methods you will implement:
func NewHub() *Hub                         // construct, init maps/chans
func (h *Hub) Run(ctx context.Context)     // THE select loop (§7). One goroutine.
func (c *Client) readPump()                // §6
func (c *Client) writePump()               // §6
```

**Why `send` is a field on `Client` and not a shared map:** the only goroutine that *writes* to a
client's socket is that client's own `writePump`. The hub never writes a socket directly — it pushes
bytes onto `client.send` and moves on. This keeps one slow/dead client from blocking the hub or any
other client. **This decoupling is the single most important production property in the design.**

---

## 6. The connection lifecycle: readPump & writePump

Per connection, exactly two goroutines. They never share mutable state with each other except the
socket's own concurrency guarantees: **gorilla/websocket allows one concurrent reader and one
concurrent writer, but not more.** So one goroutine reads, one writes — never two writers.

```go
// readPump runs in its own goroutine. It is the ONLY reader of c.conn.
func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c       // tell the hub we're gone
        _ = c.conn.Close()          // closing here unblocks writePump's next write
    }()

    // TODO(you): configure read limits & deadlines BEFORE the loop:
    //   c.conn.SetReadLimit(maxMessageBytes)
    //   c.conn.SetReadDeadline(now + pongWait)
    //   c.conn.SetPongHandler(func(string) error { reset read deadline; return nil })

    for {
        // TODO(you): _, data, err := c.conn.ReadMessage()
        //   on err: break (it's a normal close, a timeout, or a protocol error)
        //   decode the envelope; VALIDATE type/to/size; set From = c.UserID (never trust client)
        //   c.hub.inbound <- env
    }
}

// writePump runs in its own goroutine. It is the ONLY writer of c.conn.
func (c *Client) writePump() {
    ticker := time.NewTicker(pingPeriod)   // heartbeat
    defer func() { ticker.Stop(); _ = c.conn.Close() }()

    for {
        select {
        case msg, ok := <-c.send:
            // TODO(you): set write deadline; if !ok the hub closed us -> send CloseMessage, return;
            //   else WriteMessage(msg). On write error -> return (writePump exits, readPump follows).
        case <-ticker.C:
            // TODO(you): set write deadline; WriteMessage(PingMessage). On error -> return.
        }
    }
}
```

**Three production decisions baked into the skeleton above:**

1. **Heartbeat (ping/pong + deadlines).** Without it, a phone that drops off a subway tunnel leaves a
   zombie connection forever, leaking a goroutine and a slot. The writePump pings on a timer; the
   readPump expects a pong before `pongWait` or its read deadline fires and it exits. Rule of thumb:
   `pongWait ≈ 60s`, `pingPeriod ≈ (pongWait * 9 / 10)`.
2. **`SetReadLimit`.** A client must not be able to send a 2 GB frame and OOM your service. Cap it.
3. **Symmetric teardown.** When either pump exits it closes the socket, which unblocks the other
   pump, which exits too. Both `defer` cleanup. No goroutine survives a dead connection — **this is
   how you avoid the #1 production bug: goroutine leaks.**

> **Teaching note.** One connection = two goroutines (one read, one write). Whichever exits closes
> the socket, which wakes and ends the other. Heartbeats (ping/pong) + read/write deadlines are the
> key to preventing zombie connections and goroutine leaks.

---

## 7. The Hub loop: select multiplexing

This is the heart of the slice — the `select` that the spec asks you to "deeply understand". One
goroutine, one loop, owns all routing state, and **multiplexes** four event sources: registrations,
unregistrations, inbound messages, and shutdown. Because only this goroutine touches `clients` and
`rooms`, **no lock guards them.**

```go
func (h *Hub) Run(ctx context.Context) {
    for {
        select {
        case c := <-h.register:
            // TODO(you): add c to h.clients; update presence; maybe send a presence event.

        case c := <-h.unregister:
            // TODO(you): remove c from h.clients and from every room in c.rooms;
            //   close(c.send) EXACTLY ONCE (see pitfall); update presence.

        case env := <-h.inbound:
            // TODO(you): route by env.Type:
            //   room.send -> deliver(env, members of room env.To)
            //   dm.send   -> deliver(env, the single client env.To, if online)
            //   room.join/leave -> mutate membership
            //   then ack the sender.

        case <-ctx.Done():
            // TODO(you): graceful shutdown — close all clients, return (see §10).
            return
        }
    }
}
```

The critical sub-routine is **non-blocking delivery**:

```go
// deliver pushes bytes onto each recipient's send chan WITHOUT blocking the hub.
func (h *Hub) deliver(b []byte, recipients []*Client) {
    for _, c := range recipients {
        select {
        case c.send <- b:
            // queued fine
        default:
            // c's buffer is FULL -> it's a slow/stuck consumer.
            // TODO(you): drop the client: close(c.send) once, remove from registries.
            //   Do NOT block the hub waiting on one slow phone — that would freeze EVERYONE.
        }
    }
}
```

**Why the `default` case matters more than anything.** If you wrote `c.send <- b` *without* the
`select/default`, a single phone on a bad network whose buffer filled up would **block the hub
goroutine**, which would freeze message delivery for *every other user on the instance*. The
`default` branch turns "one slow client" into "drop that one client", protecting the whole system.
This is **backpressure**, and handling it is what separates a toy from a production service.

> Decision to make explicitly: on a full buffer, do you **drop the client** (simplest, correct for
> chat) or **drop the message** (keep the connection, lose a frame)? For E2EE chat where the phone
> re-syncs missed messages from Node's store-and-forward on reconnect, **drop the client** is the
> clean choice. Document whichever you pick.

---

## 8. Rooms and direct messages

**Rooms (group chat).** A room is just `map[*Client]bool`. Join adds, leave/disconnect removes.
Fan-out = iterate members, `deliver` to each. Remember the two-fan-out distinction from ROADMAP
clarification 4:

- **Encryption fan-out** happens on the *sender's phone*: it encrypts one copy per recipient public
  key and sends them as one logical message (or N envelopes). This service does **not** do this.
- **Transport fan-out** is *this service's* job: take the ciphertext frame(s) and route to the
  online members. Offline members are not your problem — they re-fetch from Node on reconnect.

> Design choice: do you ship one envelope with N embedded per-recipient ciphertexts, or N envelopes
> each addressed to one member? For small friend-group rooms, N envelopes is simpler and keeps this
> service dumb. Decide and write it in the protocol doc.

**Direct messages.** A DM is a "room of two" with no persistent membership — just look up
`clients[targetUserID]` and `deliver` if online, otherwise the message lives only in Node's
store-and-forward. Keep DM and room paths sharing the same `deliver` primitive; only routing differs.

**Authorization is Node's job, enforced with Node's token.** This service must verify that the
connecting user is *allowed* in a room — but it should not re-implement membership rules. Two clean
options:

1. The connection token (from Node, §12) embeds the room IDs the user may join; the hub trusts it.
2. The hub asks Redis/Node on join. Higher latency, always fresh. For friend scale, option 1 is fine.

---

## 9. Presence and the RWMutex question

This is where `sync.RWMutex` earns its place. "Who is online" is **read often** (every presence UI,
every DM routing check might consult it) and **written rarely** (only on connect/disconnect). That
read-mostly profile is the textbook case for `RWMutex`: many concurrent `RLock` readers, occasional
exclusive `Lock` writer.

```go
type Presence struct {
    mu     sync.RWMutex
    online map[UserID]int   // refcount: a user may have >1 connection later
}

func (p *Presence) Add(u UserID)  { /* TODO(you): Lock; online[u]++ */ }
func (p *Presence) Remove(u UserID){ /* TODO(you): Lock; if --online[u]==0 delete */ }
func (p *Presence) IsOnline(u UserID) bool { /* TODO(you): RLock; return online[u] > 0 */ }
func (p *Presence) Snapshot() []UserID     { /* TODO(you): RLock; copy keys out */ }
```

**Why a mutex here but not for the hub registries?** The hub registries are mutated *and read* only
inside one goroutine's loop — channels already serialize all access, so a lock would be redundant.
Presence is queried *synchronously from many goroutines* (e.g. a readPump checking if a DM target is
online) where routing a request through the hub channel just to read a bool would be wasteful. Match
the tool to the access pattern — **that judgment is the lesson, not the lock itself.**

**Always run with `-race`** (§14). The race detector is non-negotiable for this slice; it will catch
the moment you accidentally read a map without holding the lock.

> **Teaching note.** The hub's maps are touched by only one goroutine → channels already serialize
> access, so no lock is needed. Presence is read by many goroutines and written rarely → use a
> `RWMutex` (the classic read-mostly case). **Which one you pick depends on the access pattern** —
> that judgment is exactly the skill to learn.

---

## 10. Production hardening

A toy accepts a socket and echoes. A production service survives hostile clients, bad networks,
deploys, and its own panics. Implement each of these — they are what "生产级" means.

| Concern | What to build | Why |
| --- | --- | --- |
| **Handshake auth** | Validate a short-lived JWT (from Node) in the `Upgrade` request before accepting. Reject otherwise. | Only authenticated users get a connection. Never trust a `UserID` sent by the client. |
| **Origin check** | Set `Upgrader.CheckOrigin` to an allowlist (not `return true`). | Stops cross-site WebSocket hijacking from a browser context. |
| **Read limit & deadlines** | `SetReadLimit`, read/write deadlines, ping/pong (§6). | Bounds memory; kills zombies. |
| **Buffered send + backpressure** | Buffered `send` chan + `select/default` drop (§7). | One slow phone can't freeze the instance. |
| **Connection limits** | Cap total connections and per-user connections. | Protects a tiny free-tier box from exhaustion. |
| **Rate limiting** | Per-connection token bucket on inbound messages. | Stops a malicious or buggy client from flooding a room. |
| **Panic recovery** | `recover()` at the top of each pump goroutine; log and tear down that connection only. | A panic in one connection must not crash the process and drop everyone. |
| **Graceful shutdown** | On SIGTERM: stop accepting, `ctx.Cancel()`, send close frames, drain with a timeout, then exit. | Railway redeploys send SIGTERM. Clean shutdown = no abrupt mass-disconnect. |
| **Structured logging** | `slog` with request/connection IDs. **Never log `body`.** | Debuggable, and privacy-preserving (ciphertext-courier invariant even in logs). |

```go
// Panic recovery wrapper — wrap BOTH pumps.
func safeGo(name string, fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                // TODO(you): slog.Error("goroutine panic", "where", name, "err", r, stack)
            }
        }()
        fn()
    }()
}
```

```go
// Graceful shutdown shape.
// 1. signal.NotifyContext(ctx, SIGINT, SIGTERM)
// 2. srv.Shutdown(timeoutCtx)        // stop accepting new conns
// 3. cancel hub ctx                  // hub closes every client's send chan
// 4. wait (WaitGroup) for pumps to drain, bounded by a timeout
```

---

## 11. Horizontal scaling with Redis pub/sub

One Go instance holds connections in *its own* memory. The moment you run **two** instances (for
redundancy or load), a user on instance A and a user on instance B in the same room can't see each
other — their `rooms` maps are disjoint. This is the real lesson behind "why the realtime layer is
the first thing that needs independent scaling."

**The fix: Redis pub/sub as a cross-instance bus.**

```
   instance A                 Redis                  instance B
   hub.inbound ──publish──►  channel:room:<id>  ──►  subscriber ──► hub.deliver(local members)
   subscriber ◄────────────  channel:room:<id>  ◄──  hub.inbound ──publish
```

- On an inbound room message, the hub both delivers to **local** members *and* `PUBLISH`es the
  ciphertext to `room:<RoomID>`.
- Each instance runs a subscriber goroutine; on receiving a published frame, it delivers to **its
  own** local members of that room (and must **not** re-publish — avoid loops).
- **Presence goes to Redis too**: a shared `SET`/hash keyed by user, so any instance can answer
  "is this user online anywhere?". Use TTLs + heartbeats so a crashed instance's entries expire.

```go
type Bus interface {
    Publish(ctx context.Context, channel string, frame []byte) error
    Subscribe(ctx context.Context, channels ...string) (<-chan []byte, error)
}
// TODO(you): implement with github.com/redis/go-redis/v9; keep the interface so you can
// swap a no-op/in-memory Bus for single-instance dev and tests.
```

> **Scope discipline (from ROADMAP):** build single-instance first and get it rock-solid. Add the
> Redis bus only when you actually want to learn multi-instance scaling. Hide it behind the `Bus`
> interface so the hub code doesn't change when you flip from in-memory to Redis. This stays **one
> service** — do not split it further into microservices.

**Optional persistence.** The spec mentions "Redis 做消息持久化". Keep it honest with E2EE: you may
persist **ciphertext** frames (for a short replay window / store-and-forward handoff), never
plaintext. In Ghostline, durable history is Node's responsibility; if this service persists anything,
it's an ephemeral ciphertext buffer, and you should document the retention window.

---

## 12. Integrating with the Node backend

The two processes meet in exactly two places — keep it that way (the "one cut").

1. **Auth token at handshake.** Node issues a short-lived JWT (signed with a secret both share, or
   an asymmetric key Node holds the private half of). The phone presents it in the WebSocket upgrade
   (`Authorization` header or a query param for the handshake). The Go service verifies signature +
   expiry + extracts `UserID` and allowed room IDs. **The Go service trusts this token and nothing
   else the client says about its identity.**

2. **Redis as the decoupling bus.** When Node needs to push something into the realtime layer (e.g.
   a scheduled letter's `deliverAt` fires, or an offline message just arrived), Node `PUBLISH`es a
   frame to the relevant `room:`/`user:` channel. The Go service's subscriber fans it out. Node never
   calls the Go service directly; Go never calls Node's DB. Their only contract is Redis channels +
   the token format.

```
   letter deliverAt fires (node-cron in Node)
        └─ Node PUBLISH user:<id> { ciphertext }
              └─ Go subscriber -> deliver to that user's live connection (or it's offline -> Node already stored it)
```

This boundary is what makes the split *educational and safe*: you can restart either process
independently, and you can reason about each in isolation.

---

## 13. Observability

You cannot operate what you cannot see. Minimal but real:

- **Metrics (Prometheus / `expvar`):** active connections, messages in/out per second, send-buffer
  drops (the backpressure counter — watch this!), rooms count, goroutine count, redis publish
  latency. Expose `/metrics`.
- **Health endpoints:** `/healthz` (process up) and `/readyz` (Redis reachable, accepting conns) so
  Railway/your orchestrator can route correctly.
- **Structured logs (`slog`):** one connection ID per socket, threaded through both pumps. Log
  connect/disconnect/drops/errors. **Never the message body.**
- **`pprof`** behind an internal-only route for diagnosing goroutine leaks and CPU under load. A
  goroutine count that climbs monotonically = a leak; the heartbeat/teardown work in §6 is what
  keeps it flat.

---

## 14. Testing strategy

Concurrency bugs hide from manual testing. Lean on these:

- **`go test -race` everywhere, always.** The race detector is the single highest-value tool for
  this slice. Make it part of every test run; treat any race as a build failure.
- **Hub unit tests (no real sockets).** Make `Client.send` a plain channel and drive
  `register`/`unregister`/`inbound` directly; assert that the right `send` channels receive the right
  frames. The CSP design makes the hub testable *without* a network — that's a benefit of the
  pattern, not an accident.
- **Backpressure test.** Create a client whose `send` buffer you never drain; publish to its room;
  assert it gets dropped and that *other* clients still receive — i.e. the hub never blocked.
- **Lifecycle/leak test.** Open N connections, close them, assert `runtime.NumGoroutine()` returns to
  baseline. Catches teardown bugs.
- **Integration test.** Spin the real HTTP server on `httptest.Server`, dial with a real
  `websocket.Dialer`, exercise join → send → receive → disconnect across 2–3 clients.
- **Load test (later).** A small script opening thousands of connections to watch memory, goroutine
  count, and buffer-drop metrics under fan-out. This is where the heartbeat and backpressure choices
  prove themselves.

---

## 15. Project layout

A flat, honest layout — don't over-package a single service.

```
realtime/
├── cmd/realtime/main.go      # wire deps, read config, signal.NotifyContext, start server + hub
├── internal/
│   ├── hub/                  # Hub, Client, Run loop, deliver  (the CSP core)
│   ├── ws/                   # upgrade handler, auth check, origin check, read/write tuning
│   ├── protocol/            # Envelope types, encode/decode, validation
│   ├── presence/            # RWMutex presence (+ Redis-backed variant)
│   ├── bus/                  # Bus interface + redis impl + inmem impl
│   └── obs/                  # logging, metrics, health, pprof wiring
├── go.mod
└── README.md                 # how to run it locally + protocol reference
```

`internal/` keeps the packages unimportable from outside — appropriate for an app, not a library.

---

## 16. Learning path & checkpoints

Build it in this order. Each checkpoint is something you can *demonstrate* before moving on — same
vertical-slice philosophy as the rest of Ghostline.

| Step | Build | Checkpoint (you can show) |
| --- | --- | --- |
| 1 | Echo server: upgrade one connection, read a frame, write it back. | One phone/`wscat` round-trips a message. |
| 2 | One `Client` with read/writePump + buffered send. | A single client stays connected; heartbeat keeps it alive. |
| 3 | Hub `Run` loop with register/unregister/inbound; broadcast to all. | Two clients see each other's messages. |
| 4 | Rooms: join/leave + room-scoped fan-out. | Clients in room A don't see room B. |
| 5 | DMs + presence (`RWMutex`). | 1:1 message routes only to the target; presence reflects connect/disconnect. |
| 6 | Backpressure `select/default` drop + lifecycle tests with `-race`. | Slow client gets dropped; others unaffected; no goroutine leak; race clean. |
| 7 | Hardening: auth token, origin, limits, rate limit, panic recovery, graceful shutdown. | SIGTERM drains cleanly; bad tokens rejected; oversize frames rejected. |
| 8 | Observability: metrics, health, structured logs, pprof. | `/metrics` shows live connections + drop counter; logs carry conn IDs, no bodies. |
| 9 | `Bus` interface + Redis pub/sub; run **two** instances. | A user on instance A and one on instance B chat in the same room. |
| 10 | Wire to Node: JWT handshake + Redis decoupling for letters/offline. | A scheduled letter from Node pops on the recipient's live connection. |

> Tag each step in git (e.g. `realtime-v3`) so a working version is always recoverable — same
> discipline as the main roadmap.

---

## 17. Pitfalls checklist

The specific ways concurrent Go code breaks. Re-read this when something misbehaves.

- [ ] **Closing a channel twice** → panic. `close(c.send)` must happen **exactly once**, owned by the
      hub on unregister. Never close it from a pump.
- [ ] **Sending on a closed channel** → panic. After the hub closes `c.send`, nothing may send to it.
      Order matters: remove from registries *before* closing, or guard with the unregister path.
- [ ] **Blocking the hub on a slow client** → whole instance freezes. Always `select/default` in
      `deliver` (§7). The most important rule in the doc.
- [ ] **Unbuffered `send`** → every delivery blocks until the writePump picks it up; one slow socket
      stalls the hub. Buffer it.
- [ ] **Two goroutines writing one socket** → corrupt frames/panic. Exactly one writer (writePump),
      one reader (readPump). gorilla/websocket does not serialize for you.
- [ ] **Goroutine leak** → memory climbs forever. Every connection's pumps must exit on disconnect;
      verify with a leak test and a flat `pprof` goroutine count.
- [ ] **Reading a map without the lock / mutating a hub map outside the hub goroutine** → data race.
      Run `-race`. If presence needs a lock, take it on *every* access including reads.
- [ ] **`CheckOrigin: return true`** in production → CSWSH vulnerability. Allowlist origins.
- [ ] **Trusting client-sent `UserID`/`from`** → impersonation. Identity comes from the verified
      token only; overwrite `From` server-side.
- [ ] **No deadlines** → zombie connections accumulate. Set read/write deadlines + ping/pong.
- [ ] **Logging `body`** → leaks ciphertext metadata and breaks the courier invariant. Never log it.
- [ ] **Re-publishing a Redis-received frame** → infinite loop / message storm. Deliver locally only;
      publish only originals.

---

## 18. Glossary

Concept ⇄ where it lives in this build. (English doc; terms you'll meet repeatedly.)

| Term | Meaning in this service |
| --- | --- |
| **Hub** | The single goroutine that owns `clients`/`rooms` and routes every message (CSP owner). |
| **readPump / writePump** | The two goroutines per connection; one reads the socket, one writes it. |
| **Fan-out** | Delivering one inbound message to many recipients (transport fan-out, not encryption). |
| **Backpressure** | What you do when a recipient can't keep up; here, drop the slow client. |
| **CSP** | "Communicating Sequential Processes" — share state by passing messages, not by locking. |
| **Heartbeat** | Periodic ping/pong + deadlines that detect and reap dead connections. |
| **Graceful shutdown** | Drain connections cleanly on SIGTERM instead of dropping everyone abruptly. |
| **Ciphertext courier** | This service's identity: it routes opaque encrypted bytes, never plaintext. |

---

> **Where this sits in Ghostline:** this is the detailed design for [ROADMAP](../ROADMAP.md) slice 6.
> Keep it honest with [GOALS](../GOALS.md) — the service is a P2 *learning/architecture* goal, not a
> v1.0 blocker: group transport fan-out can live in Node first and be extracted here when you want to
> learn Go. And keep the one invariant sacred: **the server only ever sees ciphertext.**
