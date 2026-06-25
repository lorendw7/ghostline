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
| Invite-code registration | P0 | Already planned. |
| **Key recovery / backup** | **P0** | **The most important new goal.** After switching phones / reinstalling, be able to recover identity and read history. Approach below. |
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
| One-on-one / group real-time send & receive | P0 | Already planned. Group chat starts with per-recipient encryption fan-out. |
| Offline delivery (store-and-forward) | P0 | If the other party is offline, deliver when they come online. Already in slice 2. |
| Sent/delivered/read status | P1 | Already planned. |
| Images / voice / video | P1 | Media is client-encrypted before uploading to R2. Voice is a high-frequency must-have, prioritized over video. |
| **Quote reply / @mention** | **P1** | Without quote reply, group chats are basically unusable. |
| **Reactions** | P1 | Low cost, high return; greatly improves the everyday feel. |
| Unsend / delete message | P1 | Wanting to recall a typo is a must-have (except in Letter Mode, which is deliberately irreversible). |
| Forward message | P2 | |
| Burn-after-reading / timed disappearing | P1 | Already planned. |
| Screenshot alert | P2 | Already planned, a nice-to-have. |
| **"Typing…" indicator** | P2 | A small detail, but adds a strong "live person" feel. |

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
| App lock (PIN / biometrics) | P1 | Unlocking is required to open the app, so even if the phone is taken, the content can't be seen. |
| Metadata minimization | P1 | Record as little as possible about who contacted whom and when; logs keep no readable content. |
| Basic abuse protection | P1 | Even under the invite system, have rate limiting / brute-force protection to safeguard a small server. |
| Safety number verification | P1 | See Section I; prevents man-in-the-middle. |
| End-to-end "deleted means deleted" | P2 | Unsend / burn should truly delete on both parties' devices, not merely hide. |

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

## Explicit "Won't-do" list (holding the boundary of a friend-circle app)

Writing these down is to **prevent the project from becoming a second Signal and never shipping**:

- ❌ **Voice / video calls**: the tech stack (WebRTC + media server) is an entirely separate project; cut.
- ❌ **A true group encryption protocol (MLS / sender keys)**: per-recipient fan-out is already enough for small groups.
- ❌ **Real-time multi-device sync**: one person, one device + recovery phrase already covers the phone-switching scenario.
- ❌ **Federation / decentralization / self-hosted distribution**: one server shared among friends is enough.
- ❌ **Public registration / user discovery / stranger social**: the invite system is the founding intent; don't touch growth.
- ❌ **Bots / mini-programs / payments / social feed**: not building a platform, only communication.
- ❌ **Machine / automatic translation of message content**: under E2EE the server can't read plaintext, so translation can only be done on-device and is not a core need; i18n **only localizes the UI shell**, not user messages.
- ❌ **RTL (right-to-left, e.g. Arabic/Hebrew) layout**: the first batch of languages en/ja/zh are all LTR; don't invest in RTL adaptation for now.

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
- [ ] Kill the process, and new-message pushes still pop up.
- [ ] At any load/error point, the screen shows a **plain-language message** rather than a white screen and spinner.
- [ ] All UI copy goes through i18n resource files (English complete, no hardcoded strings); **dropping in a temporary Japanese translation switches the whole screen without breaking**, proving that ja/zh is "adding files" rather than "rewriting."

> Achieving these 9 = a complete and practical v1.0 (English edition). Japanese/Chinese translation belongs after v1.0, but **the capability to externalize copy must be in place at v1.0**. All other P2 items are post-v1.0.
