# Ghostline Development Roadmap (Learning-First · Vertical Slice · Web + Mobile)

> This roadmap replaces the "do all the backend first over a full 12 weeks" table in the README.
> Goal: **maximize what you learn**, while **having something usable — in a browser and on a phone — every week or two**.
>
> **One codebase, three targets.** A single **Expo / React Native** codebase ships **iOS, Android, and Web** (the Web target via `react-native-web`). The web build *is* the desktop client (browser / PWA). There is no second codebase — see clarification 9.
>
> **Web-first development order.** Each slice is built and verified **in the browser first** (instant hot-reload, no real device or Expo Go needed), then confirmed on mobile by the slice's end. The crypto and protocol are identical across platforms, so logic written for the web target carries straight over to mobile; only a handful of platform primitives (secure storage, local DB, push, screenshot detection) get a per-platform adapter.
>
> Core methodology: **vertical slice**. Instead of "finish all the backend first, then finish all the frontend" horizontal layering, each slice goes all the way from the client through to the server — build a narrow but complete line first, then deepen it layer by layer.

---

## Why It's Ordered This Way (a note to future me)

- **Fastest feedback = strongest learning.** You've written JS/frontend, but RN, real-time communication, and encryption are all new.
  Building **web-first** gives the tightest loop of all — a browser tab reloads in milliseconds with no device or Expo Go in the way — so each new concept lights up the instant it lands, far better than burying your head in two months of backend before any integration.
- **Motivation comes from "it works."** By the end of week 1 you should be able to send plaintext messages between **two browser tabs** (and the same screen runs on a phone). That moment of accomplishment is the fuel that carries you through the hard parts later — encryption, animation, and so on.
- **Web-first, but never web-only.** Because it's one Expo codebase, "verify in the browser" and "runs on the phone" are the same screen. Keep platform primitives behind a thin adapter (clarification 9) so the web work is never thrown away when mobile comes up at each slice's end.
- **Encryption comes after "plaintext works end to end."** Understand how messages flow first, then understand how to lock them up —
  encryption is "adding a layer" on top of an existing data flow, not something to start from.

---

## Key Technical Clarifications (read before you start, to avoid detours)

1. **Letter Mode's "unlock at delivery time" ≠ server-side decryption.**
   If this is truly end-to-end encryption, the server can **never** read the plaintext. The so-called "unlock" can only be:
   the server **holds back that blob of ciphertext and doesn't deliver it** before `deliverAt`, then pushes it to the recipient when the time comes, and the recipient's device decrypts it.
   The server is only ever a "mailbox that delivers ciphertext on a timer." The README's wording should be understood this way.

2. **Where the private key lives — and that it differs by platform.**
   - **Native (iOS/Android):** the private key goes into the iOS Keychain / Android Keystore (`expo-secure-store` wraps these for you). `react-native-mmkv` is fine for ordinary local data, but **it doesn't run in Expo Go** (it needs a native build) → in the getting-started phase use Expo Go + secure-store throughout; move to an EAS dev build only when you actually need MMKV.
   - **Web:** there is **no Keychain/Keystore in the browser**, so `expo-secure-store` is unavailable. Store the key as a **WebCrypto non-extractable `CryptoKey` in IndexedDB** — never plaintext `localStorage`. This is **weaker than the native OS keystore** (bigger XSS surface), which is exactly why the web/desktop client is a *convenience client* (clarification 9).
   - **Wrap both behind one interface** (e.g. a `secureStore.get/set` with a native impl and a web impl) so app logic never branches on platform. This adapter is set up in slice 0 and is the single place platform differences for key storage live.

3. **Push: use Expo Push first, leave FCM for later.**
   Native FCM is fiddly to configure inside Expo. Expo's built-in Push Notifications are far simpler for a small app.
   Get the "you received a letter" push working with Expo Push first, and switch to FCM only when there's a real need.
   On **Web**, Expo Push doesn't apply — use the browser **Web Push API** (service worker + VAPID), behind the same notification adapter as native. Treat web push as a slice-4/7 nicety, not a blocker.

