# CLAUDE.md

Project context for Claude Code. For a detailed introduction see [README.md](./README.md).

## Project in one sentence

An end-to-end encrypted (E2EE) private messaging app, invite-only registration. Two communication modes coexist: 💬 instant Chat Mode + 📜 antique-style Letter Mode (见字).

## Core principles

- **End-to-end encryption first**: private keys always stay local on the device; the server only stores ciphertext and public keys, and cannot read plaintext content.
- **Minimal storage**: the server keeps as little readable data as possible. Letter Mode content is additionally encrypted and only unlocked at delivery time.
- **Invite-only**: registration requires an invite code; public registration is not open.
- **English-first + i18n-ready**: the app UI's primary language is **English**; from slice 1 onward all user-visible strings are **externalized into resource files**, not hardcoded. Translations are added later in the order **English → Japanese → Chinese**; adding a language is just adding a translation file, not a refactor. **User message content is never translated** (E2EE, the server can't read it either). Letter Mode CJK culture items (solar term / lunar date / seal) are presented as "English label + (original)", see section 3 of [GOALS.md](./GOALS.md).

## Two main modules

- **Chat Mode (instant)**: E2EE real-time messages, group chat, multimedia, burn-after-reading, message status (sent/delivered/read), screenshot alert.
- **Letter Mode (antique-style letters)**: antique-style stationery, seal system, sealed sending (irrevocable), delayed delivery, opening/burning a letter animations, solar-term time display, incoming-letter push.

## Tech stack

- **Frontend**: React Native (Expo) · gifted-chat · tweetnacl-js (encryption) · react-native-mmkv (local encrypted storage) · expo-screen-capture (screenshot detection) · Zustand · react-native-reanimated (animations) · **expo-localization + i18next/react-i18next (i18n, English-first, later ja/zh)**
- **Backend**: Node.js + Express (main backend: auth/storage/letters/business logic) · Socket.io (realtime) · JWT · node-cron (Letter Mode delivery scheduled tasks)
- **Realtime fan-out service**: Go (standalone service holding long-lived connections + presence + group-chat transport fan-out, goroutine/channel concurrency, decoupled from Node via Redis pub/sub). **Only this one cut is made, not full microservices** — see ROADMAP slice 6 and clarification 5. Production-grade build guide (English, code written by the user): [docs/realtime-service.md](./docs/realtime-service.md).
- **Storage**: PostgreSQL/MongoDB (main database) · Redis/Upstash (presence, caching, Node↔Go decoupling) · Cloudflare R2 (media, stationery assets)
- **Push**: Firebase FCM

## Deployment (all free)

Node main backend + Go fan-out service (two services) Railway · database Supabase · Redis Upstash (also serves Node↔Go decoupling) · files Cloudflare R2 · push FCM

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
