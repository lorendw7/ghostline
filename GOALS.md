# Ghostline Goals Overview (Complete and Practical Edition)

> The README is the vision, the ROADMAP is the order of work, and this document is about **what level of completeness counts as "complete and practical."**
> There is only one criterion: **a few friends are willing to keep using it as a daily tool for the long haul, rather than deleting it after a couple of days of novelty.**

---

## What "complete and practical" actually means

The reason ordinary people abandon an encrypted chat app is almost never because "the encryption isn't strong enough" — it's because of these things:

1. **Switching phones = all chat history gone / can't get into the account** — the number-one dealbreaker for E2EE apps.
2. **Messages occasionally lost / out of order / stuck spinning without sending** — unreliability is more fatal than missing features.
3. **Having to guess "what does this button do" at every step** — no empty states, no loading indicators, incomprehensible errors.
4. **Can't find that one thing said last time** — no search, no contact management.

So "complete and practical" = **usable features + reliable + recoverable + decent UX**, and all four are indispensable.
Below, the goals are laid out across these four areas, each marked with a priority.

**Priority definitions**
- **P0 Must have**: without it, friends will delete the app after two days.
- **P1 Should have**: decides the gap between "usable" and "pleasant to use."
- **P2 Nice to have**: best if present, but cutting it doesn't affect daily use.

---

## I. Accounts and Keys (the lifeline of an E2EE app)

| Goal | Priority | Notes |
| --- | --- | --- |
| Invite-code registration | P0 | Already planned. Collects an invite code, a public key, and a username only — **no phone/email** (see Section VI). |
| **Key-based / passwordless login** | **P0** | **Identity is a keypair, not a username+password.** No server-side password at all. Day-to-day access is gated by a local PIN/biometric app lock; the server authenticates the device by **challenge-response** (it sends a nonce, the client signs it with the private key, the server verifies against the stored public key and issues a JWT). New device = enter the recovery phrase = re-derive the keypair = log in. Login flow and recovery flow are literally the same path. See ROADMAP slice 2. |
| **Key recovery / backup** | **P0** | **The most important new goal.** After switching phones / reinstalling, be able to recover identity and read history. Approach below. The recovery phrase is the single master secret (it also *is* the new-device login). |
| Safety number / public-key fingerprint verification | P1 | The two parties verify a string of fingerprints offline to confirm no one is impersonating in the middle — the final link of E2EE trust. |
| Multi-device | P2 | True multi-device sync is hard; start with "one person, one device" and revisit later. |
| Rename / avatar / bio | P1 | Otherwise the contact list is just a string of IDs and you can't tell who's who. |

### Key recovery approach (P0, must be thought through from the start)
The E2EE private key lives only on the device — lose it and nothing can be decrypted. Pragmatic options for a friend-circle app, pick one or do both:
- **Recovery phrase / mnemonic**: generate a string of recovery words at registration, which the user copies down themselves. When switching devices, entering it rebuilds the private key locally.
  (The private key can be deterministically derived from the recovery phrase, so the server never gets it.)
- **Encrypted cloud backup**: encrypt the entire private key + history with a "key derived from the recovery phrase" and store it on the server.
  The server only sees ciphertext; the user unlocks it with the recovery phrase.
> At the starting stage, at least implement the "recovery phrase" option, otherwise once slice 3 adds encryption, the first friend to switch phones is lost.

---

## II. Messaging Capabilities (Chat Mode)