4. **Group-chat end-to-end encryption is a hard nut — don't touch it yet.**
   Group E2EE (sender keys / the MLS protocol) is very complex. At a "few friends" scale,
   doing **per-recipient encryption fan-out** first (encrypt one copy for each group member with their own public key) is enough. Leave a real group encryption protocol for last, or skip it.
   Note the distinction between two kinds of "fan-out": **encryption fan-out** (on the client, encrypting one copy of ciphertext per person) and **transport fan-out**
   (on the server, distributing one message to multiple online members). The Go service below only handles the latter — it forwards ciphertext and **never touches encryption**.

5. **No microservices, but it's worth making one cut to experience "service splitting."**
   Microservices solve the problem of "a large team deploying independently + a single service that can't cope needing to scale on its own." You're one person, at friend-group scale,
   and going fully microservices would only burn your time on inter-service communication, deployment orchestration, and distributed debugging instead of the E2EE and real-time communication you actually want to learn.
   → Pragmatic approach: **the main backend is always a single Node service**; once the chat path runs smoothly, split just "presence + group-chat transport fan-out"
   out into **a single standalone Go service** that communicates with Node via **Redis pub/sub** (see slice 6).
   This way you **make only one cut** and genuinely experience drawing a service boundary, cross-language inter-process communication, and using Go's goroutine/channel
   for high-concurrency fan-out — getting the most learning-valuable part of microservices thinking without taking on all of its complexity.

6. **What `tweetnacl`'s `box` is.**
   `nacl.box(plaintext, nonce, recipient public key, my private key)` → ciphertext.
   The recipient opens it with `nacl.box.open(ciphertext, nonce, my public key, recipient private key)`.
   This is the real meaning of "encrypt with the recipient's public key" — it also requires your own private key.
   **What it does NOT provide**: forward secrecy (if the private key leaks, all historical messages are exposed), multi-device, deniability.
   Good enough for a friend group, but you need to know where the boundaries are — that's exactly the point of learning.

