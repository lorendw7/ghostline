# Ghostline

> An end-to-end encrypted private messaging app —— invite-only, built for private communication within close-knit circles of friends. **One Expo codebase runs on web and mobile** (iOS / Android / browser + PWA), developed web-first.

**Not just a private chat app, but a private *communication* app.** Two ways to express yourself, side by side:

- 💬 **Instant** —— say it as it comes, end-to-end encrypted chat
- 📜 **Letters (见字)** —— say it with intention, classical-style correspondence

---

## 📚 Docs

| Doc | Answers |
| --- | --- |
| **README** (this) | What it is, what it looks like, what it's built with |
| [ROADMAP](./ROADMAP.md) | What order to build it in — learning-first vertical slices |
| [GOALS](./GOALS.md) | What "complete & practical" means — full features + priorities + v1.0 checklist |
| [PRODUCTION](./PRODUCTION.md) | Production-grade rigor — security, reliability, observability, compliance, release |

---

## Two Core Modules

### 💬 Chat Mode (Instant)

- End-to-end encrypted real-time messaging
- Small-scale group chats (hard cap of 30 members)
- Multimedia messages (images, voice, video)
- Everything is ephemeral — every message expires (hard ceiling of 1 month; burn-after-reading is the shortest setting)
- Send-outcome status only — sending / sent / failed → resend (**no read receipts, ever**)
- Screenshot alerts (native only — browsers can't detect screenshots)

### 📜 Letter Mode (Classical Correspondence)

- Antique letter-paper UI (xuan paper, bamboo slips, brocade, plum-blossom stationery)
- Personal seal system (plum, orchid, bamboo, chrysanthemum, dragon, phoenix, etc.)
- Sealed sending (cannot be recalled once sent)
- Delayed delivery (deliver in 1 hour / tomorrow / 3 days)
- Letter-opening animation (the paper slowly unfolds)
- Burn-after-reading (with a paper-burning animation)
- Solar-term timestamps (“癸卯年 芒种” instead of a numeric time)
- Delivery push notifications (“You received a letter from XX”)

---

## Tech Stack

### Frontend (one codebase → iOS / Android / Web)

- **React Native (Expo)** + **`react-native-web`** —— single codebase, three targets; the web build is also the desktop client (browser / PWA), developed web-first
- `react-native-gifted-chat` —— chat mode UI
- `tweetnacl-js` —— end-to-end encryption (runs identically in the browser and on native)
- Local storage **per platform, behind an adapter** —— native: `react-native-mmkv` / `expo-secure-store` (Keychain/Keystore); web: IndexedDB + WebCrypto non-extractable key (weaker — see ROADMAP clarification 9)
- `expo-screen-capture` —— screenshot detection (**native only**; a no-op on web)
- `Zustand` —— state management
- `react-native-reanimated` —— letter-opening / burning animations (also targets web)
- Push behind an adapter —— Expo Push (native) / Web Push API (browser)
- `expo-localization` + `i18next` / `react-i18next` —— internationalization (English-first, ja/zh later)

### Backend

- **Node.js + Express** —— main backend: HTTP API, auth, storage, letters, business logic
- **Socket.io** —— real-time communication
- **JWT** —— authentication
- **node-cron** —— scheduled jobs (checking letter delivery times)
- **Go** (standalone real-time fan-out service) —— holds live connections, presence, and group
  transport fan-out; uses goroutines/channels for concurrency, decoupled from Node via Redis pub/sub
  (see [ROADMAP](./ROADMAP.md) slice 6)
- Minimal-storage principle (the server only stores ciphertext and public keys)

> **Why not microservices?** At friend-group scale, going full microservices only adds complexity.
> We make exactly **one cut** — extracting real-time fan-out into a Go service — to learn service
> decomposition and Go concurrency without the full overhead. It's deferrable and downgradable: group
> fan-out can stay in Node first, and be extracted only when you want to learn Go.

### Database / Storage

- **PostgreSQL** or **MongoDB** —— primary database
- **Redis (Upstash)** —— presence, unread counts, caching
- **Cloudflare R2** —— images, voice, video, stationery assets

### Push

- **Firebase FCM** —— delivery alerts + regular message push

---

## Encryption Architecture

```
A key pair is generated locally on the device at registration
The public key is uploaded to the server
The private key never leaves the device

Sending a message / writing a letter: encrypted with the recipient's public key (NaCl box — needs your private key + their public key)
The server only ever sees ciphertext and cannot read the content
```

> **On letter mode's "delayed delivery":** the server can *never* read the plaintext, so "unlock at
> delivery time" means the server simply **withholds the (still-encrypted) blob until `deliverAt`**,
> then pushes it to the recipient, whose device decrypts it. The server is just a "timed ciphertext
> mailbox," never a decryptor.

> **Crypto maturity:** we start with `tweetnacl box` (static key pairs) — fine for a circle of friends,
> but it provides **no forward secrecy / multi-device**. Upgrade path (Double Ratchet, etc.) is in
> [PRODUCTION.md](./PRODUCTION.md).

---

## Letter Data Structure

```javascript
{
  content: "encrypted content...",
  sealedAt: Date.now(),           // when it was sealed
  deliverAt: Date.now() + delay,  // delivery time
  style: "xuanzhi",               // stationery style
  seal: "plum",                   // seal design
  canBurn: true,                  // burn-after-reading enabled?
  solarTerm: "芒种",              // solar term
  lunarDate: "癸卯年五月初六"      // lunar date
}
```

---

## Languages & Internationalization (i18n)

The app's primary language is **English**. i18n is driven by **practicality**:

- **Order**: English (v1.0) → Japanese → Chinese. Translations come after v1.0, but **string
  externalization starts in slice 1**.
- **Core principle — don't hardcode now, not translate now**: every user-facing string goes through
  `expo-localization` + `i18next` resource files (`t('key')`). Adding a language is then dropping in a
  JSON file; hardcoding means a full refactor later.
- **Localize the UI chrome only, never user messages**: under E2EE the server can't read plaintext,
  so message content is not (and cannot be) auto-translated.
- **Letter-mode culture is content to preserve, not copy to translate**: solar terms / lunar dates /
  seals are CJK cultural content. The English UI shows "English label (原文)" — e.g.
  `Grain in Ear (芒种)`. Japanese natively has the 24 solar terms and the traditional calendar
  (near-zero cost); Chinese shows the original.
- **Runtime**: follow the system locale, allow manual switching in Settings, fall back to English on
  missing keys, and format dates/numbers with `Intl` per locale.

> The Go realtime fan-out service is **language-agnostic** — it only routes ciphertext, touches no
> copy, and needs no i18n.

---

## Deployment (all free tier)

| Component | Platform |
| --- | --- |
| Web / PWA client | Vercel / Netlify / Cloudflare Pages (static, also the iOS install path) |
| Backend | Railway |
| Database | Supabase (PostgreSQL) |
| Redis | Upstash |
| File storage | Cloudflare R2 |
| Push | Firebase FCM |

---

## Release Plan (one codebase → mobile + desktop)

One Expo / React Native codebase targets **iOS, Android, and Web** (the Web target via `react-native-web`). **Desktop is the Web build** — run in a browser or installed as a **PWA** — so there is no second codebase. An optional post-v1.0 **Tauri** native desktop app (chosen over Electron) wraps the same web build and can store the private key in the **OS keychain**, giving stronger key storage than the PWA. The backend needs always-on hosting (free tier is enough at friend scale; for long-term always-on use the chosen host is a single **Xserver VPS in Tokyo** — 2GB ~¥792/mo for ghostline alone, 6GB ~¥1,359/mo if it doubles as a multi-project box, Docker Compose + optionally Coolify); the mobile/desktop clients are just downloadable binaries that cost nothing to distribute.

| Platform | v1.0 (cost-free path) | Later (optional, paid) |
| --- | --- | --- |
| **Android** | EAS Build → ship the APK directly to friends, **$0** | Google Play, **$25 one-time** |
| **iOS** | **PWA** installed from Safari ("Add to Home Screen") — **no App Store, so no fee**; or sideload to your own device with a free Apple personal team (7-day re-sign) | Apple Developer **$99/year** → TestFlight → App Store, only when you want a polished native install |
| **Desktop** | **Web build / PWA**, free hosting (Vercel / Netlify / Cloudflare Pages) | **Tauri** native app (chosen over Electron — ~3–10MB, uses the OS keychain → stronger key storage than the PWA) |

> **⚠️ Web/desktop E2EE caveat:** browsers have no iOS Keychain / Android Keystore, so `expo-secure-store` is unavailable there. On Web the private key can only live in the browser (WebCrypto **non-extractable** key in IndexedDB — never plaintext localStorage), which is **weaker than the native OS keystore** (larger XSS surface). The Web/desktop client is therefore positioned as a **convenience client**; the most sensitive use is still recommended on the native mobile app. MMKV, screenshot detection, and FCM push also need Web-specific fallbacks (IndexedDB, Web Push). See ROADMAP clarification 9.

---

## Development Roadmap (vertical slices, learning-first)

Instead of "finish all the backend, then all the frontend," **every slice cuts straight through
from phone to server** — narrow but complete first, then deepen. By end of week 1 you and a friend
are chatting on two real phones.

| Slice | Focus | Verifiable when done |
| --- | --- | --- |
| 0 | Setup: Expo + minimal Express | Phone reaches local backend |
| 1 | Plaintext real-time 1:1 chat (Socket.io) | Two phones exchange messages ⭐ |
| 2 | Identity + persistence (JWT + invites + Supabase) | Offline delivery, survives restart |
| 3 | End-to-end encryption (tweetnacl + secure-store) | DB holds only ciphertext ⭐ |
| 4 | Chat polish (status / presence / media / push) | Read receipts, push delivered |
| 5 | Letter mode (stationery / sealing / delay / animation) | Timed delivery + opening ⭐ |
| 6 | Go real-time fan-out service + group chat (one cut / new language) | 3-person group in real time, two processes decoupled via Redis ⭐ |
| 7 | Hardening & release (burn / screenshot / multi-service deploy) | Friends use it daily |

> Full slice breakdown in [ROADMAP.md](./ROADMAP.md); feature priorities and the v1.0 acceptance
> checklist in [GOALS.md](./GOALS.md). Production-grade engineering in [PRODUCTION.md](./PRODUCTION.md).
> Slice 6's Go realtime service has its own production build guide: [docs/realtime-service.md](./docs/realtime-service.md).

---

## License

[AGPL-3.0](./LICENSE) —— anyone who modifies this project's code (even if only running it on a server without distributing it) must open-source their changes. Closed-source repackaging is not permitted.