| Goal | Priority | Notes |
| --- | --- | --- |
| One-on-one / group real-time send & receive | P0 | Already planned. **Small-scale groups only: hard cap of 30 members per group.** Group chat uses per-recipient encryption fan-out (one ciphertext encrypted per member with their own public key), so per-message work grows linearly with group size — the cap is a deliberate trade-off, not a temporary limit. See Section II-bis and ROADMAP slice 6. |
| Offline delivery (store-and-forward) | P0 | If the other party is offline, deliver when they come online. Already in slice 2. |
| **Send-outcome status only — NO read receipts** | P1 | Messages show the **sender's own send result only**: sending → sent (success) / failed → resend. **Read receipts are never shown, ever** — the recipient's reading activity is private and never reported back. Recipient-side "delivered to their device" indicators are likewise off, since they also leak when the other person is active; the only signal is "did *my* send succeed." This is a deliberate privacy choice, not a missing feature. See Section VI and the won't-do list. |
| Images / voice / video | P1 | Media is client-encrypted before uploading to R2. Voice is a high-frequency must-have, prioritized over video. **Image quality has two tracks by source** (ROADMAP clarification 10): casual in-app camera → lossy/small; **shared gallery / DSLR photo → a "Preserve quality / Original" mode** (full resolution, original JPEG passed through, or visually-lossless q≈95) so real photography isn't crushed — with a size cap to protect the free-tier storage. **All media has a 3-day hard expiry (II-bis), so the recipient is reminded to download/save anything they want to keep.** |
| **Quote reply / @mention** | **P1** | Without quote reply, group chats are basically unusable. |
| **Reactions** | P1 | Low cost, high return; greatly improves the everyday feel. (Note: reactions live in *Chat* only — the Moments photo surface in III-bis deliberately has none.) |
| **Stickers / emoji packs** | P1 | Start with **app-bundled sticker sets** — sending a sticker only transmits a small sticker ID, so it is near-zero-cost and clean under E2EE (no media encrypt/upload). **User-uploaded custom stickers are P2/later** (those need client-encrypt + R2 upload like any media). |
| Unsend / delete message | P1 | Wanting to recall a typo is a must-have (except in Letter Mode, which is deliberately irreversible). |
| Forward message | P2 | |
| **Global message expiry (everything is ephemeral)** | **P0** | **Every message carries an expiry time; once it passes, the message is deleted everywhere — both devices and the server.** There is no "keep forever" option. Burn-after-reading and per-conversation custom timers are stricter variants of this same mechanism. See Section II-bis. |
| Screenshot alert | P2 | Already planned, a nice-to-have. |
| **"Typing…" indicator** | P2 | A small detail, but adds a strong "live person" feel. |

---

## II-bis. Two hard product boundaries (write them down so they never drift)

These two are not "features" but **load-bearing constraints** — every other decision in Chat Mode has to respect them.

### 1. Small-scale groups only — hard cap of 30 members

