# Sibling Project · Email Aggregator (learning-first, kept OUTSIDE ghostline's E2EE core)

> **Status:** planned sibling project, not part of ghostline v1.0. **Why a sibling and not a ghostline module:** an email aggregator has a *fundamentally different threat model* from ghostline (the server must read plaintext mail and hold mailbox OAuth tokens), which directly contradicts ghostline's "server reads nothing / collects no identifiers" identity. Keeping them separate preserves ghostline's privacy story and gives a genuinely independent service boundary to practice microservices on. **This doc should eventually graduate into its own git repo;** it lives here only as a staging note.

---

## Why this project exists (the learning goals)

Two things the user wants to learn, both of which this project is a natural vehicle for:

1. **A real microservices boundary** — *not* "split a small app for the sake of it" (the anti-pattern ghostline's ROADMAP clarification 5 deliberately avoids). This service is genuinely independent of the messenger: its own data, its own auth domain (mailbox OAuth), its own background workers and cron. That's a *justified* boundary, which is the right place to learn service splitting, inter-service contracts, and deployment of more than one process.
2. **OAuth2 + IMAP + mail handling** — connecting Gmail / Outlook / generic IMAP, token lifecycle, MIME parsing, and a TTL-based cleanup pipeline. All new ground relative to ghostline's E2EE/realtime focus.

> Relationship to ghostline: **none at the data level.** It may share the same Xserver VPS (separate containers, separate database) and could later reuse ghostline's key-based login *as an identity provider*, but mail data never touches ghostline's E2EE store.

---

## The core feature

- Connect **multiple email accounts** (Gmail, Outlook, generic IMAP) and present a single aggregated, organized inbox.
- **Read-and-expire:** an email pulled into this platform is **auto-deleted from *our* server one day after it is read** (a TTL + cron sweep).
- **Never deletes from the origin mailbox** — the source Gmail/Outlook copy is untouched; this platform is a transient, organizing lens, not a store of record.

---

## ⚠️ Honest threat model (this is NOT E2EE — say so plainly)

Unlike ghostline, this server **must read plaintext** to organize mail, and **must hold mailbox credentials**. State this openly in-product; do not borrow ghostline's "we can't read anything" language.

| Risk | Mitigation |
| --- | --- |
| Server reads plaintext email | Unavoidable for organizing; **minimize what is persisted** (prefer headers/metadata + on-demand body fetch over storing full bodies); aggressive **1-day-after-read TTL**. |
| Holds OAuth tokens / IMAP credentials (full-mailbox access — the crown jewels) | **Encrypt tokens at rest** (KMS / libsodium sealed box with a server-held key kept out of the DB); never log them; scope OAuth to the narrowest permission that works (read-only where possible). |
| A breach exposes a user's whole mailbox | Short token TTL + refresh; per-user encryption; the 1-day content TTL limits the standing blast radius; offer one-click disconnect that revokes tokens. |
| Provider ToS / API limits (Gmail API, OAuth verification) | Use official OAuth + APIs, respect rate limits; for a personal/friends tool, unverified-app scopes may suffice but note the user-count cap. |

> **Founding rule for this project:** store as little, as briefly, as possible; the origin mailbox is always the source of truth; we are a disposable reading lens.

---

## Architecture (where the microservices learning happens)

```
            ┌──────────────────────────────┐
  Client ──►│  API service (HTTP/auth/UI)  │
            └──────────────┬───────────────┘
                           │  queue / job (Redis or similar)
                           ▼
            ┌──────────────────────────────┐
            │  Mail-sync worker            │  ← OAuth/IMAP fetch, MIME parse
            └──────────────┬───────────────┘
                           ▼
            ┌──────────────────────────────┐
            │  Cleanup cron                │  ← delete OUR copy 1 day after read
            └──────────────────────────────┘
              (its own database; tokens encrypted at rest)
```

- **API service** — auth, serves the aggregated UI, enqueues sync jobs.
- **Mail-sync worker** — pulls from each connected mailbox (OAuth/IMAP), parses MIME, writes the minimal organized record. A natural place to feel "I/O-bound work belongs off the request path."
- **Cleanup cron** — enforces the read-and-expire rule; never touches the origin mailbox.
- These can start as one process and be split as the learning calls for it — *that decision itself is the lesson*, made on a real boundary rather than an invented one.

---

## Suggested vertical slices (same learning-first method as ghostline)

0. **One mailbox, read-only:** OAuth-connect a single Gmail (read-only scope); list subject lines in a web page. Verify in the browser.
1. **Fetch + parse:** pull message bodies on demand, parse MIME, render one email cleanly.
2. **Multi-account aggregate:** connect a second provider (Outlook/IMAP); merge into one timeline.
3. **Read-and-expire:** mark-as-read → TTL → cron deletes *our* copy after 1 day; prove the origin mailbox still has it.
4. **Split the worker out:** move fetch/parse into a separate worker process behind a queue — the deliberate microservices cut, with its own deploy.
5. **Harden:** encrypt tokens at rest, one-click disconnect + token revoke, rate-limit handling, honest privacy page.

---

## Explicit non-goals (hold the boundary)

- ❌ Not an email *client* that sends mail or replaces Gmail — a transient organizing lens only.
- ❌ Not under ghostline's E2EE banner — separate product, separate honest threat model.
- ❌ No long-term storage of full mail bodies — minimize + 1-day-after-read TTL.
- ❌ Never deletes/modifies anything in the origin mailbox.
- ❌ Don't bolt it into ghostline's repo/database long-term — graduate it to its own repo.
