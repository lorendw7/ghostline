# Ghostline Production-Grade Checklist

> README = vision · ROADMAP = the order you build in · GOALS = complete and practical · **This doc = the engineering rigor it takes to reach industrial grade.**
>
> "Production-grade" ≠ lots of users. It means: **even if only a few friends use it, the system holds up against real-world network jitter, attackers, key leaks, server outages, bad releases, app store review, and privacy regulations.**
>
> Key insight: **production-grade is a "process," not a "feature."** For the same chat feature, the difference between the toy version and the production version is whether it has tests, monitoring, rollback, threat modeling, and disaster recovery — things users never see, but whose absence will eventually bite you.

---

## 0 · Calibrate expectations first: is your encryption "production-grade"?

Honest take: `tweetnacl box` + mnemonic recovery is **good enough** for a circle of friends, but it falls short of earning the "production-grade end-to-end encryption" label. True production-grade E2EE (Signal, WhatsApp, Matrix) has **forward secrecy** and **post-compromise security** — a single leaked private key won't expose all past/future messages. Give yourself an **encryption maturity ladder**; don't try to do it all at once, and don't fool yourself either:

| Stage | Approach | Guarantee | When to ship |
| --- | --- | --- | --- |
| L1 | `nacl.box` (static key pairs) | Server can't read plaintext | Start here (slice 3) |
| L2 | + ephemeral per-session keys + message sequence numbers for replay protection | Basic replay protection | Before v1.0 |
| **L3** | **Double Ratchet** | **forward secrecy + post-compromise security** | **When you truly want to call it "production-grade E2EE"** |
| L4 | MLS / sender keys | Efficient group forward secrecy | When scaling large group chats (may skip) |

> Pragmatic advice: **use an audited off-the-shelf library like `libsignal`; do not hand-roll Double Ratchet yourself.**
> Rolling your own crypto protocol is one of the most dangerous things you can do in production. Get L1 working first, and when you make public claims, honestly label your current level.

---

## 1 · Security hardening and threat modeling (the first pillar of production-grade)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Write a threat model document** | **P0** | Be explicit: who you defend against (a curious server admin? a phone thief? a network eavesdropper?) and who you don't (nation-state adversaries). The foundation of every security decision. |
| Key rotation mechanism | P1 | Public keys can be updated and revoked; a lost old device can have its keys invalidated. |
| Replay / tampering protection | P0 | Every message carries a sequence number + time window + authentication tag; reject duplicate and modified ciphertext. |
| Never reuse a nonce | P0 | Under NaCl, nonce reuse with the same key = encryption breaks outright. Use a counter, or random + a check. |
| Transport-layer TLS + certificate validation | P0 | Even with content already E2EE, the channel must be HTTPS/WSS, with certificate pinning. |
| Dependency vulnerability scanning | P1 | `npm audit` / Dependabot / Snyk; you must learn about CVEs in crypto libraries immediately. |
| Re-validate keys/ciphertext before persisting | P1 | The server rejects any field that looks like plaintext (defense in depth). |
| Rate limiting / brute-force protection | P0 | Throttle login, invite codes, and message sending — protect a small server from being knocked over. |
| Security response process | P1 | When a friend reports "my key leaked," have a clear revoke + rotate procedure. |
| Third-party security review (lightweight) | P2 | At minimum, get someone knowledgeable to review the crypto code once — don't build it alone behind closed doors. |

---

## 2 · Infrastructure and environments (IaC + multiple environments)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Three environments: dev / staging / prod** | **P0** | Never experiment on the production database. Keep staging isomorphic to prod and verify there before releasing. |
| Config-as-code / reproducible deployment | P1 | Write deployment steps as scripts or Railway/Render config so a new machine can be rebuilt with one click. |
| **Centralized secret/credential management** | **P0** | Database passwords, JWT secret, and R2 credentials go into secret management (encrypted environment-variable storage) and **never into git**. |
| Automated database migrations | P0 | Use a migration tool (e.g., Prisma Migrate / Drizzle) so schema changes are versioned and reversible. |
| Health check endpoint | P1 | `/health` for monitoring liveness; the load balancer uses it to remove bad instances. |
| Graceful shutdown | P1 | On restart/deploy, stop accepting new connections first, finish processing in-flight messages, then exit. |