- **The cap is 30 members per group, enforced server-side** (creating/joining beyond 30 is rejected).
- **Why a cap exists at all:** group E2EE here is per-recipient encryption fan-out — the sender encrypts one copy of every message for each member with that member's own public key. Work and bandwidth per message grow **linearly** with group size, all on the sender's phone. Beyond ~30 this becomes painful (a single image fans out into 30 encrypted copies), so the cap is the honest edge of the chosen design, not a placeholder for "scale later."
- This keeps us out of the group-protocol rabbit hole (MLS / sender keys are on the won't-do list) while staying genuinely end-to-end encrypted.
- **Group-message consistency:** because each member receives a *separately encrypted* copy, a malicious sender could in principle send different members different content. The sender includes a hash of the **plaintext** inside every per-recipient copy; receivers compare it so everyone can confirm they got the same message. (Mirrored in ROADMAP slice 6.)

### 2. Everything is ephemeral — every message has an expiry

- **Every message (Chat Mode) carries an expiry timestamp. When it passes, the message is deleted on both devices and purged from the server.** There is no "store forever."
- **Text messages: maximum lifetime is one month (30 days) — a hard ceiling no text message can exceed.** Within that, the expiry is **adjustable per conversation** (range 1 day → 1 month), with a 7-day default. Burn-after-reading is just the shortest setting; there is no setting for "never" and none longer than a month.
- **Media expires faster — a 3-day hard ceiling.** Images / voice / video (and Moments photos, III-bis) are auto-deleted from devices and server **within 3 days** — they're the bulk of storage, the most sensitive to leave lying around, and the heaviest on the free R2 tier. A message's attached media **never outlives 3 days even if the surrounding text lives longer** (media TTL = min(text TTL, 3 days)). **Because media is short-lived, prompt the recipient to download/save it themselves if they want to keep it** (e.g. a "saved before it's gone" hint on media). The save is local and manual — the app never keeps a permanent copy.
- **Deletion is real, not hidden:** the client drops it from local storage and the server hard-deletes the ciphertext row (or never stores past the TTL). Expiry must survive app restarts and offline periods (check on launch + a periodic sweep), so a message can't outlive its deadline just because the app was closed.
- **Interaction with key recovery (Section I):** encrypted history backup only ever contains **not-yet-expired** messages; expired content is never resurrected on a new device.
- **Letter Mode is the deliberate exception** — a letter is meant to be kept; its lifecycle (seal → deliver → optionally burn) is governed by Section III, not by this global timer.

---

## III. Letter Mode (the soul of the product, kept restrained)

| Goal | Priority | Notes |
| --- | --- | --- |
| Writing / sealing / delayed delivery | P0 | This is the fundamental difference from other chat apps and must be polished well. |
| Opening a letter / burning a letter animations | P0 | The sense of ceremony is the entire point of this mode; if the animation is crude, it has no soul. |
| Stationery / seals (start with 2–3) | P1 | Don't build the full set at once; first make 2–3 truly refined. |
| Solar term / lunar date time | P1 | Use an off-the-shelf lunar calendar library, don't compute it yourself. **Presentation under English-first:** solar terms / lunar dates are CJK cultural items; in the English UI use "English label + (original)", e.g. `Grain in Ear (芒种)`, `Start of Spring (立春)`; Japanese naturally has the twenty-four solar terms (二十四節気) (`芒種 / Bōshu`) and the old calendar, at near-zero cost; Chinese shows the original. **The cultural original is not "copy" to be translated, but content to be preserved** — don't translate "芒种" out of existence as a UI string. |
| Letter drafts | P1 | A letter is "written seriously"; you need to be able to save a half-written draft. |
| Letter collection / letter-box archive | P2 | Storing treasured letters separately has emotional value. |
| Reply quoting the original letter | P2 | The back-and-forth feel of correspondence. |

---

## III-bis. Moments — quiet, no-pressure photo sharing (third surface, name "见影" tentative)

> A third surface beside Chat and Letters, built on one design principle: **"share and let go."** A photo goes into a quiet shared space that creates **no "they saw it but didn't reply" debt**. It is deliberately stripped of every engagement mechanic — that absence *is* the feature, the same way Letter Mode's restraint is its soul.

**What it deliberately does NOT have (the point):**
- ❌ **No likes / reactions / comments** — kills social-metric pressure and one-upmanship.
- ❌ **No read receipts** — already a global rule (Section VI); nobody can see who looked.
- ❌ **No reply obligation** — a shared photo is "being present," not a message awaiting an answer. It never appears in Chat and never produces an unread badge that demands action.

**What it is:**

| Goal | Priority | Notes |
| --- | --- | --- |
| Share a photo to a chosen **friend / small circle** | P1 | Audience is picked per share (one friend or a small circle), **not broadcast to all**. Keeps it intimate, not a "social platform." **This is the prime use case for the "Preserve quality / Original" image mode** (Section II / ROADMAP clarification 10) — everyday sharing of real photography (DSLR) should keep full fidelity, not be crushed like a casual snapshot. |
| **E2EE via per-recipient fan-out** | P0 (if built) | Photos are client-encrypted before upload to R2 and fanned out per recipient public key — the *same* mechanism as group chat (II-bis), so the **30-member cap and the plaintext-consistency hash apply equally**. The server only ever stores ciphertext. |
| **≤ 3-day auto-expiry + "save it yourself" reminder** | P0 (if built) | Shared photos obey the **media** ceiling (II-bis): auto-deleted from both devices and the server **within 3 days**. No permanent gallery. Because it vanishes fast, prompt the viewer to **download/save** any photo they want to keep — the save is local and manual. |
| No engagement mechanics | P0 (if built) | No likes/comments/reactions/read-state, by design. Don't add them "because every app has them." |
| Quiet, separate surface | P1 | Its own tab; viewing is passive and private. No counters, no "X new" pressure — at most a subtle "there's something new" hint. |

> **Scope discipline:** this is a *restrained* third mode, like Letters — resist turning it into an Instagram. If a proposed addition would create comparison, obligation, or a metric, it belongs on the won't-do list. **Sequencing:** it reuses media-encryption (slice 4) and group fan-out (slice 6), so it naturally lands *after* those — a post-Letter / post-v1.0 feature, not an early slice.

---

## IV. Social and Organization (making it feel like a real app)

| Goal | Priority | Notes |
| --- | --- | --- |
| Contact list / add friends | P0 | Under the invite system, establish relationships via invite codes or mutual adds. |
| Conversation list (recent chats) | P0 | The first screen on entering the app; without it the app is simply unusable. |
| **Message search** | P1 | Local search (because the server holds ciphertext, search can only be done locally after decryption). |
| Mute / per-conversation mute | P1 | Without mute, one noisy group makes you want to delete the app. |
| Block / leave group | P1 | Basic relationship controls. |
| Pin conversation | P2 | |
| Read position / unread divider | P2 | Return to a conversation and pick up where you left off. |

---

## V. Reliability (more important than features, yet the easiest to overlook)

> This whole section is the real watershed between a "toy" and "usable daily." Every item is P0/P1.

| Goal | Priority | Notes |
| --- | --- | --- |
| **Auto-reconnect + message backfill** | **P0** | After network jitter, reconnect automatically and backfill messages missed during disconnection. |
| **Send queue + failure retry** | **P0** | Sending/send-failed must have clear states; under weak networks, no loss, resendable. |
| Message dedupe + ordering guarantee | P0 | Each message carries a client-generated unique ID + sequence number to prevent duplication and out-of-order. |
| Local-first (optimistic update) | P1 | Tapping send puts it on screen immediately (marked as sending), without waiting for server confirmation, so it feels responsive. |
| Offline-readable at launch | P1 | View history even without network (locally decrypted cache). |
| Reliable push delivery | P1 | Still receive "new message / letter arrived" after the process is killed. |

---

## VI. Privacy and Security (true to the project's founding intent, but not overdone)

| Goal | Priority | Notes |
| --- | --- | --- |
| Server stores only ciphertext + public keys | P0 | A fundamental principle of the project, already planned. |
| **No personal identifiers collected** | **P0** | Registration never asks for a phone number or email — only an invite code, a public key, and a chosen username (a handle, not PII). A "don't ever add it" discipline enforced from day one, not a later feature. |
| **No third-party trackers / analytics SDKs** | **P0** | The app ships no third-party analytics or tracking SDKs that phone home. Any diagnostics stay self-hosted (e.g. self-managed Sentry). Otherwise a single tracking SDK quietly undoes the whole privacy posture. |
| **Global message expiry (everything is ephemeral)** | **P0** | Every message has an expiry; past it, it is deleted on both devices and purged from the server. Hard ceiling of one month (30 days) — no message can live longer; per-conversation adjustable down from there (1 day → 1 month, 7-day default), no "never" option. Full design in Section II-bis. Privacy payoff: a seized phone or server holds at most one expiry window of content. |
| **Safety number / public-key fingerprint verification** | **P0** (was P1) | The weakest link in this E2EE design is the moment a public key is uploaded — a malicious server could hand out a key it controls (man-in-the-middle). Without offline fingerprint comparison (scan a QR / read out a number string), the whole encryption chain is only as trustworthy as the server. Promoted to P0. See Section I. |
| **Public-key-change alert (TOFU)** | **P0** | Trust-on-first-use: pin a contact's key on first contact; if it later changes (they switched phones — or someone swapped it), show a loud "⚠️ this contact's safety number changed" warning before the next message goes out. Without this, a server can rotate in a key it controls and you'd never notice. |
| App lock (PIN / biometrics) | P1 | Unlocking is required to open the app, so even if the phone is taken, the content can't be seen. |
| **Encrypted local storage at rest** | P1 | The on-device message cache and the private key must be encrypted at rest (SQLCipher / encrypted MMKV + `expo-secure-store` for the key), so an unlocked, seized phone doesn't expose plaintext history. Pairs with App lock. |
| **Push payloads carry no content** | P1 | Pushes say only "you have a new message / a letter arrived" — never message text or sender-revealing detail in the payload, since the push provider (Expo/FCM) can read it. |
| **Backup-passphrase brute-force resistance** | P1 | The encrypted key/history backup (Section I) is unlocked by a recovery phrase. Use a high-entropy mnemonic (e.g. BIP39 12 words) + a slow KDF (Argon2id) so a server holding the backup ciphertext can't offline-guess the passphrase. Write the threat model down. |
| **No read / activity receipts** | P1 | The recipient's reading and presence is never signalled to the sender — no read receipts, no "delivered to their device" ticks. The sender only ever sees their own send outcome (sent / failed). Reduces the activity metadata the app produces in the first place. See Section II. |
| Metadata minimization | P1 | Record as little as possible about who contacted whom and when; logs keep no readable content. **Known unavoidable leak:** group membership (who is in which group) reveals social graph; we accept it at friend-circle scale but list it honestly here rather than pretending it's hidden. |
| Certificate pinning (transport) | P1 | Pin the server's TLS certificate so a hostile network / middlebox can't silently downgrade or MITM the transport layer beneath the E2EE. |
| Basic abuse protection | P1 | Even under the invite system, have rate limiting / brute-force protection to safeguard a small server. |
| End-to-end "deleted means deleted" | P2 | Unsend / burn should truly delete on both parties' devices, not merely hide. Subsumed by the global-expiry mechanism above. |

### VI-bis. What the server actually stores (the honest inventory)

> A plain-language answer to "does the server keep my data?" — useful for explaining the privacy model to a friend. **Two rules hold across every row: the server only ever holds ciphertext + public data (it can decrypt nothing), and chat content is purged within a month.** What the server retains *long-term* is a tiny, non-readable, non-PII core.

| What | Stored on server for how long | Readable by the server? |
| --- | --- | --- |
| **Chat messages** | **Not long-term** — auto-purged at expiry; hard ceiling 1 month, often much less (down to burn-on-read). Removed once delivered + expired. | No — ciphertext only. |
| **Media** (images / voice / video) | **Even shorter than chat — a 3-day hard ceiling** (II-bis); stored on R2, then purged. Users are reminded to download/save anything they want to keep. | No — client-encrypted before upload. |
| **Account record** (username + public key) | **Long-term** — kept while the account exists. | Public key is public by design; username is a handle, **not PII**. |
| **Invite codes** | Long-term. | No sensitive content. |
| **📜 Letters (Letter Mode)** | **Long-term by design — the deliberate exception to ephemerality** (a letter is meant to be kept). Delayed letters are held as ciphertext until `deliverAt`. | No — ciphertext only; "unlock at delivery" = the server releases the blob on a timer, it never decrypts (clarification 1). |
| **Encrypted key/history backup** (optional, Section I) | Long-term, **as ciphertext**. | No — encrypted under a recovery-phrase-derived key; the server never sees the key. |
| **Minimal metadata** (routing: who↔who, timestamps, group membership) | Minimized; logs keep no readable content. | This is the **known unavoidable leak** — see "Metadata minimization" above. Content stays unreadable; the *fact* of contact is what a relay server inevitably sees. |

> **One-line summary:** the server is a **mailbox that holds only ciphertext and burns chat content within a month**; what it keeps long-term is just the unreadable minimum (public key + username + invite code) plus deliberately-kept letters (still ciphertext).

---

## VII. Experience Polish (decides "comfortable to use")

| Goal | Priority | Notes |
| --- | --- | --- |
| Onboarding / first-launch flow | P1 | The first open teaches the user to "enter invite code → copy down recovery phrase → start." |
| Empty state / loading state / error state | P1 | Without these, any load is just a white screen + spinner. |
| Dark mode | P1 | Private chats are mostly used at night. |
| Settings page | P1 | Notifications, privacy, app lock, about, and backup-recovery entry all live here. |
| Consistent design language | P1 | A unified visual of classical + modern, not a patchwork. |
| **English-first + string externalization (i18n-ready)** | **P0** | The app's primary language is English; from slice 1, all user-visible strings go through i18n resource files (`expo-localization` + `i18next`), **not hardcoded**. This is P0 not because we need multiple languages now, but because **the cost of adding translations later depends on whether they were externalized now** — externalized, adding a language = adding a JSON; hardcoded, it's a full rewrite. |
| Japanese / Chinese translation | P1 | After v1.0, add in the order **English → 日本語 → 中文**. Auto-select following the system language, with a manual switch on the settings page; missing keys fall back to English. Format dates/numbers per locale with `Intl`; leave plural rules to the i18n library (English has singular/plural, ja/zh don't). |
| **Responsive web / desktop layout** | P1 | The same Expo codebase renders in a browser (desktop & mobile widths) and as a phone app. Layouts must adapt to wide desktop viewports (not just a stretched phone column), and platform-only affordances degrade gracefully on web (e.g. screenshot alerts are a native-only no-op). See ROADMAP clarification 9. |
| Accessibility / font-size adaptation | P2 | Don't hardcode widths for i18n copy — German/Japanese lengths vary widely, so the layout must adapt. |
| Haptic feedback (haptics) | P2 | Vibration on sealing and burning a letter maxes out the sense of ceremony. |

---

## VIII. Engineering and Operations (keeping it alive long-term)

| Goal | Priority | Notes |
| --- | --- | --- |
| Error monitoring (e.g. Sentry) | P1 | When a friend reports "it crashed," you can find the cause. |
| Automated tests for critical paths | P1 | Encryption, sending, and recovery — these core chains need test coverage as a backstop. |
| Invite codes / simple admin backend | P2 | Generate invite codes, view registration counts — a minimal backend is enough. |
| Database backup strategy | P1 | Even just periodic exports — don't let one failure wipe out everyone's ciphertext. |
| CI (run tests/build on commit) | P2 | |
| Go real-time fan-out service (split out) | P2 | **A learning/architecture goal, not a v1.0 threshold.** Split presence + group-chat transport fan-out into a standalone Go service (ROADMAP slice 6). Can be deferred, can be downgraded — group-chat fan-out can first be done in Node and still reach v1.0. |

---

## IX. Post-v1.0 optional privacy / learning experiments (not on the main line)

> These are **not v1.0 gates and not part of the core slices.** They're "do it after v1.0 ships, for the learning and the privacy bump." Each is self-contained and can be skipped with zero feature loss. Listed roughly cheapest-first.

> (Two items that started here — "no identifiers at registration" and "no third-party trackers" — were promoted to **P0 core discipline** in Section VI, because they are "don't ever add it" rules to hold from day one, not things to bolt on after v1.0.)

| Experiment | Cost | What it buys | Notes |
| --- | --- | --- | --- |
| **Tauri native desktop client** | med | Wraps the existing `react-native-web` build in a tiny (~3–10MB) native window (Win/Mac/Linux) via the OS WebView + a Rust backend. **Crucially, it can store the private key in the OS keychain** (macOS Keychain / Windows Credential Manager-DPAPI / Linux Secret Service) — **stronger than the PWA's browser storage**, pulling desktop back up toward native-mobile security. | **Chosen over Electron** (1/10th the size, safer defaults, a bit of Rust to learn). Same codebase — only the `secureStore` adapter gets a desktop (Rust keychain) impl, a seam already reserved in slice 0. Caveat: Tauri uses the per-OS system WebView, so render/behaviour must be checked on each target OS. Distributed as a downloadable binary — needs no hosting (only the shared backend). |
| **Ciphertext length padding** | low | Pad every message to fixed-size buckets before encrypting, so ciphertext size doesn't leak message length. **Pad *after* compression** (compress → pad → encrypt) — see ROADMAP clarification 10 (CRIME/BREACH length-leak interaction). | Cheap, real metadata win; pairs naturally with E2EE. |
| **Account / data self-deletion** | low | A user can wipe their account + all their server-side ciphertext on demand ("right to be forgotten"). | Good privacy hygiene and simple to build. |
| **Duress / panic unlock** | low–med | A second "duress" PIN that opens an empty or decoy state, or instantly wipes local data, if forced to unlock. | High emotional value for a privacy app; client-only. |
| **Tor hidden service (.onion)** | med | The server runs as a Tor onion service; clients connect over Tor, hiding **user IP / location** from the server and the network. | The lightest taste of "anonymity network" without rewriting any protocol — just a deployment + transport change. A genuine learning project. |

> Heavier anonymity (mix networks, sealed sender, PIR) stays on the won't-do list below — those rewrite the delivery protocol and trade away the realtime feel for a threat model a friend-circle app doesn't need.

---

## Explicit "Won't-do" list (holding the boundary of a friend-circle app)

Writing these down is to **prevent the project from becoming a second Signal and never shipping**:

- ❌ **Voice / video calls**: the tech stack (WebRTC + media server) is an entirely separate project; cut.
- ❌ **A true group encryption protocol (MLS / sender keys)**: per-recipient fan-out is already enough for small groups.
- ❌ **Real-time multi-device sync**: one person, one device + recovery phrase already covers the phone-switching scenario.
- ❌ **Federation / decentralization / self-hosted distribution**: one server shared among friends is enough.
- ❌ **Heavy anonymity networks (mix networks, sealed sender, PIR)**: these rewrite the delivery protocol and trade away realtime feel for a global-passive-adversary threat model a friend-circle app doesn't need. (A *lightweight* Tor onion-service deployment is allowed as a post-v1.0 experiment — see Section IX — because it changes only transport, not the protocol.)
- ❌ **Public registration / user discovery / stranger social**: the invite system is the founding intent; don't touch growth.
- ❌ **Bots / mini-programs / payments / social feed**: not building a platform, only communication.
- ❌ **Engagement mechanics on Moments (III-bis): likes, comments, public follower feed, view counts, "streaks"**: the no-pressure quiet-sharing model *is* the feature; any metric or comparison turns it into an Instagram and is explicitly off-limits.
- ❌ **Machine / automatic translation of message content**: under E2EE the server can't read plaintext, so translation can only be done on-device and is not a core need; i18n **only localizes the UI shell**, not user messages.
- ❌ **RTL (right-to-left, e.g. Arabic/Hebrew) layout**: the first batch of languages en/ja/zh are all LTR; don't invest in RTL adaptation for now.
- ❌ **Read receipts / "delivered to recipient" indicators**: the recipient's reading and online activity is never reported back to the sender. The only status shown is the sender's own send outcome (sent / failed). A deliberate privacy choice — see Section II and VI.

> Whenever you want to add a new feature, first ask: **"Will my few friends delete the app because it's missing?"** If not, it goes to P2 or won't-do.

---

## "Complete and practical" acceptance checklist (achieving it counts as v1.0)

When every line below is true, the app goes from a "learning project" to "a genuinely usable product":

- [ ] A friend can register with an invite code, copy down the recovery phrase, add you as a friend, and send the first encrypted message.
- [ ] He switches to a new phone, logs back in with the recovery phrase, and **can still read previous messages**.
- [ ] Sending under a weak network has clear "sending/failed/resend" states, and **ultimately no loss, no duplication, no out-of-order**.
- [ ] In groups you can quote-reply, @, and drop a reaction.
- [ ] He can write a letter, set it to "deliver tomorrow," and the next day you **receive a push and open it**.
- [ ] He can search for that thing said last week, mute a noisy group, and add a lock to the app.
- [ ] Kill the process, and new-message pushes still pop up (and the push payload contains no message content).
- [ ] A message past its expiry is gone from both phones **and** the server — verified by querying the database directly — and stays gone across app restarts.
- [ ] A group is capped at 30 members (the 31st is rejected), and three phones in one group all confirm the same group-message consistency hash.
- [ ] When a contact's public key changes, the other side sees a safety-number-changed warning before the next message is sent.
- [ ] The same codebase runs as a **desktop/mobile web app (installable PWA, no Apple fee)** and as a native mobile app, with the web client honestly marked as a convenience client (weaker key storage).
- [ ] At any load/error point, the screen shows a **plain-language message** rather than a white screen and spinner.
- [ ] All UI copy goes through i18n resource files (English complete, no hardcoded strings); **dropping in a temporary Japanese translation switches the whole screen without breaking**, proving that ja/zh is "adding files" rather than "rewriting."

> Achieving these 9 = a complete and practical v1.0 (English edition). Japanese/Chinese translation belongs after v1.0, but **the capability to externalize copy must be in place at v1.0**. All other P2 items are post-v1.0.
