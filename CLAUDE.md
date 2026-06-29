# CLAUDE.md

Project context for Claude Code. For a detailed introduction see [README.md](./README.md).

## Project in one sentence

An end-to-end encrypted (E2EE) private messaging app, invite-only registration, running on **web and mobile from one codebase**. Two communication modes coexist: 💬 instant Chat Mode + 📜 antique-style Letter Mode (见字).

## Core principles

- **End-to-end encryption first**: private keys always stay local on the device; the server only stores ciphertext and public keys, and cannot read plaintext content.
- **Minimal storage**: the server keeps as little readable data as possible. Letter Mode content is additionally encrypted and only unlocked at delivery time.
- **Everything is ephemeral**: every Chat Mode message carries an expiry; once it passes, the message is deleted on both devices and purged from the server — no "keep forever" option, and a **hard ceiling of one month** — no message can outlive 30 days (per-conversation adjustable down to as short as 1 day; 7-day default). Letter Mode is the deliberate exception. See GOALS Section II-bis.
- **Small-scale groups only**: hard cap of **30 members** per group, enforced server-side. Group chat is E2EE via per-recipient encryption fan-out, whose per-message cost grows linearly with size — the cap is a deliberate design edge, not a temporary limit. A plaintext-hash is embedded in every per-recipient copy so members can confirm they all received the same message. See GOALS Section II-bis and ROADMAP slice 6.
- **Trust the keys, not the server**: safety-number/fingerprint verification and public-key-change (TOFU) alerts are P0 — the server is untrusted at the moment a public key is handed out, so MITM defense lives in the client.
- **Invite-only**: registration requires an invite code; public registration is not open.
- **No personal identifiers, no trackers** (P0 discipline from day one): registration collects **no phone number / email** — only an invite code, a public key, and a chosen username (a handle, not PII). The app ships **no third-party analytics / tracking SDKs**. These are "don't ever add it" rules, not later features. See GOALS Section VI.
- **One codebase → web + mobile, web-first, convenience-client caveat**: a single **Expo / React Native** codebase ships **iOS, Android, and Web** (web via `react-native-web`); **desktop = the web build (browser / PWA)** — no second codebase; an optional post-v1.0 **Tauri** native desktop app (chosen over Electron) wraps the same web build and can use the OS keychain (stronger key storage than the PWA). Develop **web-first** (fastest feedback loop), confirming mobile by each slice's end. Platform primitives (secure key storage, local DB, push, screenshot detection) live behind a thin per-platform **adapter** so app logic never branches on `Platform.OS`. **⚠️ The web client's key storage is weaker than native** (browsers have no Keychain/Keystore → WebCrypto non-extractable key in IndexedDB), so web/desktop is a **convenience client** and the UI says so. iOS ships first as a **PWA to avoid Apple's $99/year**. See ROADMAP clarification 9.
- **English-first + i18n-ready**: the app UI's primary language is **English**; from slice 1 onward all user-visible strings are **externalized into resource files**, not hardcoded. Translations are added later in the order **English → Japanese → Chinese**; adding a language is just adding a translation file, not a refactor. **User message content is never translated** (E2EE, the server can't read it either). Letter Mode CJK culture items (solar term / lunar date / seal) are presented as "English label + (original)", see section 3 of [GOALS.md](./GOALS.md).

## Main modules

- **Chat Mode (instant)**: E2EE real-time messages, group chat, multimedia, **bundled sticker/emoji packs** (custom uploads later), reactions, burn-after-reading, send-outcome status (sending / sent / failed — **no read receipts, ever**, see GOALS Section II & VI), screenshot alert.
- **Letter Mode (antique-style letters)**: antique-style stationery, seal system, sealed sending (irrevocable), delayed delivery, opening/burning a letter animations, solar-term time display, incoming-letter push.
- **Moments (quiet photo sharing, name "见影" tentative, post-Letter feature)**: a third surface built on "share and let go" — share a photo to a chosen friend / small circle with **no likes, no comments, no read receipts, no reply obligation** (the absence of engagement mechanics is the point — avoids "saw it but didn't reply" pressure). E2EE via per-recipient fan-out (same 30-cap + consistency hash as group chat), ≤1-month auto-expiry. Deliberately restrained — not an Instagram. See GOALS Section III-bis.

## Tech stack

- **Frontend (one codebase, iOS/Android/Web)**: React Native (Expo) + **`react-native-web` (Web/desktop target)** · gifted-chat · tweetnacl-js (encryption, runs identically in browser & native) · local storage **per platform behind an adapter** (native: react-native-mmkv / expo-secure-store → Keychain/Keystore; web: IndexedDB + WebCrypto non-extractable key) · expo-screen-capture (screenshot detection, **native-only**) · Zustand · react-native-reanimated (animations) · push behind an adapter (Expo Push native / Web Push API browser) · **expo-localization + i18next/react-i18next (i18n, English-first, later ja/zh)**
- **Backend**: Node.js + Express (main backend: auth/storage/letters/business logic) · Socket.io (realtime) · JWT · node-cron (Letter Mode delivery scheduled tasks)
- **Realtime fan-out service**: Go (standalone service holding long-lived connections + presence + group-chat transport fan-out, goroutine/channel concurrency, decoupled from Node via Redis pub/sub). **Only this one cut is made, not full microservices** — see ROADMAP slice 6 and clarification 5. Production-grade build guide (English, code written by the user): [docs/realtime-service.md](./docs/realtime-service.md).
- **Storage**: PostgreSQL/MongoDB (main database) · Redis/Upstash (presence, caching, Node↔Go decoupling) · Cloudflare R2 (media, stationery assets)
- **Push**: Firebase FCM

## Deployment (all free)

**Web/PWA client** static-hosted on Vercel / Netlify / Cloudflare Pages (also the iOS install path via Safari, dodging Apple's $99) · **Android** EAS APK direct to friends ($0; Google Play optional $25 one-time) · **iOS** PWA first, Apple Developer $99/yr + TestFlight/App Store only when wanted · Node main backend + Go fan-out service (two services) Railway · database Supabase · Redis Upstash (also serves Node↔Go decoupling) · files Cloudflare R2 · push FCM (native) + Web Push (browser)

## Letter (见字) data structure

```javascript
{
  content,      // encrypted content
  sealedAt,     // sealed time
  deliverAt,    // delivery time (delayed delivery)
  style,        // stationery style e.g. "xuanzhi"
  seal,         // seal pattern e.g. "plum"
  canBurn,      // whether the letter can be burned
  solarTerm,    // solar term e.g. "芒种"
  lunarDate     // lunar date e.g. "癸卯年五月初六"
}
```

## Development order (vertical slices, see [ROADMAP.md](./ROADMAP.md))

Environment → plaintext one-on-one chat (Socket.io) → identity + persistence (JWT + invite code + Supabase) → E2EE (tweetnacl) → chat polish (status/presence/media/push) → Letter Mode + animations → **Go realtime fan-out service + group chat** → hardening and release.

> Follow vertical slices: each slice connects all the way from the phone to the server. Group chat's "transport fan-out" is split into a standalone Go service (slice 6), while encryption fan-out remains on the client.

## License

AGPL-3.0 (see [LICENSE](./LICENSE)). Mind the copyleft requirements when involving others' contributions or derivatives.