---

## 3 · Reliability and scalability (handling real load)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Multi-instance Socket.io + Redis adapter** | P1 | When a single process can't keep up, multiple backend instances sync via Redis broadcast for horizontal scaling. |
| Connection count / backpressure management | P1 | Cap per-user connections and set limits on message queues to prevent one client from dragging down the service. |
| Idempotency | P0 | If the same message is resent multiple times, the server persists it only once (deduplicated via the client message ID). |
| Graceful degradation | P1 | If Redis goes down, messages can still be sent (degrading to not showing online status) rather than the whole thing crashing. |
| Offline message cap + expiry | P1 | For people who stay offline long-term, accumulated ciphertext needs a capacity cap and an expiry policy. |
| Load / stress testing | P2 | Before launch, simulate dozens of concurrent connections so you know where the ceiling is. |

---

## 4 · Data management (backups, migration, right to deletion)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Automated periodic backups + restore drills** | **P0** | A backup never tested = no backup. Periodically do a real restore to staging to verify. |
| Data retention and cleanup policy | P1 | Delivered-and-burned messages and expired media need a background job that truly deletes them, not just marks them. |
| User data export / account deletion | P1 | When a friend asks "delete all my data," you must be able to fully execute it (in the spirit of GDPR). |
| Media storage lifecycle | P1 | Orphaned files on R2 (message deleted but file remains) should be cleaned up periodically to control cost. |
| Schema evolution without data loss | P0 | Migrations must be forward-compatible; old clients still work during the upgrade window. |

---

## 5 · Observability (you can only fix what you can see)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Error monitoring (Sentry)** | **P0** | Report both frontend crashes and backend exceptions, with stack traces — when a friend's app crashes, you know instantly. |
| Structured logging | P0 | JSON logs, searchable; **never log any message plaintext or private keys**. |
| Key metrics / dashboards | P1 | Online users, message throughput, error rate, push success rate, latency. |
| Alerting | P1 | When the service goes down or the error rate spikes, automatically notify your phone/email. |
| Tracing | P2 | Pinpoint which step a slow request is stuck on. |
| Uptime monitoring | P1 | External liveness checks (UptimeRobot, etc.) so you're notified the moment the service goes down. |
| SLO / error budget | P2 | Set yourself a target (e.g., 99% availability) to quantify "good enough." |

---

## 6 · Quality assurance (test pyramid + static checks)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Unit tests for crypto logic** | **P0** | Encryption/decryption, key derivation, recovery phrase — a small mistake in these leaks secrets, so they must be tested thoroughly. |
| Integration tests for core flows | P1 | Register → add friend → send message → other party decrypts, run end-to-end. |
| E2E tests for critical UI | P2 | Detox/Maestro running the real-device flow of login + sending a message. |
| Static typing / Lint | P1 | TypeScript + ESLint, catching a whole class of bugs before runtime. |
| **i18n translation completeness check** | P1 | Verify in CI: no hardcoded user-visible strings, each language's key set matches `en` (a missing key fails the build), no unused keys. When a translation is missing, **fall back to English at runtime** rather than showing the key name or blank. |
| Pseudolocalization walkthrough | P2 | Run the UI through lengthened/accented pseudo-translations to catch hardcoded widths, truncation, and non-wrapping layout bugs early — cheaper than waiting for real Japanese translations to come back before fixing. |
| Test coverage threshold | P2 | Set a minimum coverage for core modules; new code isn't allowed to lower it. |
| Code review (even self-review) | P1 | Never directly merge crypto-related changes; at minimum, calmly self-review or have someone look. |

---

## 7 · CI/CD and release management (shipping versions safely)

