# Cloudflare-Only Mailbox Feasibility and Zero-Charge Limits

**Date:** 2026-10-13
**Ticket:** [Research Cloudflare-only mailbox feasibility and zero-charge limits (issue #3)](https://github.com/tmih06/cf-mailbox/issues/3), under the map [Chart a free-tier Cloudflare personal unified mailbox (issue #1)](https://github.com/tmih06/cf-mailbox/issues/1)
**Baseline:** [Research Thunderbird's mailbox model and feature surface (issue #2)](https://github.com/tmih06/cf-mailbox/blob/research/thunderbird-mailbox-model/docs/research/thunderbird-mailbox-model.md) at commit `e5ef5e4` — capabilities labeled A1–A7 (approved requirements) vs B8–B19 (undecided extras).
**Constraint under test:** Cloudflare-hosted application compute and storage only; no paid Cloudflare plan; no usage charges. External providers remain external. Owner-side OAuth registrations and secrets are acceptable; existing domain/mailbox subscriptions are out of the hosting budget.

**Method.** Official Cloudflare docs (developers.cloudflare.com), Google Workspace/Gmail developer docs, and Microsoft Learn/Graph docs, all fetched 2026-10-13. Each row cites its owning source. Anything I could not confirm from a current primary source is listed under [Limits and unknowns](#10-limits-and-unknowns).

---

## 1. Executive summary
A Cloudflare-only unified mailbox connecting **existing** Gmail / Outlook.com / M365 / generic-IMAP mailboxes has **documented API paths for every required capability on the Workers Free plan**. "Documented" here means the API surface, scopes, and Free-plan quotas exist in primary sources — it is **not** runtime proof: no end-to-end flow (IMAP session, MIME parse, provider auth) was exercised, OAuth registrations and tenant consent gates are per-owner variables, and a single-owner workload is assumed but not measured. Three sharp boundaries:

1. **Outbound mail cannot come from Cloudflare Email Sending.** Email Sending is available only on the **Workers Paid plan** for arbitrary recipients (3,000/mo included, then $0.35/1,000). The only free sends are to *verified destination addresses* on the owner's own account — i.e., the app replying to the owner, not the owner replying as their mailbox. Sending "as the original identity" therefore routes through each provider's own submission path (Gmail API `messages.send`, Microsoft Graph `sendMail`, or authenticated SMTP over `connect()` to ports 587/465), never through Cloudflare's service. Cloudflare cannot send `From: user@gmail.com` — sending domains must be onboarded zones on the owner's account.
2. **R2 is a documented charge risk, not a provable-free service.** Activation is a dashboard checkout ("Complete the checkout flow to add an R2 subscription"); community reports and a Cloudflare bug-bounty report state a payment method is mandatory at activation. Once active, overage is **metered**: storage beyond 10 GB-month and operations beyond the free allowances are automatically billable with **no documented hard cap**. Budget alerts notify; they do not stop spend. The owner's constraint is *no paid plan / no usage charges* — R2's documented auto-overage means the app cannot guarantee zero charge once R2 exists; this is a requirement conflict needing an explicit owner decision. A D1-only storage design has no metered surface at all but is smaller and slower.
3. **Always-on IMAP IDLE is not proven viable on Workers.** Docs allow unlimited HTTP wall-time while a client stays connected, cron handlers get 15-minute wall time, and a Durable-Object TCP socket keeps running even after the DO stops accruing keep-alive duration — so opening and briefly holding an IDLE session is *possible*. What is **not established** is unattended, always-on IDLE reliability at zero cost: sockets are handler-scoped (no global-scope pooling), nothing guarantees an idle session survives between invocations, and cron (5 triggers max, 1-minute granularity) cannot hold one. Polling via cron + delta sync (`history.list` for Gmail, `/delta` for Graph, `UID`/`NOOP` for IMAP) is the conservative, documented candidate.

Everything else in the required set (attachments, search, two-way flags/folders/deletes, old-history access) has documented paths on Free with provider-API or provider-IMAP plus bounded local indexes — subject to the CPU, request, and storage ceilings quantified below and to the "documented ≠ runtime-proven" caveat above.

---

## 2. Verdict table by capability and provider

Verdicts: **Feasible** (documented API/quota path on Workers Free with no charge surface — *not* exercised end-to-end), **Conditionally feasible** (documented path gated on an unverified runtime property, an owner decision, or a per-tenant/per-account setting), **Not proven** (plausible but unverified), **Infeasible** (hard blocker under the zero-charge constraint). Every "Feasible" below means *documented and no charge surface*, never runtime-proven.

| Capability (from Thunderbird A-list) | Gmail | Outlook.com | M365 | Generic IMAP/SMTP + custom domain |
|---|---|---|---|---|
| A1 Connect multiple mailboxes | Feasible via IMAP `connect()` or Gmail API | Feasible via IMAP `connect()` or Graph | Conditionally feasible — Graph works; IMAP needs tenant SMTP-AUTH/OAuth gates (see §8) | Feasible via IMAP `connect()` (see §4) |
| A2 Read mail (headers + body on demand) | Feasible — `messages.list`/`get` or IMAP FETCH | Feasible — Graph `/messages` or IMAP | Feasible — Graph/IMAP | Feasible — IMAP FETCH over `connect()` |
| A3 Send/reply as original address | Feasible — `messages.send` / SMTP 587 XOAUTH2 or app password | Feasible — Graph `sendMail` or `smtp-mail.outlook.com:587` | Feasible — Graph `sendMail` (preferred); SMTP AUTH must be enabled by tenant admin | Feasible — SMTP submission on 587/465 |
| A4 Attachments in/out | Conditionally feasible — MIME `raw` + `attachments.get` documented; large-message CPU/limits not exercised (see §4 CPU realism) | Conditionally feasible — attachments or MIME `$value`; Exchange Online limits apply; large-MIME path not exercised | Conditionally feasible — same | Conditionally feasible — MIME over IMAP/SMTP; provider size caps + Workers CPU apply |
| A5 Search | Feasible — `messages.list?q=` (Gmail syntax) + local FTS | Feasible — `$search` (KQL) on `me/messages`, ≤1,000 results | Feasible — `$search` | Feasible — IMAP `SEARCH`/`UID SEARCH` + local FTS |
| A6 Two-way read/label/move/delete | Feasible — `messages.modify`/`batchModify` or IMAP flags+MOVE | Feasible — `message` PATCH (`isRead`, `categories`) + dedicated `POST …/messages/{id}/move` | Feasible | Feasible — IMAP STORE/MOVE/EXPUNGE |
| A7 Old-history access via bounded cache | Feasible — full-sync then `history.list` deltas (history ~≥1 wk; 404 → re-sync) | Feasible — per-folder `delta` (deltaLink) | Feasible | Feasible — IMAP `UID`/`FETCH` windows |
| Push / near-real-time new mail | Not proven free — `watch` requires Cloud Pub/Sub (external Google infra); polling is the documented alternative | Feasible — Graph webhook to a Worker URL, renew ≤7 d | Feasible — same; admin consent may apply | Not proven — always-on IDLE not established on Workers (see §4); poll instead |

**Hard blockers (vs. soft risks):**
- *Email Sending to arbitrary recipients*: **infeasible** on Free — documented "Available on Workers Paid plan." Hard plan gate.
- *Always-on IMAP IDLE*: **not proven** — sockets open fine inside handlers and durations are generous (unlimited HTTP wall-time while a client is connected, 15-min cron wall-time, DO sockets outliving DO keep-alive), but unattended reliability and free capacity are unestablished. Polling is the conservative candidate.
- *R2*: **charge-risk, owner-gated** — activation is a checkout; overage is auto-billed and no hard cap is documented. Owner decision required; "payment method mandatory" is strong external evidence pending provisioning — the owner's prohibition is paid *usage*, not a card per se.
- *Outbound TCP port 25*: **infeasible** — `connect()` refuses port 25 to SMTP servers. 587/465/993 unaffected.
- *Inbound TCP listeners*: **infeasible** — not supported; inbound mail arrives only via Email Routing MX or provider pull.

---

## 3. Cloudflare Email Service vs. legacy Email Routing / Email Workers

Cloudflare has folded the old "Email Workers" docs into **Email Service**, which now bundles two features:

| Feature | What it is | Plan | Sender/recipient constraints | Source |
|---|---|---|---|---|
| **Email Routing** (legacy `email()` handler + `message.forward/reply`) | Inbound MX for a Cloudflare-DNS zone; routes to verified external addresses or a Worker `email()` handler | Free + Paid; inbound unlimited | Destination addresses must be **verified** in the account (200/account); routing rules 200/domain; inbound message ≤25 MiB; Worker handler subject to Free CPU limits | [Email Service](https://developers.cloudflare.com/email-service/), [Pricing](https://developers.cloudflare.com/email-service/platform/pricing/), [Limits](https://developers.cloudflare.com/email-service/platform/limits/), [Route emails](https://developers.cloudflare.com/email-service/get-started/route-emails/) |
| **Email Sending** (`send_email` binding, REST API, or SMTP submission `smtp.mx.cloudflare.net:465`) | Outbound transactional email API | **Workers Paid only** for arbitrary recipients (3,000/mo included, then $0.35/1k); **Free** only for sends to verified destination addresses | `from` must be an **onboarded domain** on the account (SPF/DKIM/DMARC on `cf-bounce` subdomain); message ≤5 MiB (25 MiB for verified-destination sends); 50 recipients; transactional-only policy | [Pricing](https://developers.cloudflare.com/email-service/platform/pricing/), [Limits](https://developers.cloudflare.com/email-service/platform/limits/), [Send emails](https://developers.cloudflare.com/email-service/get-started/send-emails/), [FAQ](https://developers.cloudflare.com/email-service/reference/faq/) |

Consequences for this app:

- **No spoofing Gmail/Outlook identities.** Email Sending requires the `from` domain to be onboarded to the owner's Cloudflare account; `gmail.com`/`outlook.com` can never be onboarded. Sending as the original identity is exclusively a provider-side operation. Cloudflare Email Sending is *optional*, never required, for A3.
- **Email Routing is inbound-only plumbing.** It requires the domain's MX to point at Cloudflare (zone on Cloudflare DNS). It **cannot coexist** as primary MX with an existing hosted mailbox on the same addresses — it replaces where inbound mail lands. Per-provider inboxes stay at the provider; Email Routing is only relevant if the owner wants app-facing addresses (e.g., `mailbox@their-domain`) or free inbound→Worker ingest on a dedicated domain/subdomain. Docs allow Email Sending/Routing on **subdomains** within a zone.
- `message.reply()` inside an `email()` handler is part of Routing (free) but restricted: only to the original sender, sender domain = the receiving domain, valid DMARC, once per event, ≤100 `References` entries. It is an auto-reply primitive, not a general mail client send path.
- Free-plan `email()` Workers share Free CPU/memory limits; heavy MIME parses can exceed them (`EXCEEDED_CPU` documented).

---

## 4. Generic connectivity: Workers `connect()` TCP sockets

Source: [TCP sockets](https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/) (updated 2026-06-19) and [Workers limits](https://developers.cloudflare.com/workers/platform/limits/) (updated 2026-10-12).

- **`connect()` supports IMAP/SMTP reachability.** `secureTransport` options: `off`, `on` (TLS), `starttls` (upgrade via `startTls()`). That covers IMAP `993` (implicit TLS), `143`+STARTTLS, SMTP `465` (implicit) and `587` (STARTTLS) — the standard provider ports.
- **Blocked destinations:** outbound TCP to **port 25** is prohibited (documented error `Connections to port 25 are prohibited`); sockets to **Cloudflare IP ranges**, `localhost`, and **private network IPs** are disallowed. This is port-25-specific — do not generalize to 587/465.
- **Inbound TCP is not supported** ("not possible to make an inbound TCP connection to your Worker").
- **Sockets are handler-scoped:** cannot be created in global scope/shared across requests; must originate inside `fetch()`, `scheduled()`, `queue()`, or DO `alarm()`. Each open socket counts toward the **6 simultaneous open connections per invocation**.
- **Lifetime:** no per-invocation wall-clock limit for HTTP-triggered Work while the client is connected, but sockets are tied to an invocation; there is no worker-level persistent connection pool. A Durable Object socket keeps the DO billed as active for up to 15 min per connection (duration charges apply beyond free DO allowance).
- **IMAP IDLE conclusion:** nothing in the docs forbids an IDLE session — `connect()` works inside handlers, HTTP invocations have no wall-time limit while the client is connected, cron handlers get 15-minute wall time, and a DO-held socket keeps running after the DO stops accruing keep-alive duration. What is **not established** is unattended, always-on IDLE at zero cost: sockets are handler-scoped with no global pooling, nothing guarantees a socket survives between invocations or that an idle session isn't reaped, and cron can't hold one. Polling (`history.list`/`delta`/`UID SEARCH` + `NOOP`) is the documented substitute; Gmail's own cap (15 concurrent IMAP connections per account) makes per-folder IDLE impractical anyway.

**CPU realism for MIME:** Workers Free allows **10 ms CPU per invocation** (HTTP and cron). `postal-mime` is the doc-recommended parser and runs in Workers (Email Worker examples use it), but parsing multi-MB raw MIME inside the *invocation* is at risk of exceeding 10 ms. Mitigation: stream `message.raw`/`$value` bodies to storage, parse incrementally, or fetch only headers for list views — all feasible patterns, but large-MIME parse latency is a *risk*, not a number I could verify.

---

## 5. Workers Free ceilings that bound the design

[Workers limits](https://developers.cloudflare.com/workers/platform/limits/) (Free column):

| Limit | Free value | Relevance |
|---|---|---|
| Requests | 100,000/day (error 1027 after; fail-open/closed configurable) | Ample for a single-owner app + polling |
| CPU time | 10 ms/invocation (HTTP & cron) | MIME parsing, crypto, token refresh must be tight |
| Memory | 128 MB per isolate | Multi-MB MIME: stream, don't buffer |
| Subrequests | 50/invocation | Pages of provider-API calls per sync cycle are bounded |
| Simultaneous open connections | 6 per invocation (establishing phase) | Caps parallel IMAP/API calls per request |
| Cron Triggers | 5 triggers/account; 15 min wall clock; 10 ms CPU each; min interval 1 min | Bounded sync cadence |
| Worker size / startup | 64 MiB / 1 s | Fine |
| Static assets | 20,000 files, 25 MiB each, requests **free and unlimited** | Host the SPA free |
| Environment variables/secrets | 64/Worker, 5 KB each | Enough for OAuth client secrets + refresh tokens if stored in D1/KV, not env |

**Breach behavior on Free:** requests beyond 100k/day → hard error 1027; CPU beyond 10 ms → error 1102; KV/D1 quota exhaustion → operation errors. These are **hard stops**, not charges — the Free plan has no billing meter attached. [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/) confirms: "By default, users have access to the Workers Free plan" and Workers/KV/Hyperdrive free-tier limits fail closed ("further operations of that type will fail with an error"), while Paid is a $5/mo subscription with metered overage. So core compute+KV+D1 carry **no charge surface on the Free plan** — zero-charge operation is proven *for the platform quotas*; it is not a guarantee for the whole workload, which still depends on provider auth gates and unexercised CPU/memory costs. R2, Email Sending (arbitrary), Durable Objects KV-backend, Vectorize, Containers are **paid/metered** surfaces.

---

## 6. Storage and data services

| Service | Free allowance | Overage behavior | Fit |
|---|---|---|---|
| **D1** | 10 DBs/account; 500 MB/DB; **5 GB total**; 5M rows read + 100k rows written **per day**; FTS5 + JSON1 supported; 2 MB max row; 30 s query limit; 50 queries/invocation | **Hard stop** — API errors once daily row caps or 5 GB are hit; must delete data/DBs to recover | Primary store for mail metadata, folder/label maps, OAuth tokens, FTS index of synced bodies. No cost surface. [D1 limits](https://developers.cloudflare.com/d1/platform/limits/), [pricing](https://developers.cloudflare.com/d1/platform/pricing/), [SQL/FTS5](https://developers.cloudflare.com/d1/sql-api/sql-statements/) |
| **R2** | 10 GB-month, 1M Class-A, 10M Class-B ops/month (Standard class only; egress free) | **Auto-billed overage** once the R2 subscription is active; no documented hard cap; activation is a checkout (docs) + payment method mandatory per community/bug-bounty evidence — provisioning unconfirmed. Unauthorized (401) ops not billed; `r2.dev` throttled | Large MIME/attachment archive *if the owner accepts a metered dependency*; alternatives: chunked BLOBs in D1 (≤5 GB) or no body archive (on-demand provider fetch). [R2 pricing](https://developers.cloudflare.com/r2/pricing/), [get started](https://developers.cloudflare.com/r2/get-started/), [limits](https://developers.cloudflare.com/r2/platform/limits/) |
| **KV** | 100k reads/day, 1k writes/day (1/s per key), 1 GB total | Hard error past free quota | Session tokens, watch/delta cursors, small caches — not message bodies (writes too low) [KV limits](https://developers.cloudflare.com/kv/platform/limits/) |
| **Durable Objects (SQLite)** | Free plan: SQLite-backend only; 100k requests + 13,000 GB-s duration **per day**; 5 GB storage | Hard stop at daily quotas | Per-mailbox sync coordinator / connection holder if needed; DO alarms can wake cron-like work; TCP socket in DO only keeps it billed-active 15 min — not a persistent IDLE host [DO limits](https://developers.cloudflare.com/durable-objects/platform/limits/), [pricing](https://developers.cloudflare.com/workers/platform/pricing/) |
| **Queues** | 10k operations/day, 24 h retention | Hard stop | Optional async fan-out for sync batches — nice-to-have only |
| **Workflows** | 3,000 steps/day | Hard stop | Optional multi-step sync orchestration — nice-to-have only |
| **Pages vs Workers static assets** | Pages Free: 20k files/site, 25 MiB/file, 500 builds/mo; Workers static assets same file limits, requests free | Hard limit | Equivalent for a SPA; Workers + assets is simpler (one deploy, functions co-located) [Pages limits](https://developers.cloudflare.com/pages/platform/limits/), [assets billing](https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/) |

**Zero-charge guardrails that actually exist:** Cloudflare offers **budget alerts** (email notification at a dollar threshold) and per-product usage notifications — informational only, "your monthly invoice is the most reliable source." There is **no self-serve hard spend cap** in the billing docs; the only proven hard boundaries are the Free-plan error ceilings above. [Usage-based billing](https://developers.cloudflare.com/billing/understand/usage-based-billing/), [budget alerts](https://developers.cloudflare.com/billing/manage/budget-alerts/).

**Application-side guards can** count requests, enforce sync windows, cap stored bytes, refuse >N-MB bodies, and stop issuing provider calls at threshold. **They cannot** prevent a metered service (R2/Email-Sending-on-Paid) from accruing charges once its subscription exists, because billing is account-level and triggered by any authorized use — including traffic the app didn't initiate if endpoints/keys leak. Guardrails reduce likelihood; they do not convert a metered service into a hard-free one.

---

## 7. Gmail

| Area | Finding | Source |
|---|---|---|
| Auth | Owner-registered Google Cloud OAuth **web app** client; `mail.google.com/` (restricted) or narrower `gmail.modify`/`gmail.send`/`gmail.readonly` scopes. Personal-use/personal-project apps are an **explicit exception** to restricted-scope verification; Testing-state apps cap at ~100 allowlisted users with **refresh tokens expiring after 7 days**; unverified published apps show a warning screen. **App passwords** (16-digit, requires 2SV, not available for security-key-only 2SV / Advanced Protection / some org accounts) still work for IMAP/SMTP and are a documented escape hatch. | [Restricted-scope verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification) (exceptions section), [app passwords](https://support.google.com/mail/answer/185833) |
| Read/list/search | `messages.list`/`threads.list`, `q=` uses Gmail search syntax; `messages.get`, `attachments.get`. 5–40 quota units per call; per-user 6,000 units/min ⇒ ~300 `messages.get`/min. | [Quota](https://developers.google.com/workspace/gmail/api/reference/quota) |
| Send | `messages.send`/`drafts.send` with raw RFC 2822 MIME (base64url); 100 units each; the **25 MB** figure is the documented *attachment* ceiling for standard Gmail/Workspace editions (total message content + attachments *before encoding* per admin docs); the API upload path uses media-upload mechanics and Enterprise Plus raises it to 50 MB; 500 recipients/msg via API, 100 via SMTP. | [Sending guide](https://developers.google.com/workspace/gmail/api/guides/sending), [sending limits](https://knowledge.workspace.google.com/admin/gmail/gmail-sending-limits-in-google-workspace) |
| Sync | Partial sync via `history.list(startHistoryId)` — history "typically ≥1 week, sometimes shorter"; out-of-range → HTTP 404 → full re-sync. | [Sync guide](https://developers.google.com/workspace/gmail/api/guides/sync) |
| Push | `watch` posts to **Cloud Pub/Sub only** (topic + push webhook or pull subscription); `watch` must be renewed **at least every 7 days**; max 1 event/s/user. Pub/Sub is a Google Cloud dependency — technically outside "Cloudflare-only compute," though a push subscription can target the Worker's HTTPS URL. Poll-only avoids it entirely. | [Push guide](https://developers.google.com/workspace/gmail/api/guides/push) |
| Label/flag/move/delete | `messages.modify` (add/remove labelIds → INBOX/TRASH/label ops), `batchModify`, `trash`/`untrash`/`delete`. Two-way and matches Gmail's label model directly (better than folders-as-labels over IMAP). | [Quota](https://developers.google.com/workspace/gmail/api/reference/quota) (methods), baseline §4.2 |

**Verdict:** Gmail via API = **documented free path** (read/send/labels/search/sync) — push is *not proven free* because it needs Pub/Sub; polling is the free path. IMAP route is also viable via `connect()` + XOAUTH2 (or app password) but loses label fidelity and hits the 15-connection cap.

## 8. Microsoft Graph / Outlook.com / M365

| Area | Finding | Source |
|---|---|---|
| Coverage | Graph Mail API works for **personal Microsoft accounts and M365** with delegated `Mail.Read`/`Mail.ReadWrite`/`Mail.Send`; `/me` path for the signed-in owner. | [Mail API overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview), [message-get perms](https://learn.microsoft.com/en-us/graph/api/message-get), [sendMail perms](https://learn.microsoft.com/en-us/graph/api/user-sendmail) |
| Registration | Owner registers an Entra app (`AzureADandPersonalMicrosoftAccount` audience to cover both MSA and org accounts). For M365 tenants, delegated scopes may need **admin consent**; if the tenant blocks user consent or requires approval, that is a per-tenant blocker, not a platform one. | Baseline + [OAuth for IMAP/POP/SMTP](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth) |
| Read/list/search | `/me/messages`, `/me/mailFolders`; `$search` on `from`/`subject`/`body` + KQL properties; up to 1,000 results. MIME via `$value`. | [search param](https://learn.microsoft.com/en-us/graph/search-query-parameter), [mail overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview) |
| Sync | Per-folder **`/messages/delta`** returning `@odata.deltaLink`; `$filter` limited to `receivedDateTime`, `$orderby` to `receivedDateTime desc`, **no `$search` in delta**, `$filter` capped at 5,000 messages. | [Delta query](https://learn.microsoft.com/en-us/graph/delta-query-messages) |
| Push | **Webhook subscriptions on `message` support personal accounts**, max lifetime **10,080 min (~7 days)**, ≤1,000 active subscriptions/mailbox; `notificationUrl` is a public HTTPS endpoint — a Worker URL qualifies. Renewal via cron fits the Free trigger budget. | [Outlook change notifications](https://learn.microsoft.com/en-us/graph/outlook-change-notifications-overview), [subscription resource](https://learn.microsoft.com/en-us/graph/api/resources/subscription) |
| Throttle | Outlook service: **10,000 req/10 min per app per mailbox, 4 concurrent, 150 MB upload/5 min** — far above single-owner sync needs. | [Throttling](https://learn.microsoft.com/en-us/graph/throttling-limits#outlook-service-limits) |
| IMAP/SMTP OAuth | Alternative transport: `outlook.office365.com` IMAP/POP and `smtp-mail.outlook.com:587` with XOAUTH2 (`IMAP.AccessAsUser.All`, `SMTP.Send`, `offline_access`). Outlook.com IMAP/POP are **off by default** (owner toggles in settings); M365 tenants may have SMTP AUTH disabled and need admin enablement; some tenants require admin consent for these scopes. | [OAuth for legacy protocols](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth), baseline §4.3 |
| Mutate/move/delete | `PATCH /me/messages/{id}` for `isRead`/`categories`/flags; folder move is a **dedicated action** `POST /me/messages/{id}/move` (destinationId, well-known folder names); `DELETE /me/messages/{id}`; all support personal accounts. | [message-move](https://learn.microsoft.com/en-us/graph/api/message-move), [message-update](https://learn.microsoft.com/en-us/graph/api/message-update) |

**Verdict:** Graph covers Outlook.com + M365 uniformly (read, MIME send, attachments, folders, delta sync, webhooks, KQL search) and is the **preferred path**; IMAP/SMTP is a viable fallback where the owner/admin has enabled it. Webhooks are the only provider-side push that lands directly on a Worker — a real advantage vs. Gmail's Pub/Sub dependency.

## 9. Generic IMAP/SMTP and custom-domain providers

- **Sockets reach them:** `connect()` + `startTls()` covers IMAP 993/143 and SMTP 465/587; OAuth2 issuers beyond Google/Microsoft are a per-provider config problem (Thunderbird's custom-OAuth model port: the app stores issuer/endpoints itself).
- **Bounded requests:** sync = `SELECT` + `UID FETCH`/`SEARCH` windows + `UID STORE`/`MOVE`/`EXPUNGE` for two-way state; body-on-demand keeps both subrequests (≤50) and CPU bounded. No provider-agnostic push exists — poll by cron.
- **Custom-domain mailboxes** hosted externally are just endpoints; nothing in Workers requires the *domain's* DNS or MX to be Cloudflare-managed unless the app also wants Email Routing on it.
- **Provider message sizes** are provider-side: Gmail's documented attachment ceiling is 25 MB (standard editions; 50 MB Enterprise Plus web) and Cloudflare inbound is 25 MiB; other providers vary — treat as per-provider config, not a platform limit.

---

## 10. Limits and unknowns

1. **Durable Objects free storage** is internally inconsistent across official pages: the [DO limits table](https://developers.cloudflare.com/durable-objects/platform/limits/) says **5 GB/account** for SQLite-backed DOs on Free, while the same page's SQLITE_FULL FAQ says "**1 GB** on the Free plan," and [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/) lists "SQL Stored data 5 GB (total)" for Free — unresolved; verify before sizing a DO store.
2. **10 ms CPU vs. real MIME work** is unverified empirically: `postal-mime` is doc-recommended inside Email Workers, but no published timing for multi-MB messages exists. Large-message parse may need chunking, a DO with higher CPU, or Paid — a *risk*, not a proven blocker.
3. **Socket lifetime** during an open HTTP request has no documented upper bound, but sockets cannot outlive invocations and cannot be pre-warmed in global scope; per-mailbox persistent IMAP connections therefore can't be assumed — design for connect-on-demand.
4. **R2 payment-method requirement** is confirmed by Cloudflare's own checkout copy + community + a Cloudflare HackerOne report, but not stated as "card required" in pricing docs; treat as strong evidence, confirm at provisioning.
5. **Graph `$search` on personal accounts** — docs document `$search` on `/me/messages` generally; no explicit "consumer supported" line was found this pass. Delta sync is per-folder only (no mailbox-wide delta).
6. **Gmail `history.list` retention** is "typically at least one week, often longer" — not contractual; offline >7 days can force full re-sync (404 path).
7. **Gmail API daily threshold (80M units/day)** maps to a *planned* 2026 billing model — currently free, but "all standard use … at no additional cost" language signals a future billable surface to watch.
8. **Outlook.com IMAP toggle** and **M365 SMTP-AUTH/admin-consent** are per-account/tenant gates outside the app's control — connection can be blocked even when the platform path is sound.
9. **KV-backend Durable Objects, Vectorize, Containers, Workers Logpush** are paid-only surfaces — irrelevant to the required feature set, listed so they aren't accidentally pulled in.
10. **Email Routing domain limits** — 30 email-configured domains/zone, 200 rules/domain, 200 verified destinations/account — irrelevant for connecting existing mailboxes but bound any inbound-forwarding design.

---

## 11. Narrow architecture candidates (not a decision)

| Shape | Core path | Free? | Trade-off |
|---|---|---|---|
| **A — Provider-API-first, D1-only** | Gmail API + Graph + generic IMAP over `connect()`; metadata+FTS+tokens in D1; bodies fetched on demand or chunked in D1 (≤5 GB); cron `*/1` sync; Worker SPA assets | No metered Cloudflare surface *once running* — but depends on owner OAuth registrations, per-tenant consent gates, and unexercised runtime costs; not a blanket guarantee | 5 GB/500 MB-per-DB bounds the local cache; no R2 |
| **B — Same + R2 body/attachment store** | As A, plus raw MIME + attachments in R2 | Needs owner sign-off: R2 subscription = metered overage (no documented cap) + payment-method evidence pending provisioning | More cache headroom; introduces the only metered dependency |
| **C — + optional inbound layer** | As A/B, plus Email Routing on a dedicated subdomain for app addresses/notifications to verified destinations | Free (Routing); never sends outbound as mailbox identity | Optional; does not replace provider mailboxes |
| **D — Push-assisted (owner-opted)** | As A/B, plus Graph webhooks (≤7 d renew) and *optionally* Gmail `watch`→Pub/Sub push to the Worker | Graph webhooks free on CF side; Pub/Sub is an external Google dependency | Gmail push breaks strict "Cloudflare-only compute" — owner decision |

All shapes keep the three hard rules: send through providers, poll/delta for changes, and treat R2 as a gated add-on, not a default.

---

## 12. Decisions for the owner

1. **R2 yes/no** — accept a metered-overage dependency (with app-side byte caps + budget alerts as *soft* guard, plus payment-method evidence to confirm at provisioning), or mandate D1-only. This is a requirement conflict, not an implementation choice.
2. **Email Sending** — confirm exclusion: it is Paid-plan for arbitrary recipients and cannot send as Gmail/Outlook identities anyway; provider paths cover A3. The only free Email-Service sends are to the owner's own verified addresses.
3. **Push vs poll** — accept ≥1-min cron/delta polling as the baseline (free); decide separately whether Graph webhooks and/or Gmail-Pub/Sub are worth the extra moving parts.
4. **Gmail auth model** — owner-registered web client in *Testing* (7-day refresh tokens, ≤100 users), *unverified published* (warning screen), *verified* (review + possible security assessment), or **app-password fallback**; all documented, pick per-instance.
5. **Microsoft path** — default to Graph for both Outlook.com and M365; keep IMAP/SMTP-OAuth as an opt-in fallback for tenants that have it enabled.
6. **Cache bound** — pick the sync window and the on-demand-body policy; 5 GB D1 or 10 GB R2 are the ceilings, not the floor.
7. **Inbound features** — Email Routing only makes sense for app-facing addresses on a Cloudflare-DNS domain the owner already controls; it does not touch existing provider MX.

---

## 13. Source list

Cloudflare (developers.cloudflare.com):
- Email Service overview — https://developers.cloudflare.com/email-service/
- Email Service pricing — https://developers.cloudflare.com/email-service/platform/pricing/
- Email Service limits — https://developers.cloudflare.com/email-service/platform/limits/
- Route emails (Email Routing setup) — https://developers.cloudflare.com/email-service/get-started/route-emails/
- Send emails (Email Sending setup) — https://developers.cloudflare.com/email-service/get-started/send-emails/
- Email handler / `message.reply` constraints — https://developers.cloudflare.com/email-service/api/route-emails/email-handler/
- Email Service FAQ — https://developers.cloudflare.com/email-service/reference/faq/
- Workers limits — https://developers.cloudflare.com/workers/platform/limits/
- Workers pricing — https://developers.cloudflare.com/workers/platform/pricing/
- TCP sockets `connect()` — https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/
- Cron Triggers — https://developers.cloudflare.com/workers/configuration/cron-triggers/
- Workers static assets billing — https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/
- Pages limits — https://developers.cloudflare.com/pages/platform/limits/
- D1 limits — https://developers.cloudflare.com/d1/platform/limits/
- D1 pricing — https://developers.cloudflare.com/d1/platform/pricing/
- D1 SQL statements / FTS5 — https://developers.cloudflare.com/d1/sql-api/sql-statements/
- R2 pricing — https://developers.cloudflare.com/r2/pricing/
- R2 limits — https://developers.cloudflare.com/r2/platform/limits/
- R2 get started (subscription/checkout) — https://developers.cloudflare.com/r2/get-started/
- KV limits — https://developers.cloudflare.com/kv/platform/limits/
- Durable Objects limits — https://developers.cloudflare.com/durable-objects/platform/limits/
- Usage-based billing / budget alerts — https://developers.cloudflare.com/billing/understand/usage-based-billing/

Google:
- Gmail API usage limits — https://developers.google.com/workspace/gmail/api/reference/quota
- Sync guide (history.list) — https://developers.google.com/workspace/gmail/api/guides/sync
- Push notifications (watch→Pub/Sub) — https://developers.google.com/workspace/gmail/api/guides/push
- Create and send email messages — https://developers.google.com/workspace/gmail/api/guides/sending
- `users.messages.send` — https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages/send
- Restricted-scope verification + exceptions — https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification
- Gmail sending limits (Workspace admin) — https://knowledge.workspace.google.com/admin/gmail/gmail-sending-limits-in-google-workspace
- App passwords — https://support.google.com/mail/answer/185833

Microsoft (learn.microsoft.com):
- Mail API overview — https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
- Get message (delegated perms incl. personal) — https://learn.microsoft.com/en-us/graph/api/message-get
- `sendMail` — https://learn.microsoft.com/en-us/graph/api/user-sendmail
- Send MIME via Graph — https://learn.microsoft.com/en-us/graph/outlook-send-mime-message
- Delta query for messages — https://learn.microsoft.com/en-us/graph/delta-query-messages
- `$search` parameter — https://learn.microsoft.com/en-us/graph/search-query-parameter
- Outlook change notifications — https://learn.microsoft.com/en-us/graph/outlook-change-notifications-overview
- Subscription resource (lifetime table) — https://learn.microsoft.com/en-us/graph/api/resources/subscription
- Service throttling limits (Outlook section) — https://learn.microsoft.com/en-us/graph/throttling-limits
- OAuth for IMAP/POP/SMTP — https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth

Corroborating non-primary evidence (flagged as such):
- R2 card-on-file requirement: Cloudflare Community threads [945179](https://community.cloudflare.com/t/why-using-r2-free-tier-involves-giving-card-info/945179) and [887578](https://community.cloudflare.com/t/if-i-want-to-use-cloudflare-r2-i-have-to-link-a-payment-method-i-suggest-not-doin/887578), and Cloudflare's own HackerOne report #2170559 ("R2 requires customers to have a valid payment method on file in order to activate").