7. **English-first, but externalize strings from day one — don't wait until you want to do multilingual.**
   The app's primary language is **English**; the first batch of target languages is, in order, **English → 日本語 → 中文**.
   The key isn't "translate now" but **don't hardcode now**: every sentence on the first screen goes through i18n resource files
   (`expo-localization` reads the system language + `i18next`/`react-i18next` fetches the entries).
   This way, adding a language afterward = adding one JSON file; if you hardcode all the way through, adding a language is a full rewrite — the cost difference is enormous.
   Keep two boundaries firmly in mind: ① **only localize the UI shell, never translate user messages** (E2EE — the server can't read them either);
   ② Letter Mode's **solar terms / lunar dates / seals are cultural content to preserve, not UI strings to translate away**
   (in English, present them as "`Grain in Ear (芒种)`" — an "English label + (original)" style).

8. **Why key-based login instead of username+password — and what the recovery phrase actually is.**
   The login model is **passwordless / key-based**: identity *is* a keypair, the server holds no password, and authentication is challenge-response (server sends a nonce, client signs it with the private key, server verifies against the stored public key → JWT). New device = enter the recovery phrase = re-derive the keypair = log in. (Full flow in slice 2 / GOALS Section I.)
   **Why not the "real-world" username+password?** Mainstream apps (WeChat/WhatsApp/etc.) use phone-number+password because they want to *find people* (the identifier powers social discovery + growth) and because **their servers can read your data**, so "prove you own the account" = "let you read server-side data." Both are reversed here: this app is invite-only / anti-growth, and **the server can read nothing (E2EE)** — so a server password protects nothing (without the private key you decrypt nothing anyway). The apps that take privacy seriously already move this way: **Signal** leans on a device key + recovery PIN, **encrypted wallets** use a mnemonic and have no account password at all. Username+password is an artifact of the "server holds your data" era; once the server holds only ciphertext, "prove you possess the private key" *replaces* "tell the server your password." We keep a **username purely as a human-readable handle** for friends to add you — never as a login credential, never PII.
   **The recovery phrase = a BIP39 mnemonic, not a random digit/letter string.** 12 English words drawn from the fixed 2048-word BIP39 list (128 bits of entropy). Words (not `aB3$xK9…`) because a human must hand-copy it to paper and type it back: lower transcription error, and the last word is a **checksum** so a mistyped word is rejected immediately. Use a ready-made library (`bip39`) — don't roll your own; only the encryption core is hand-written. English wordlist first (ja/zh wordlists exist, optional later). The phrase surfaces only twice — at registration and at new-device recovery; day-to-day access is a local PIN/biometric, so users rarely face the 12 words.

9. **One codebase → Web + mobile, web-first — and how to dodge Apple's $99 at the start.**
   **Architecture:** one **Expo / React Native** codebase targets **iOS, Android, and Web** (Web via `react-native-web`). **Desktop has two tiers:** (1) for v1.0, **desktop = the web build** run in a browser or installed as a **PWA** — no second codebase; (2) post-v1.0, an optional **Tauri** native desktop app (chosen over Electron: ~3–10MB vs ~100MB, safer defaults, a bit of Rust to learn) wraps the *same* web build in the OS WebView. **The Tauri build matters for security, not just packaging:** it can store the private key in the **OS keychain** (macOS Keychain / Windows DPAPI / Linux Secret Service) via a Rust impl of the `secureStore` adapter — **stronger than the PWA's browser storage**, lifting desktop back toward native-mobile security. It ships as a downloadable binary (no hosting; talks to the same backend). Caveat: Tauri uses each OS's system WebView, so render/behaviour must be checked per target OS. (See GOALS Section IX.)
   **Develop web-first:** the browser is the fastest feedback loop (millisecond reload, no device / Expo Go). Build and verify each slice in the browser, then confirm mobile by slice end. Logic (crypto, protocol, state) is platform-agnostic and shared 1:1; only platform **primitives** get a thin per-platform adapter — **secure key storage** (clarification 2), **local DB** (IndexedDB on web / MMKV·SQLite native), **push** (Web Push / Expo Push, clarification 3), and **screenshot detection** (native-only; a no-op on web). Keep these behind interfaces so app code never does `if (Platform.OS === 'web')` scattered everywhere.
   **The Apple cost:** putting an app on an iPhone (even via TestFlight) needs the Apple Developer Program, **$99/year**. To avoid paying up front: ship iOS as a **PWA** ("Add to Home Screen" in Safari installs it with **no App Store, no fee**), or sideload to your *own* device with a free Apple personal team (7-day re-sign). Pay the $99 only when you want a polished App Store / TestFlight install. Android stays free (EAS APK straight to friends; Google Play optional $25 one-time). The web/PWA hosts free on Vercel / Netlify / Cloudflare Pages.
   **⚠️ E2EE caveat — say it honestly in-product:** the web client's key storage is weaker than native (clarification 2). Position web/desktop as a **convenience client**; the most sensitive use stays on the native mobile app, and the UI should say so rather than implying all platforms are equal.

10. **Media/message compression — a self-study track (you write the code). Compress BEFORE you encrypt, and don't blanket-prefer lossless.**
   - **Order is non-negotiable: compress → encrypt → upload.** Ciphertext is ~random and **incompressible**, so compressing after encryption does nothing. All compression runs client-side on plaintext, before the `nacl.box`.
   - **Lossless vs lossy — be honest about photographs.** Lossless (PNG / WebP-lossless) is right for **text, screenshots, line-art, logos, charts** (flat regions compress well). For actual **camera photos** lossless is often *huge* — sometimes bigger than the source — while high-quality **lossy** (WebP / AVIF at q≈80–90) is a fraction of the size at imperceptible loss. With the ≤1-month ephemeral model + free R2 tier, blanket lossless wastes bandwidth and storage. **Policy: lossy WebP/AVIF for camera photos, lossless for graphics/screenshots — choose per content type, not a blanket rule.** ("Prefer lossless *where it's actually appropriate*.")
   - **Security caveat (CRIME/BREACH family) — ties to GOALS IX padding.** Compress-then-encrypt makes ciphertext *length* depend on content compressibility, which can leak information. Low risk for a friend messenger (no attacker-mixed secrets), but it's exactly why the optional **ciphertext-length padding** (GOALS IX) must pad **after** compression. Understand the interaction rather than just bolting one on.
   - **Don't compress tiny text messages.** Per-message gzip header overhead usually costs more than it saves on short strings, and it adds the length-leak surface. Compress large blobs/media, not one-liners.
   - **Cross-platform via the media adapter (clarification 9).** Resize/transcode runs client-side: web (`createImageBitmap` / Canvas / browser WebP-AVIF encoders) and native (`expo-image-manipulator`), behind the same seam as everything else.
   - **Self-study reading list (hands-on > reading):** general lossless — DEFLATE (LZ77 + Huffman) → zstd / brotli / LZ4; image lossy — JPEG (DCT + chroma subsampling + quantization) → WebP → AVIF (AV1 intra) → JPEG-XL; image lossless — PNG (filters + DEFLATE), WebP-lossless; entropy coding — Huffman → arithmetic → ANS. **Do:** run `zstd` at several levels and measure size/time, or hand-write a toy DEFLATE — that's where it sticks.

---

## Slice Roadmap (web-first: each slice ends with "something you can verify in the browser, then on the phone")

### Slice 0 · Environment and "Hello, Web + Phone" (2–3 days)
**What to do**
- `npx create-expo-app`, then run **both targets**: `expo start --web` (opens in the browser — your primary dev loop) and Expo Go on your own phone (confirm the same screen renders).
- Stand up a minimal Express server (on your machine); the **browser** hits a `/ping` on `localhost`, and the **phone** hits the same `/ping` over the LAN via the local IP.
- **Scaffold the platform-adapter layer now** (clarification 9): create thin interfaces — at minimum `secureStore` (key storage) — with a web impl and a native impl, even if they're near-empty. Setting this up on day one is what keeps every later slice single-source.

**Verify**: a sentence coming from the backend shows up **in the browser** and on the phone, from the same code.
**What you learn**: Expo/RN project structure, the `react-native-web` target, Metro, how browser ↔ phone ↔ local server connect, and why a platform-adapter seam exists.

---

### Slice 1 · Plaintext real-time one-on-one chat (week 1) ⭐ the first high point
**What to do**
- Backend uses Socket.io to forward messages (in memory first — no database, no accounts).
- Frontend: a single chat screen (use `react-native-gifted-chat` to start). Verify **two browser tabs** first (fastest), then one tab ↔ one phone.
- **Install i18n from this very first screen**: `expo-localization` + `i18next`, all visible strings go through `t('key')`,
  with English entries in `en.json`. Spending 10 extra minutes now saves you a full rewrite later when doing Japanese/Chinese (see clarification 7).

**Verify**: two browser tabs send/receive plaintext in real time; then confirm the same works tab ↔ phone, and (when a friend joins) phone ↔ phone.
**What you learn**: the WebSocket / Socket.io event model, the "connect — event — broadcast" mental model, and that the *same* RN screen runs in the browser and on the phone.
> This is the most important week of the whole project: it proves "this thing can work" — and it does so in a browser tab within hours, not after a device-setup detour.

---

### Slice 2 · Identity and persistence (week 2)
> **Login model: key-based / passwordless (no username+password).** Your identity *is* a keypair; there is no server-side password. This is the honest E2EE model and it stays consistent with "server stores only ciphertext + public keys, collects no personal identifiers." See GOALS Section I.

**What to do**
- **Registration**: validate the invite code → pick a username (a handle for friends to find you, **not** a login credential, not PII) → the client generates the keypair and a recovery phrase (mnemonic) → upload only the **public key** + username; the private key goes through the **`secureStore` adapter** (native: `expo-secure-store` → Keychain/Keystore; web: WebCrypto non-extractable key in IndexedDB — clarification 2). The user copies down the recovery phrase.
- **Server authentication = challenge-response, not a password**: the server sends a random nonce, the client **signs it with its private key**, the server verifies against the stored public key and issues a JWT. No password is ever sent to or stored on the server.
- **Daily access**: the device stays authenticated; opening the app is gated by a **local PIN / biometrics** (an app lock that never leaves the device), not a server login.
- **New device = recovery = login (one and the same)**: entering the recovery phrase deterministically re-derives the same keypair locally, which then authenticates via the same challenge-response. (Full recovery design in GOALS Section I.)
- Connect Supabase (Postgres), store a users table (username + public key, **no password hash, no phone/email**) + a messages table.
- Offline messages: when the other party is offline, store to the database and re-deliver when they come online (store-and-forward).

**Verify**: messages aren't lost when the service restarts; a message sent while the other party is offline is received when they come online; logging in from a **second browser profile (or a phone) with only the recovery phrase** works, and the server's users table holds no password and no personal identifiers.
**What you learn**: challenge-response auth + JWT, deterministic key derivation from a mnemonic, SQL table design, invite-code logic, the offline delivery pattern.

---

### Slice 3 · End-to-end encryption (weeks 3–4) ⭐ the learning core
**What to do**
- On registration, generate `nacl.box.keyPair()` locally (`tweetnacl-js` runs identically in the browser and on the phone), upload the public key, store the private key via the **`secureStore` adapter** (clarification 2).
- Sending a message: encrypt with the recipient's public key + your own private key; the server only forwards/stores ciphertext.
- Receiving a message: decrypt locally. Switching devices = re-derive keys from the recovery phrase (slice 2); no live multi-device sync.

**Verify**: querying the messages table directly in the database shows garbled ciphertext; only the two clients (sender and recipient — easiest to check as **two browser tabs**, then confirm on phones) can restore the plaintext.
**What you learn**: public-key cryptography, key exchange, what E2EE actually guarantees — and what it does **not** (see clarification 6 above).
> Spelling out the "imperfections" clearly is what real learning looks like.

---

### Slice 4 · Polishing the chat experience (weeks 5–6)
**What to do**
- Message status: **send outcome only — sending / sent / failed → resend. No read receipts and no recipient-side "delivered" indicators** (a deliberate privacy choice — the recipient's activity is never reported back; see GOALS Section II & VI).
- Presence, unread counts: connect Upstash Redis (in the one-on-one phase, do presence in Node first — simple and sufficient;
  in slice 6 you'll migrate it, together with group-chat fan-out, into the standalone Go service, and that's when you'll grasp "why split it out").
- Image messages: **compress → encrypt → upload** to Cloudflare R2 (media is client-side encrypted before upload; compression happens *before* encryption — see clarification 10: lossy WebP/AVIF for photos, lossless for graphics; don't compress tiny text). On web, file-pick instead of camera-roll; the path is identical.
- Push: get "you have a new message" working with **Expo Push (native)** and the **Web Push API (browser: service worker + VAPID)** behind one notification adapter (clarification 3). Push payloads carry no message content (GOALS Section VI).

**Verify**: send-success / send-failed states are correct and a failed send can be resent (no read receipts appear anywhere); a push still arrives after backgrounding/closing — verify in the browser (closed tab, service-worker push) and, when set up, on the phone.
**What you learn**: the message state machine, using Redis for volatile state, object storage, the push pipeline, and **client-side media compression** (lossless vs lossy, and why you compress *before* encrypting — clarification 10).

---

### Slice 5 · Letter Mode (weeks 7–9) ⭐ product differentiation
**What to do**
- Antique-style stationery UI (start with 1–2 papers + 1–2 seals; don't build the full set right away).
- Sealing and sending (irrevocable), delayed delivery (`node-cron` delivers the ciphertext only at `deliverAt`).
- Opening-a-letter / burn-a-letter animations (`react-native-reanimated`, which also targets web — verify the animation in the browser, then check feel/perf on the phone, since web and native animation drivers differ).
- Solar term / lunar date display (find a lunar-calendar library, don't compute it yourself).

**Verify**: write a letter set to "deliver in 1 hour"; only at that time does a notification arrive on the recipient (browser Web Push and/or phone) and they can open it.
**What you learn**: scheduled tasks, animation libraries, correctly stitching "delayed delivery" together with E2EE (see clarification 1).

---

### Slice 6 · Go realtime fan-out service + group chat (weeks 10–12) ⭐ architecture & new language
> This is your "learn more" cut: the first time you split a responsibility out of the main backend **into a standalone service**,
> and write it in a new language (Go). **Make only this one cut** — don't go splitting other things off while you're at it (see clarification 5).
>
> 📘 **Production-grade build guide (in English, all code written by you yourself): [docs/realtime-service.md](docs/realtime-service.md)**
> — covers architecture, type contracts, `select` multiplexing, the Hub pattern, heartbeat/backpressure, RWMutex trade-offs, Redis horizontal scaling,
> graceful shutdown, observability, testing, a 10-step learning path, and a pitfalls checklist. Below is the outline of this slice; the details are all in that document.

**Architecture (what it looks like after the split)**
```
Phone ──Socket.io──►  Node main backend (auth / storage / Letter Mode / business logic)
                        │
                        │  Redis pub/sub (the only contact point between the two processes)
                        ▼
Phone ──WebSocket──►  Go fan-out service (holds real-time connections / presence / group-chat transport fan-out)
```
> Who holds the phone's real-time long-lived connection is a real design choice:
> to start, you can let Node keep holding the connection and have Go only do the fan-out computation behind it;
> for the advanced version, let **Go hold the WebSocket long-lived connection directly**, with Node handing "the ciphertext to send" to Go via Redis.
> Go for the advanced version — that's Go's home turf for concurrent connections, and it's where you actually learn something.

**What to do**
- Stand up a WebSocket service in Go (standard library `net/http` + `gorilla/websocket` or `nhooyr/websocket`).
- One goroutine per connection to read, one goroutine to write; use a **channel** to deliver messages to the target connection —
  this is the essence of Go's concurrency model, an entirely different mental model from Node's event loop.
- Move presence into this service; Node publishes "deliver this ciphertext blob to group X" via Redis pub/sub, and Go subscribes and fans it out to online members.
- Group chat: **encryption fan-out stays on the client** (encrypt one copy per member with their own public key, see clarification 4),
  Go only handles **transport fan-out** (distributing these ciphertexts to the online people); offline people still go through Node's store-and-forward.
- **Enforce the 30-member group cap server-side** (reject create/join beyond it). The cap exists because per-recipient fan-out makes per-message work grow linearly with group size — this is the honest edge of the design, not a temporary limit. See GOALS Section II-bis.
- **Group-message consistency hash:** since each member gets a separately encrypted copy, a malicious sender could send different members different content. Embed a hash of the **plaintext** in every per-recipient copy; receivers compare it to confirm everyone got the same message.

**Verify**: create a 3-person group and have three phones send/receive in real time; Node and Go are two independent processes (restartable separately),
decoupled via Redis; confirm in the database/logs that the Go service only ever handles ciphertext throughout.
**What you learn**: how to draw a service boundary, decoupling cross-language processes via a message queue, Go's goroutine/channel concurrency model,
and a comparison of two real-time stacks (event loop vs CSP) — something a single-language project can't give you.
> Want to go further: later you can add horizontal scaling to the Go service (multiple instances + a shared presence table in Redis),
> to feel "why the real-time layer is the first to need independent scaling" — but that's a post-v1.0 thing, don't do it now.

---

### Slice 7 · Hardening and multi-platform release (from week 13)
**What to do**
- Disappearing messages (read-and-burn); screenshot detection (`expo-screen-capture`) — **native-only**, a no-op behind the adapter on web (browsers can't detect screenshots; the UI should not pretend otherwise).
- **Web/desktop release:** build `expo export --platform web`, host the static PWA free on Vercel / Netlify / Cloudflare Pages; add a service worker + manifest so it installs to desktop and to iOS via Safari "Add to Home Screen."
- **Android release:** EAS Build → APK straight to friends ($0); Google Play is an optional $25 one-time.
- **iOS release:** start as the **PWA** (no Apple fee). Pay the Apple Developer **$99/year** for TestFlight / App Store only when you want a polished native install.
- **Desktop (optional, post-v1.0):** wrap the web build with **Tauri** for a native Win/Mac/Linux app whose `secureStore` adapter uses the OS keychain (stronger than the PWA — clarification 9 / GOALS IX). Distributed as a downloadable binary; no extra hosting.
- **Backend deployment:** the Node main backend and (if built) the Go fan-out service, tied together by a shared Redis. **Free-tier is enough for friend scale ($0)** to start (Railway + Supabase + Upstash). For long-term always-on hosting (no cold starts), the chosen host is a **single Xserver VPS in Tokyo** (developer lives in Japan → low latency, yen billing, JP support; far better spec/yen than Vultr Tokyo). Plan: **2GB/3-core** (~¥792/mo on a long term) runs ghostline alone; **6GB/4-core** (~¥1,359/mo) is the sweet spot if the box doubles as a multi-project playground. Run everything (Node + Go + Postgres + Redis) via Docker Compose, optionally behind **Coolify** (self-hosted PaaS) so new projects deploy with a push. The clients (mobile/desktop binaries) cost nothing to distribute; only the backend needs always-on hosting.

**Verify**: a few friends actually start using it as a daily tool — some on the **web/PWA**, some on the **Android app**, at least one having installed the **iOS PWA without paying Apple**.
**What you learn**: the web (PWA) *and* mobile release processes, why the same codebase ships to three targets, multi-service deployment, and the distance between a toy and a product "other people want to use."

---

## Trade-off Principles (when to cut)

- **Narrow first, complete later**: each slice does only the minimum needed to "verify one line"; don't draw 8 kinds of stationery at once in slice 5.
- **Good-enough encryption**: for a friend-group scenario, `tweetnacl box` + secure-store is enough;
  forward secrecy, multi-device, and group protocols are all "later" or "not at all."
- **Borrow ready-made wheels**: use libraries for lunar calendar, animation, and push; **only the encryption layer should be written yourself**, because that's what you most want to learn.
- **Make only one cut**: the Go fan-out service is the only standalone service. Don't get addicted to splitting just because you learned how — don't carve out auth, Letter Mode, and media each into their own service —
  that's falling into the microservices trap (see clarification 5). Everything else stays in the Node main backend.
- **The Go service can be deferred or downgraded**: it's a learning/architecture slice, not a hard gate for v1.0. If time is tight,
  group chat's transport fan-out **can perfectly well be done in Node first**, and extracted later when you want to learn Go — the product loses no features as a result.
- **Translation can be deferred, externalization cannot**: Japanese/Chinese translations are all post-v1.0, but "strings go through resource files"
  must be done from slice 1 — it's one of the few "don't do it now and the cost balloons later" decisions (see clarification 7).
- **One codebase, web-first, behind a thin platform seam**: build and verify in the browser first, but keep every platform difference (key storage, local DB, push, screenshot) behind an adapter so the work is never web-only. The cost balloons if `Platform.OS` checks leak into app logic — like i18n, the seam is cheap on day one and a rewrite later (see clarification 9).
- **The native desktop wrapper is optional but chosen = Tauri**: web/PWA *is* the desktop client for v1.0; the **Tauri** native app is a post-v1.0 upgrade (it buys OS-keychain key storage, stronger than the PWA), never a gate. Don't let "make a real desktop app" block shipping.
- **Web/desktop is a convenience client, not an equal**: its key storage is weaker than native (clarification 2). Be honest about it in-product rather than over-promising parity.
- **Tag a git tag at the end of each slice**, so a "working version" can always be found again.

---

## One-Sentence Summary

> Horizontal layering means you see nothing for two months; vertical slices mean you have something usable every week.
> One Expo codebase ships to the browser and the phone at once — so build web-first for the fastest loop, keep platform differences behind a thin seam, and the knowledge sticks best the moment it lights up in a tab *and* on a friend's phone.