| Goal | Priority | Notes |
| --- | --- | --- |
| **CI: run tests + build on every commit** | **P0** | GitHub Actions; no merge if tests fail. |
| EAS Build automated packaging | P1 | Pipeline the Android/iOS builds; don't click manually. |
| **OTA updates (Expo Updates)** | P1 | Changing JS without re-passing store review; small fixes can be pushed to users in seconds. |
| Staged / phased rollout | P2 | Release to one or two friends first, then go full once it's stable. |
| **Forced upgrade mechanism** | P1 | The server can require overly old clients to update (especially when the crypto protocol changes). |
| One-click rollback | P0 | When a new version breaks, you can immediately revert to the last good version (OTA rollback + backend rollback). |
| Version number + changelog | P1 | Semantic versioning, recording what changed in each version, so issues are easy to locate. |
| Database migrations in the pipeline | P1 | Run migrations automatically and safely on deploy, with a pre-migration backup. |

---

## 8 · Compliance and app stores (the hurdles you can't skip to publish)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Privacy policy + terms of service** | **P0** | Mandatory for store listing. Truthfully state what you store (ciphertext + public keys) and what you don't (plaintext). |
| **Encryption export compliance declaration** | **P0** | App Store/Google Play have export-compliance questionnaires for apps containing encryption; you must answer truthfully (most open-source crypto qualifies for an exemption, but you still must declare it). |
| Store "data safety" form | P0 | Google Data Safety / Apple Privacy Nutrition Label must be filled out accurately. |
| Age rating / content policy | P1 | The rating and compliance questionnaire for a private messaging app. |
| Store asset localization (en→ja→zh) | P2 | App Store/Play titles, descriptions, and screenshots produced per target language; ship English first, fill in Japanese/Chinese as translation progresses. The privacy policy should also be provided in the corresponding languages. |
| Account deletion entry point (store requirement) | P0 | Apple/Google now both require that account deletion can be initiated in-app. |
| Open-source license compliance (AGPL) | P1 | When pulling in third-party libraries, check license compatibility; mind copyleft obligations for derivatives / server-side operation. |

---

## 9 · Operations and incident response (keeping it alive long-term)

| Goal | Priority | Notes |
| --- | --- | --- |
| **Runbook / operations manual** | P1 | "How to restart when the service goes down," "how to rotate when a key leaks," "how to restore a backup" — written as steps so you can follow them when panicked. |
| Blameless postmortem | P2 | Record what happened, with root cause + improvements, so you don't fall into the same pit twice. |
| Cost monitoring | P1 | Free tiers have caps; alert as you approach them so you don't suddenly go into arrears and lose service. |
| Status page / notification channel for friends | P2 | Have a channel to tell users about planned maintenance or outages. |
| Data center / region selection | P2 | Choose a region close to users and compliance-friendly. |

---

## Rollout cadence: production-grade isn't done in one shot — it's hardened layer by layer

> Don't let this big checklist scare you off. **First build out the feature slices per the ROADMAP, then "harden" each slice with the P0s in this doc.**
> Three recommended hardening gates:

- **Gate A (must pass before the v1.0 release)**: threat model document · TLS + certificate pinning · no nonce reuse · rate limiting · three environments · keys not in git · automated backups + restore drills · Sentry · CI test gate · one-click rollback · privacy policy + export compliance + account deletion entry point.
  → These are P0; **missing even one means you shouldn't hand the app to friends for daily use**.

- **Gate B (filled in during the v1.x stabilization period)**: multi-instance Redis · graceful degradation · data export/deletion · metrics dashboards + alerting · integration tests · OTA updates · forced upgrade · runbook.

- **Gate C (when you truly want to claim the "production-grade E2EE" label)**: upgrade encryption to **L3 Double Ratchet (using libsignal)** · key rotation/revocation · third-party security review · automated E2E testing · SLO.

---

## One-line summary

> Features make friends want to open it; **production-grade engineering rigor keeps it from failing while you sleep, and lets you fix it when it does fail.**
> Users never see the tests, monitoring, backups, and rollback — but those are exactly what decide whether this app is a "toy" or a "service."
