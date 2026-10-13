# Thunderbird Mailbox Model and Feature Surface

**Date:** 2026-10-13
**Ticket:** [Research Thunderbird's mailbox model and feature surface (issue #2)](https://github.com/tmih06/cf-mailbox/issues/2), under the map [Chart a free-tier Cloudflare personal unified mailbox (issue #1)](https://github.com/tmih06/cf-mailbox/issues/1)
**Scope:** What Thunderbird actually does for Gmail, Outlook.com / Microsoft 365, and generic IMAP/SMTP/POP accounts — identities, folders/labels, sync, threading, search, attachments, sending, authentication, and two-way state changes — plus which of those behaviors can port to a browser-based client hosted on Cloudflare and which cannot be assumed to port.

**Method.** Primary sources only: Mozilla Support (SUMO) knowledge base, Microsoft Learn/Support, Google Workspace developer docs, comm-central source on Searchfox, and IETF RFCs. Each claim cites its owning source at the row or paragraph level. Where a fact could not be verified from a primary source, it is listed under [Limits and unknowns](#limits-and-unknowns) rather than asserted.

---

## 1. Executive summary

Thunderbird is a **local-first, multi-account, multi-protocol mail client**: it stores a full local replica of message metadata (and optionally bodies) per account, keeps that replica synchronized with the provider over IMAP, and layers presentation (folder-pane views, unified inboxes, threaded lists, virtual "Saved Search" folders, the Gloda full-text index) on top of the local replica. Sending is always SMTP or a provider-specific API; identities (From name/address, signature, per-identity copy folders) are per-account configurations, not server objects. Gmail's label model is squeezed into IMAP folders: one Gmail message with N labels materializes as N message copies in Thunderbird, and "archive" = removing the `Inbox` label = moving to `All Mail`. Microsoft-hosted accounts require OAuth2 (basic auth is gone); on Outlook.com, IMAP/POP3 are offered but disabled by default, and on Microsoft 365, SMTP AUTH is disabled by default for new tenants; EWS is being phased out for M365 in favor of Microsoft Graph (Thunderbird's own EWS path is being replaced in 2026–2027). POP3 exists but is deliberately one-way — a candidate for explicit exclusion rather than a baseline.

The single most important portability finding: **everything Thunderbird does below the folder-pane UI assumes (a) raw TCP/TLS sockets to arbitrary ports, (b) a persistent process that holds long-lived IMAP connections and IDLE waits, and (c) a local filesystem profile holding mbox/maildir stores, per-folder `.msf` databases, a global full-text index, and credentials.** Whether a browser UI plus Cloudflare Workers runtime can express the same assumptions is unverified at this stage — that is exactly what the next ticket must test, along with which behaviors move into Workers/Durable Objects and which must be re-shaped.

---

## 2. Terminology (as Thunderbird uses it)

| Term | Meaning in Thunderbird | Source |
|---|---|---|
| **Account** | A configured server connection: incoming protocol (IMAP, POP3, NNTP, RSS, EWS/Exchange, Graph) + outgoing SMTP + an identity. Each account owns its folder tree, filters, junk settings, sync settings. | [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts); [SUMO: Glossary](https://support.mozilla.org/en-US/kb/glossary-terms-including-types-accounts) |
| **Identity** | A per-account tuple: display name, From address, optional Reply-To, signature, outgoing server, Copies & Folders settings. Multiple identities per account; picked in the From dropdown when composing; auto-selected on reply by matching which address the original message was sent to. | [SUMO: Using Identities](https://support.mozilla.org/en-US/kb/using-identities) |
| **Folder** | An IMAP mailbox on the server **or** a folder in Local Folders. Folders are protocol-aware — "the folder knows what protocol it's based on … the folder and protocol are tightly intertwined." | [comm-central `mailnews/base/docs/folder_storage.md`](https://searchfox.org/comm-central/source/mailnews/base/docs/folder_storage.md) |
| **Local Folders** | A pseudo-account with no server: stores POP downloads, Send Later outbox, and any local archive/moved mail. | [SUMO: Unify POP accounts with a global inbox](https://support.mozilla.org/en-US/kb/unify-your-pop-email-accounts-global-inbox); [SUMO: Glossary](https://support.mozilla.org/en-US/kb/glossary-terms-including-types-accounts) |
| **Unified folders (folder mode)** | Folder-pane *view* that merges same-kind folders across accounts into virtual "Inbox/Drafts/Sent/Trash/…" nodes. A newer, non-destructive alternative to the POP "Global Inbox" deferral. | [SUMO forum answer #1566028](https://support.mozilla.org/en-US/questions/1566028); [SUMO: global inbox](https://support.mozilla.org/en-US/kb/unify-your-pop-email-accounts-global-inbox) |
| **Global Inbox (POP)** | Older technique: defer multiple POP accounts' storage into one Local Folders inbox. Requires restart; hides the per-account branches. | [SUMO: global inbox](https://support.mozilla.org/en-US/kb/unify-your-pop-email-accounts-global-inbox) |
| **Saved Search folder** | A virtual folder that evaluates search criteria live over chosen source folders (can span accounts). Shows the real messages, not copies — deleting inside a saved search deletes the real message. | [SUMO: Using Saved Searches](https://support.mozilla.org/en-US/kb/using-saved-searches) |
| **Tags** | Named, color-coded keywords applied to a message (Important, Work, Personal, To Do, Later + custom). Filterable via Quick Filter. Backed by IMAP keywords where the server supports them. | [SUMO: Message Tags](https://support.mozilla.org/en-US/kb/message-tags) |
| **Archive** | Manual action (button/`A`) that moves messages into a dedicated archive folder — either on the server (IMAP) or in Local Folders — with per-identity location and folder-structure options. | [SUMO: Archived Messages](https://support.mozilla.org/en-US/kb/archived-messages) |
| **Gloda** | "Global database": Thunderbird's cross-account full-text index powering Global Search (Ctrl+K), facets, and autocomplete. Indexes synced/downloaded messages only. | [SUMO: Global Search](https://support.mozilla.org/en-US/kb/global-search); [SUMO: Rebuilding the Global Database](https://support.mozilla.org/en-US/kb/rebuilding-global-database) |
| **Message store / message database** | Per folder: a raw store (`mbox` file or `maildir` directory) + a `.msf` header/metadata database (Mork format). Headers are fetched for every message; bodies may or may not be downloaded. | [comm-central `folder_storage.md`](https://searchfox.org/comm-central/source/mailnews/base/docs/folder_storage.md) |
| **Offline mode** | Manual/Automatic "work offline" state; operations queue locally and sync on reconnect (IMAP). Send Later → Outbox. | [SUMO: Glossary](https://support.mozilla.org/en-US/kb/glossary-terms-including-types-accounts); [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization) |
| **Expunge / Compact** | IMAP `\Deleted`-flagged messages are permanently removed ("expunged") — per-folder "Compact" or on exit settings. | [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts) |
| **Junk/Spam** | Adaptive Bayesian filter trained by marking spam/not-spam, plus whitelist-by-address-book and optional trust of server-set headers; renamed "Spam" in v145. | [SUMO: Thunderbird and Junk/Spam](https://support.mozilla.org/en-US/kb/thunderbird-and-junk-spam-messages) |
| **Biff** | Thunderbird jargon for new-mail checking (poll interval, IDLE notifications, alerts). | [comm-central `mailnews.js` prefs](https://searchfox.org/comm-central/source/mailnews/mailnews.js) |

---

## 3. Accounts, identities, and sending

| Capability | Behavior | Provider notes | Source |
|---|---|---|---|
| Multiple accounts | Unlimited accounts side-by-side; each gets own folder tree, filters, junk settings, sync settings, SMTP server choice. | Same for all providers. | [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts) |
| Multiple identities per account | Name + From address + Reply-To + signature + per-identity archive/copy settings + per-identity outgoing SMTP server. From dropdown in compose lists identities across all accounts. | Identities are a **client-side construct**: the server just sees an SMTP envelope + `From:` header. Works for any SMTP server that accepts the From domain. | [SUMO: Using Identities](https://support.mozilla.org/en-US/kb/using-identities) |
| Reply identity selection | On reply, Thunderbird scans the message's recipients and picks the identity the sender used to reach you; user can override per message. | Same for all providers; fails gracefully when recipient info is missing (e.g., some mailing lists). | [SUMO: Using Identities](https://support.mozilla.org/en-US/kb/using-identities) |
| Sent copies | "Place a copy in" — configurable sent folder per identity; option "Place replies in the folder of the message being replied to." | **Gmail & Outlook.com auto-file SMTP-sent mail server-side** — Google explicitly recommends *unchecking* the client-side copy to avoid duplicates. | [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts); [Google: IMAP client settings](https://support.google.com/mail/answer/78892?hl=en) |
| Drafts & Templates | Configurable Drafts and Templates folders per account (IMAP folder → server-synced, or local). | Google: set Drafts to `[Gmail]/Drafts` so drafts appear in Gmail web. | [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts); [Google: IMAP client settings](https://support.google.com/mail/answer/78892?hl=en) |
| Outbox / Send Later | Offline or deferred sends queue in the Local Folders "Outbox" and send on reconnect/`Send Unsent Messages`. | Client-side queue; invisible to server until sent. | [SUMO: Glossary](https://support.mozilla.org/en-US/kb/glossary-terms-including-types-accounts) |
| Compose | HTML or plain-text compose, quoting position, signatures, vCard attach, per-identity CC/Bcc defaults, spellcheck, return-receipt handling. | Message is RFC 5322 + MIME; provider limits on message size apply on send (SMTP) — see Limits. | [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts) |
| Large attachments (Filelink) | Optional: uploads attachments >N MB (default offer threshold 5 MB) to a storage provider add-on and inserts a download link instead of attaching. Regular attachments still work. | Provider-agnostic; depends on third-party storage add-ons (WebDAV, Thundermail/Send). | [SUMO: Filelink](https://support.mozilla.org/en-US/kb/filelink-large-attachments) |
| Redirect / "Edit as new" | Beyond scope-checked docs; listed as optional capability. | — | capability listed in Thunderbird UI; not separately documented in the pages read |

---

## 4. Message and folder model

### 4.1 IMAP (the primary path)

- **Two-way mirror:** IMAP is "a two-way protocol that copies your email … while preserving the original copy … Changes … such as marking them as read, adding or removing labels or tags, or moving them to a different folder, will be replicated on the remote server." Offline changes queue and sync on reconnect. [SUMO: Difference between IMAP and POP3](https://support.mozilla.org/en-US/kb/difference-between-imap-and-pop3)
- **Sync granularity:** Per-account "Keep messages for this account on this computer" toggle; per-folder Advanced sync selection (sub-folders are NOT auto-synced with the parent — each folder checked individually). Non-synced folders show headers only; the body is fetched on click. Disk-space controls can limit sync to recent messages / under a size cap. [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization)
- **Store vs. database:** headers populate a per-folder `.msf` database (Mork) immediately; full bodies stream to the `mbox`/`maildir` store in the background for synced folders. EWS, by contrast, stores only DB entries and downloads bodies on demand. [comm-central `folder_storage.md`](https://searchfox.org/comm-central/source/mailnews/base/docs/folder_storage.md)
- **Server-side folder listing & subscription:** Thunderbird lists server folders and defaults to showing only subscribed folders; subscription toggled per folder (`Subscribe…`). Advanced settings cover namespaces, dual-use folders (folders containing both messages and subfolders), server directory prefix (e.g. `[Gmail]`), and a max-cached-connections cap. [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts)
- **New-mail detection:** poll ("check every N minutes") **or** IMAP IDLE ("Allow immediate server notifications when new messages arrive"), plus "check at startup". Non-inbox folders can opt into being checked via a per-folder property. [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts); [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail)
- **Delete models (per account, IMAP only):** (1) *Move to Trash* (default), (2) *Move to a chosen folder*, (3) *Just mark as deleted* — `\Deleted` flag with strike-through, cleared by Compact/expunge — or (4) *Remove immediately*. Additional toggles: "Expunge Inbox on Exit", "Empty Trash on Exit". [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts)
- **Extension support in the IMAP engine (source):** CONDSTORE (RFC 7162, flagged `useCondStore`), IDLE (RFC 2177), MOVE (RFC 6851), UIDPLUS (RFC 4315), LIST-EXTENDED (RFC 5258), SPECIAL-USE (RFC 6154), COMPRESS=DEFLATE (RFC 4978), ENABLE (RFC 5161), UTF8=ACCEPT (RFC 6855), XLIST (legacy Gmail), X-GM-EXT-1 detection, plus auth SASL mechanisms LOGIN/PLAIN/CRAM-MD5/NTLM/GSSAPI/EXTERNAL/XOAUTH2. [comm-central `mailnews/imap/src/nsImapCore.h`](https://searchfox.org/comm-central/source/mailnews/imap/src/nsImapCore.h); [nsImapIncomingServer.cpp `useCondStore`](https://searchfox.org/comm-central/search?q=condstore&path=mailnews)

### 4.2 Gmail over IMAP — the divergent model

Gmail "does not use a standard IMAP archive folder … archiving simply removes the Inbox label while the email remains in All Mail." [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail)

| Gmail semantic | How it surfaces in Thunderbird | Source |
|---|---|---|
| Labels | Each label = an IMAP folder. Moving a message to a folder assigns the label; copying to a second folder adds a second label — but **Thunderbird stores multiple local copies of the same message**, one per folder. | [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail); [Google: IMAP extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions) |
| Archive | Thunderbird's Archive moves the message to an "Archive"/`All Mail` folder = removes `\Inbox` on the server. Google recommends pointing the archive to `Other` → the account root (i.e., All Mail semantics) for parity. | [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail) |
| Delete | Deleting any copy to Trash removes **all** labels and deletes the message everywhere in Thunderbird. Google recommends "Just mark it as deleted" + no expunge/empty-trash so deleted mail lands in All Mail; mapping Trash to `[Gmail]/Trash` makes deletes permanent after ~30 days. | [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail); [Google: IMAP client settings](https://support.google.com/mail/answer/78892?hl=en) |
| System labels as special folders | `[Gmail]` namespace children: All Mail, Drafts, Sent, Spam, Starred, Trash, Important — advertised via SPECIAL-USE (`\All`, `\Drafts`, `\Junk`, `\Flagged`, `\Trash`, `\Sent`, plus nonstandard `\Important`). | [Google: IMAP extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions) |
| Star | The `\Flagged` IMAP flag = Gmail Starred; bidirectional. | [Google: IMAP extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions) (`\Flagged` → `[Gmail]/Starred`) |
| Spam | Moving to `[Gmail]/Spam` trains Gmail. Google recommends **disabling Thunderbird's adaptive junk controls** for the account. | [Google: IMAP client settings](https://support.google.com/mail/answer/78892?hl=en) |
| Subscription | All folders subscribe + download by default; Mozilla explicitly recommends unsubscribing `[Gmail]/All Mail` to avoid duplicate local copies. | [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail) |
| Connection/rate limits | Google hard-codes **15 simultaneous IMAP connections per account across all clients and folders**; exceeding it yields missed/late mail. Mitigate by disabling IDLE, disabling "check all folders for new messages", and lowering cached-connection count. | [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail) |
| `X-GM-EXT-1` | Thunderbird *detects* `X-GM-EXT-1` (`kGmailImapCapability`, `SetIsGMailServer`) and stores `X-GM-LABELS` fetch responses as a string property on message headers (`SetStringProperty("X-GM-LABELS", …)`), but the client-visible model remains folders-as-labels. `X-GM-THRID`/`X-GM-RAW` exist server-side; no uses of `X-GM-RAW` were found in comm-central (see Limits). | [nsImapCore.h](https://searchfox.org/comm-central/source/mailnews/imap/src/nsImapCore.h); [nsImapIncomingServer.cpp](https://searchfox.org/comm-central/search?q=kGmailImapCapability); [nsMsgDatabase.cpp](https://searchfox.org/comm-central/search?q=X-GM-LABELS); [Google: IMAP extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions) |

### 4.3 Outlook.com / Microsoft 365

- **Protocols:** IMAP (`outlook.office365.com:993`), POP3 (`:995`), SMTP (`smtp-mail.outlook.com:587`, STARTTLS), plus Thunderbird's native **EWS and Microsoft Graph** account types (v154+/recommended v157+; email only — no calendar/address book/shared mailboxes, no on-prem OAuth yet). [Microsoft: POP/IMAP/SMTP settings](https://support.microsoft.com/en-us/office/pop-imap-and-smtp-settings-for-outlook-com-d088b986-291d-42b8-9564-9c414e2aa040); [SUMO: Thunderbird and Exchange](https://support.mozilla.org/en-US/kb/thunderbird-and-exchange)
- **Gates:** POP & IMAP are **disabled by default** on Outlook.com — the owner must toggle "Let devices and apps use POP/IMAP" ON in web settings. Outlook.com **requires Modern Auth/OAuth2**. IMAP on a personal account may hit "suspicious activity" connection errors that require approving the session at account.live.com/activity. [Microsoft: settings page](https://support.microsoft.com/en-us/office/pop-imap-and-smtp-settings-for-outlook-com-d088b986-291d-42b8-9564-9c414e2aa040)
- **AuthZ:** Entra-registered app must request `https://outlook.office.com/IMAP.AccessAsUser.All`, `…/POP.AccessAsUser.All`, `…/SMTP.Send` (+`offline_access` for refresh tokens); SASL `XOAUTH2` carries `base64("user=" + user + "^Aauth=Bearer " + token + "^A^A")` for IMAP, POP and SMTP alike. [Microsoft Learn: Authenticate IMAP/POP/SMTP with OAuth](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth)
- **M365 tenant restrictions:** basic auth is disabled in all tenants and cannot be re-enabled; SMTP AUTH "was disabled in all tenants where it wasn't being used" — new tenants start with SMTP AUTH off and an admin must re-enable per-mailbox/tenant; EWS for M365 is being shut down starting **October 2026**, complete shutdown in **2027** → Thunderbird Graph path is the future-proof one. Also possible: admin-consent screen requiring a tenant admin to approve the app for `IMAP.AccessAsUser.All`, `POP.AccessAsUser.All`, `SMTP.Send`, `offline_access`. [Microsoft Learn: Basic auth deprecation](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-basic-authentication-exchange-online); [SUMO: Microsoft OAuth in 2026](https://support.mozilla.org/en-US/kb/microsoft-oauth-authentication-and-thunderbird-202); [SUMO: EWS→Graph migration](https://support.mozilla.org/en-US/kb/microsoft-ews-graph-migration)
- **State fidelity:** standard IMAP folders/flags; no label concept. Exchange-side categories, follow-up flags, conversation view etc. do not map into Thunderbird's IMAP view; Thunderbird tags are stored as IMAP keywords where the server permits (keyword support on Exchange Online is not confirmed — see Limits).

### 4.4 Generic IMAP/SMTP + externally hosted domains

- Any provider not in Thunderbird's autoconfig database falls back to manual server configuration; the ISPDB (ISP database) + autoconfig XML + provider autodiscovery power the automatic flow. External custom-domain mailboxes are just IMAP/SMTP endpoints. [SUMO: Automatic Account Configuration](https://support.mozilla.org/en-US/kb/automatic-account-configuration); [SUMO: Manual Account Configuration](https://support.mozilla.org/en-US/kb/manual-account-configuration)
- Auth methods selectable per server: normal password, encrypted password, OAuth2 (known issuers or **custom OAuth config** — add-on or manual entry of issuer/client ID/endpoints, v156+), client certificate, GSSAPI/Kerberos, NTLM, CRAM-MD5, or "no authentication". [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts); [SUMO: Custom OAuth](https://support.mozilla.org/en-US/kb/tb-custom-oauth); [comm-central OAuth2Providers.sys.mjs](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs)
- Custom OAuth issuers for IMAP/POP/Exchange can be defined without code changes since v155–156; Mozilla explicitly notes the hard-coded built-in issuer table is temporary until dynamic client registration (DCR) / PACC lands. [SUMO: Custom OAuth](https://support.mozilla.org/en-US/kb/tb-custom-oauth); [OAuth2Providers.sys.mjs comment](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs) ("Don't copy these values for your own application — register one for yourself!")

### 4.5 POP3 — supported but deliberately limited

- One-way download from server Inbox to local storage; **default is to delete from the server after 7 days** unless "Leave messages on server" is set (`for at most N days` or `until I delete them`). No server folders — everything lands in the account's local Inbox (or a shared "Global Inbox" if deferred). Read/flag/move states are local-only and never sync back. Headers-only fetch ("Fetch headers only") exists as a size guard. Cannot convert POP↔IMAP in place — the account must be re-created. [SUMO: IMAP vs POP3](https://support.mozilla.org/en-US/kb/difference-between-imap-and-pop3); [SUMO: Glossary](https://support.mozilla.org/en-US/kb/glossary-terms-including-types-accounts); [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts)

---

## 5. Reading surface: folders, threads, tags, flags

| Feature | Behavior | Provider notes | Source |
|---|---|---|---|
| Threaded view | Default since v102. Hierarchical thread trees per folder; collapse/expand, mass actions on a thread, "ignore thread/subthread" (hides + silences notifications), apply current view to other folders. Threading is by References/In-Reply-To plus subject fallback — `mail.strict_threading` (default true) and `mail.correct_threading` (default true) govern it. | Gmail's own "conversation view" is a Gmail display pref that can be aligned with Thunderbird's; Gmail also exposes X-GM-THRID but Thunderbird builds threads locally. | [SUMO: Message threading](https://support.mozilla.org/en-US/kb/message-threading-thunderbird); [SUMO: Ignoring threads](https://support.mozilla.org/en-US/kb/ignoring-threads); [comm-central `mailnews.js` / `nsMsgDatabase.cpp`](https://searchfox.org/comm-central/source/mailnews/mailnews.js) |
| Read/unread | Standard `\Seen` flag, two-way on IMAP. | Works on all IMAP providers; POP is local-only. | [SUMO: IMAP vs POP3](https://support.mozilla.org/en-US/kb/difference-between-imap-and-pop3) |
| Star | `\Flagged` flag → Gmail Starred; generic IMAP flag where supported. | — | [Google: IMAP extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions) |
| Tags | Named colored keywords; custom tags supported; Quick Filter by tag. Backed by IMAP keywords (see Limits — Gmail keyword storage unverified; likely labels-as-folders instead). | Provider-dependent persistence. | [SUMO: Message Tags](https://support.mozilla.org/en-US/kb/message-tags) |
| Quick Filter | Non-indexed instant filter over the current list: unread/starred/contact/tags/attachment + sender/recipient/subject/body term filters; pinnable across folders. | Client-side; no server involvement. | [SUMO: Quick Filter Toolbar](https://support.mozilla.org/en-US/kb/quick-filter-toolbar) |
| Saved Searches | Cross-account virtual folders over live query criteria; operate on real messages (not copies); "Search Online" option delegates body search to the IMAP server for non-downloaded mail (slower). | Server-search path = IMAP SEARCH command; only needed for body criteria on unsynced messages. | [SUMO: Using Saved Searches](https://support.mozilla.org/en-US/kb/using-saved-searches) |
| Archive | Manual move to dedicated archive folder (server or Local Folders), per-identity config, optional yearly/monthly/original-structure subfolders; ~2,000-message batch limit recommendation. | Gmail: see §4.2 — archive maps to label removal / All Mail. | [SUMO: Archived Messages](https://support.mozilla.org/en-US/kb/archived-messages) |
| Ignore thread | Marks conversation ignored; hidden by default, no new-mail notifications for replies; toggle visibility via View → Threads → Ignored Threads. | Local flag (MDB), not an IMAP concept — see Limits. | [SUMO: Ignoring threads](https://support.mozilla.org/en-US/kb/ignoring-threads) |

---

## 6. Search

| Layer | Scope | How it works | Source |
|---|---|---|---|
| Global Search (Gloda) | All accounts, all synced messages | Local full-text index ("Gloda"): tokenizes on delimiters, min term 3 chars, Porter stemming ("find"↔"finds"↔"finding"), phrase search via quotes, faceted results (people/folders/accounts), timeline. Trash & Spam excluded unless per-folder opt-in. **Only synced/downloaded messages are indexed** — unsynced messages contribute headers only. | [SUMO: Global Search](https://support.mozilla.org/en-US/kb/global-search); [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization) |
| Classic Search (Ctrl+Shift+F) | Per account/folder | Rule-based header/body matching; can run against the server ("Search Online" / IMAP SEARCH) for unsynced bodies. | [SUMO: Using Saved Searches](https://support.mozilla.org/en-US/kb/using-saved-searches); [SUMO forum #1274179](https://support.mozilla.org/en-US/questions/1274179) |
| Quick Filter | Current message list | Local, instant, no index needed beyond headers. | [SUMO: Quick Filter Toolbar](https://support.mozilla.org/en-US/kb/quick-filter-toolbar) |
| Gmail X-GM-RAW | Server-side Gmail query syntax (`has:attachment in:unread`) exposed as an IMAP SEARCH attribute | Available to any IMAP client; **no comm-central uses found** — Thunderbird does not currently leverage Gmail's native search syntax. Verify before assuming parity with Gmail web search. | [Google: IMAP extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions); source search "X-GM-RAW" on comm-central (zero hits) |

**Search implication for our model:** Thunderbird's "search everything instantly" promise rests on (a) downloaded bodies and (b) a local SQLite index. On a bounded-cache design, full-text recall is limited to the synced window unless the app issues server-side `SEARCH`/`UID SEARCH` (headers, envelope, and `X-GM-RAW` for Gmail) — a real capability gap to decide on, not an artifact.

---

## 7. Attachments

- Received attachments are part of the RFC 5322 message body — they download with the synced message and can be opened/saved/detached locally; attachment state rides the message copy, there is no separate object. For unsynced messages the body (and thus attachments) is fetched on demand. [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization); [comm-central folder_storage.md](https://searchfox.org/comm-central/source/mailnews/base/docs/folder_storage.md)
- Sending uses standard MIME attachments; Filelink (§3) is the optional path for large files. [SUMO: Filelink](https://support.mozilla.org/en-US/kb/filelink-large-attachments)
- Provider message-size ceilings apply on send (SMTP 587 submission) and are enforced by the provider, not Thunderbird. Exact per-provider limits were not re-verified for this report (see Limits).

---

## 8. Filters and spam

- **Message filters:** per-account rule lists (property/test/value → actions like move/copy/delete/forward/tag/mark-read/star/reply). Run on *Getting New Mail* (before or after junk classification), *Manually Run*, *After Sending* (sent-mail filters), or *Archiving*. Automatic filters apply to the Inbox only. Filter order is heeded unless impossible. [SUMO: Message Filters](https://support.mozilla.org/en-US/kb/organize-your-messages-using-filters)
- **Adaptive junk/spam:** Bayesian filter trained by user's spam/not-spam marks; global + per-account settings; whitelist by address book; can trust server-set spam headers (SpamAssassin, Bogofilter, DSPAM, POPFile, SpamPal). Renamed "Spam" in v145. Mozilla's own guidance: choose a provider with good server-side spam protection; disable Thunderbird's filter for Gmail since Gmail handles it. [SUMO: Junk/Spam](https://support.mozilla.org/en-US/kb/thunderbird-and-junk-spam-messages); [Google: IMAP client settings](https://support.google.com/mail/answer/78892?hl=en)

---

## 9. Authentication

| Provider | Mechanism | Evidence |
|---|---|---|
| Gmail / Google Workspace | Password-based client auth ("less secure apps") is off for consumer accounts (announced Jan 2025) and Workspace (March 14, 2025); third-party clients use OAuth2 — **with a documented exception: Google "app passwords"** (16-digit codes, only with 2-Step Verification on; not offered for accounts using security-key-only 2SV, Advanced Protection, or some work/school orgs) still work for apps without "Sign in with Google", though Google calls them "not recommended". Thunderbird itself uses OAuth: consent in the system browser (v153+), `usePKCE` + `useExternalBrowser`, built-in client ID, scope `https://mail.google.com/` (a restricted scope; verification paths detailed below). | [Google: less-secure-apps transition](https://support.google.com/a/answer/14114704); [Google: Sign in with app passwords](https://support.google.com/mail/answer/185833?hl=en); [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail); [OAuth2Providers.sys.mjs](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs); [Google: restricted scope verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification) |
| Outlook.com / M365 | OAuth2/XOAUTH2 for IMAP, POP, SMTP; basic auth dead in all tenants; admin consent may be needed for org tenants; SMTP AUTH per-mailbox/tenant toggle. Thunderbird uses `/common` endpoints, app `9e5f94bc-…`, `useExternalBrowser: true`. | [Microsoft Learn: OAuth for IMAP/POP/SMTP](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth); [Microsoft Learn: basic-auth deprecation](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-basic-authentication-exchange-online); [SUMO: Microsoft OAuth 2026](https://support.mozilla.org/en-US/kb/microsoft-oauth-authentication-and-thunderbird-202); [OAuth2Providers.sys.mjs](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs) |
| Generic IMAP/SMTP | Password (PLAIN/LOGIN/CRAM-MD5), OAuth2 for known issuers, client-cert, GSSAPI, NTLM; custom OAuth issuers user-configurable since v156 (or via add-on earlier). | [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts); [SUMO: Custom OAuth](https://support.mozilla.org/en-US/kb/tb-custom-oauth); [nsImapCore.h auth flags](https://searchfox.org/comm-central/source/mailnews/imap/src/nsImapCore.h) |
| Other known OAuth issuers in Thunderbird | Yahoo, AOL, Fastmail, Mail.ru, Yandex, Seznam, Comcast, tb.pro — each with hard-coded client ID/endpoints, `usePKCE`, some with `clientSecret` and `redirectionEndpoint` (loopback http://localhost). | [OAuth2Providers.sys.mjs](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs) |

**OAuth shape for a hosted app:** Thunderbird, as a desktop/native app, legitimately embeds client IDs (and even public "secrets") and opens the system browser, with loopback or no redirect URI. A Cloudflare-hosted web app would register as a **web application client** (HTTPS redirect URI, confidential client, tokens held server-side) — matching what Google and Microsoft already require for registered web apps. Thunderbird's own IDs cannot be reused (the source says "register one for yourself"), so the owner registers their own Google project and Microsoft Entra app; Google offers several states short of full verification — *Testing* (external, ≤100 allowlisted test users, refresh tokens expire after 7 days), *Published–unverified* (any Google user but "Danger" warning UI and a hard 100-user cap), *Verified*, and *Internal* (inside one Workspace org, no verification needed) — plus the app-password escape hatch, so per-instance registration is one of several viable auth models, not the only one; the settled choice belongs to the Cloudflare ticket. [OAuth2Providers.sys.mjs](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs); [Microsoft Learn](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth); [Google: OAuth app state overview](https://developers.google.com/identity/protocols/oauth2/production-readiness/overview); [Google: Sign in with app passwords](https://support.google.com/mail/answer/185833?hl=en)

---

## 10. What "all features" could mean — capability catalogue

Grouped so the owner can pick a boundary. Two classes: **(A) approved requirements** — the behaviors the map already asks for — and **(B) undecided extras** — additional Thunderbird capabilities that are *not* required and need an explicit yes/no. Nothing in list B should be read as implicitly required.

**(A) Approved requirements (all present in Thunderbird):**

1. Multiple mailboxes (accounts) side-by-side; per-account folders, sync, junk, SMTP. [SUMO: Configuration Options for Accounts](https://support.mozilla.org/en-US/kb/configuration-options-accounts)
2. Read mail: header-first listing, body-on-demand, full sync where enabled. [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization)
3. Send/reply using the original address: identities per account, reply auto-picks the identity the sender used; per-identity From/signature/SMTP. [SUMO: Using Identities](https://support.mozilla.org/en-US/kb/using-identities)
4. Attachments in and out (standard MIME). [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization); [SUMO: Filelink](https://support.mozilla.org/en-US/kb/filelink-large-attachments)
5. Search across mailboxes: local FTS over synced bodies (Gloda) + header search over everything + optional server-side search. [SUMO: Global Search](https://support.mozilla.org/en-US/kb/global-search); [SUMO: Using Saved Searches](https://support.mozilla.org/en-US/kb/using-saved-searches)
6. Two-way read-state, folder/label, move, and deletion sync over IMAP (with provider-specific semantics per §4.2–4.3). [SUMO: IMAP vs POP3](https://support.mozilla.org/en-US/kb/difference-between-imap-and-pop3)
7. Accessible old history via bounded caching: sync caps (recent N days / max size) + per-folder sync opt-in/out + on-demand body fetch for unsynced mail. [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization)

**(B) Undecided extras — Thunderbird capabilities NOT covered by the approved requirements (each needs an explicit owner decision):**

8. Folder subscribe/unsubscribe UI; folder create/rename/delete propagation to server. (Basics of "folder sync" are arguably inside (A6); subscription management is the extra.)
9. Unified folders view + POP Global Inbox + cross-account Saved Search virtual folders — three different "unified" surfaces; the map does not yet define which one(s) "unified mailbox" means.
10. Threaded view, ignore thread/subthread, per-thread mass actions.
11. Tags/keywords (beyond read/star), Quick Filter bar, custom message columns.
12. Message filters/rules (new-mail/manual/after-send/archive triggers) and adaptive Bayesian spam filter + server-header trust.
13. Compose extras: drafts/templates folders, sent-copy rules, CC/Bcc defaults, quoting/signature placement, HTML compose, Send Later/Outbox queueing, return receipts, remote-content controls.
14. Filelink large-attachment offload to third-party storage add-ons.
15. POP3 accounts (leave-on-server, global inbox) — one-way protocol.
16. Account autoconfig via ISPDB/autoconfig XML vs. manual-only setup; custom OAuth issuer config; non-password auth methods (client-cert, GSSAPI, NTLM).
17. OpenPGP/S/MIME encryption and signing.
18. Import/export (mbox), Local Folders, mbox-vs-maildir storage choice.
19. Calendar (CalDAV), address books (CardDAV/LDAP), chat/feed/newsgroup accounts — outside mail scope; Thunderbird's Google consent asks for calendar/contacts scopes too, and its Exchange/Graph path has no calendar/address book yet. [OAuth2Providers.sys.mjs](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs); [SUMO: Thunderbird and Exchange](https://support.mozilla.org/en-US/kb/thunderbird-and-exchange)

---

## 11. Portability: desktop assumptions vs. browser/Cloudflare reality

**Facts about the Thunderbird side (verified in this pass):**

- Raw TCP/TLS sockets to IMAP(993/143)/POP3(995/110)/SMTP(587/465), long-lived connections with IDLE waits, TCP keepalive tuning (`mail.imap.tcp_keepalive.*`). [comm-central mailnews.js](https://searchfox.org/comm-central/source/mailnews/mailnews.js); [nsImapCore.h](https://searchfox.org/comm-central/source/mailnews/imap/src/nsImapCore.h)
- Persistent local profile: mbox/maildir message stores + per-folder `.msf` Mork databases + Gloda index + credential store + filter rules + identity config, all local files. [comm-central folder_storage.md](https://searchfox.org/comm-central/source/mailnews/base/docs/folder_storage.md); [SUMO: Glossary profile](https://support.mozilla.org/en-US/kb/glossary-terms-including-types-accounts)
- Continuous process: background sync, biff timers, IDLE respawn, offline queue, Outbox flushing — all assume a running process. [SUMO: IMAP Synchronization](https://support.mozilla.org/en-US/kb/imap-synchronization)
- OAuth UX: opens the system browser / external browser (`useExternalBrowser: true` for Google/Microsoft), loopback or no redirect, then holds refresh tokens locally. [OAuth2Providers.sys.mjs](https://searchfox.org/comm-central/source/mailnews/base/src/OAuth2Providers.sys.mjs); [SUMO: Thunderbird and Gmail](https://support.mozilla.org/en-US/kb/thunderbird-and-gmail)
- Local search index (Gloda) on local bodies; full-text impossible without a downloaded body. [SUMO: Global Search](https://support.mozilla.org/en-US/kb/global-search)
- OS services: system browser for OAuth; OS notifications; OS file dialog; NSS/client-cert store; crash handling; multi-process desktop app. (Documented across the account/OAuth pages above.)

**Unverified portability risks for a Cloudflare-hosted client (facts for the next ticket to test — these are reasonable inferences, not yet proven):**

- **Socket layer:** unverified whether Workers' outbound-networking capabilities (e.g., `connect()`/TCP sockets) can sustain an IMAP/SMTP session the way a desktop process does. If they cannot, the mail path would need a provider-API route (Gmail API, Microsoft Graph) or another transport — undecided until the Cloudflare ticket tests it. Until verified, treat every "client holds an IMAP connection" behavior — IDLE push, per-folder `SELECT`, CONDSTORE, MOVE — as *at risk*, not impossible.
- **Process model:** Thunderbird assumes a continuously running process for IDLE, biff timers, background sync, and Outbox flushing. Whether Workers/Durable Objects/alarms/queues can express that loop within free-tier limits is unverified; connection-cache limits (Gmail's documented 15-connection cap) are an architectural concern across invocations either way.
- **Storage:** Thunderbird assumes a local filesystem (mbox/`.msf`/Gloda). Mapping that to Workers-visible storage (R2/D1/KV/DO) and an index is unverified; "bounded cache" decisions (sync window, body-on-demand, subscription equivalents) become cost/latency tradeoffs rather than a disk-space preference.
- **Search:** Gloda semantics (stemming, facets, min-term) are reproducible in any FTS, but the "search everything instantly" promise depends on what was synced — the same tradeoff, sharper under free-tier limits.
- **OAuth:** browser-cookie/http-only-token storage rather than a local credential store; confidential-web-app registration rather than native app + loopback. Google's `mail.google.com/` restricted scope has multiple routes (Testing / Published-unverified / Verified / Internal per Workspace, plus app passwords) — settled choice belongs to the Cloudflare ticket, not assumed here.
- **Per-connection provider quirks:** Gmail delete=label-removal asymmetry, All-Mail duplication, SPECIAL-USE folder mapping, Outlook.com's off-by-default IMAP/POP toggle and suspicious-activity challenge, M365 admin-consent gates and SMTP AUTH default-off — all remain true regardless of where the client runs; a hosted app inherits every one of them.
- **JMAP:** no production JMAP path in Thunderbird today (source notes JMAP only as a possible future wire format); don't count on JMAP availability for the providers we target. [comm-central folder_storage.md](https://searchfox.org/comm-central/source/mailnews/base/docs/folder_storage.md)

---

## 12. Limits and unknowns

1. **IMAP keyword (tag) storage per provider is unverified.** Thunderbird tags map to IMAP keywords where the server supports them, but Gmail's documented folder-based label model and lack of explicit keyword support mean tag persistence on Gmail is *not* confirmed from a primary source this pass. Verify before promising two-way tag sync on Gmail.
2. **`X-GM-RAW` unused by Thunderbird** — zero comm-central hits; confirms Thunderbird does not leverage Gmail's native search syntax over IMAP, but does not prove the Gmail search gap is unfillable. Next ticket: test server-side `UID SEARCH X-GM-RAW`.
3. **Exact Gmail connection/UIDVALIDITY behavior under a hosted pool** unverified — only the documented 15-connection cap is primary-sourced.
4. **Provider message-size limits** exist but were not re-cited from a primary source this pass (numbers removed to avoid asserting approximate values); confirm exact ceilings per provider in the next ticket.
5. **Ignore-thread, per-message flags like "junk" scores, and some internal flags are local-only MDB state** — not confirmed to round-trip through IMAP flags/keywords for each provider.
6. **EWS/Graph feature gap in Thunderbird** (no address book/calendar/shared mailboxes/on-prem OAuth yet; email only) is current as of this writing but changing quickly (EWS→Graph migration October 2026–2027). [SUMO: Thunderbird and Exchange](https://support.mozilla.org/en-US/kb/thunderbird-and-exchange); [SUMO: EWS→Graph](https://support.mozilla.org/en-US/kb/microsoft-ews-graph-migration)
7. **Attachment-content search** — Gloda indexes message bodies; attachment-content indexing is not documented in the pages read; unverified.
8. **Thunderbird's unified-folders merge rules** (which folders coalesce: by type flag or by name) were verified from a forum answer and the global-inbox KB, not from source this pass; treat the exact coalescing rule as "good enough to model on, confirm in implementation."
9. **Sender-side "From must match authenticated account" restrictions** — Gmail's "Send mail as" aliases and Microsoft tenant From-domain policies are provider-side and untested here; a hosted app may hit the same walls Thunderbird users hit when an SMTP server rejects a mismatched From.

---

## 13. Sharp follow-on decisions for the owner

1. **Feature boundary** — §10 splits into approved requirements (A1–A7) and undecided extras (B8–B19). The owner needs to explicitly approve or drop each B row: folder-subscription UI, unified-view flavor(s), threading surface, tags/Quick Filter, filter rules, adaptive spam, compose extras, Filelink, POP3, autoconfig, non-password auth, E2E encryption, import/export, calendar/contacts.
2. **POP3 support?** One-way, server-destructive-by-default, no two-way anything. Recommend: exclude from v1 unless the owner has a POP-only mailbox that cannot be reached via IMAP.
3. **Gmail fidelity level** — Do we reproduce the folder-as-label illusion (one message, N views) or expose real labels? Reproducing Thunderbird semantics is simpler over plain IMAP but inherits the duplicate-copy problem in local cache; using `X-GM-EXT-1` (`X-GM-LABELS`, `X-GM-THRID`, `X-GM-RAW`) gives true label/thread/search fidelity but is Gmail-only work.
4. **Delete model default** — adopt Gmail's "archive ≠ delete" semantics per provider, or expose Thunderbird's four delete models verbatim? The safest default (match Google's recommendation) avoids accidental 30-day permanent deletion via Trash mapping.
5. **Protocol floor for Microsoft** — IMAP+SMTP OAuth2 only, or also Graph/EWS paths? Graph is the long-lived API and would bypass raw-IMAP portability risk entirely, but introduces a second abstraction surface.
6. **Unified-view definition** — Thunderbird merges *like-kind folders* (all Inboxes, all Drafts, …) across accounts, keeps per-account trees too, and adds cross-account Saved Search virtual folders. Which of the three does "unified mailbox" mean for this product?
7. **Sync window** — Thunderbird's default is sync-everything-with-per-folder-opt-out plus disk-space caps; free-tier will likely require sync-window + body-on-demand. Pick the default window and the reach-back behavior (server-side search over unsynced history?) explicitly.
8. **Threading model** — local header-based threading (References + subject) is provider-agnostic and reproducible anywhere; Gmail's X-GM-THRID would align threads with the Gmail web view. Choose per provider or uniform?
9. **OAuth app model** — one owner-registered Google project + Entra app per personal instance is allowed by the ticket and matches Thunderbird's "register your own" guidance, but it is not the only option: Google's Testing/Published-unverified/Verified/Internal states and app passwords are documented alternatives with different UX and longevity tradeoffs. Confirm which auth model(s) the owner accepts.
10. **On-prem/self-hosted Exchange** — out of scope unless the owner needs it; Thunderbird itself notes OAuth gaps there.

---

## 14. Source list

Mozilla SUMO (support.mozilla.org):
- Thunderbird and Gmail — https://support.mozilla.org/en-US/kb/thunderbird-and-gmail
- IMAP Synchronization — https://support.mozilla.org/en-US/kb/imap-synchronization
- Difference between IMAP and POP3 — https://support.mozilla.org/en-US/kb/difference-between-imap-and-pop3
- Configuration Options for Accounts — https://support.mozilla.org/en-US/kb/configuration-options-accounts
- Using Identities — https://support.mozilla.org/en-US/kb/using-identities
- Archived Messages — https://support.mozilla.org/en-US/kb/archived-messages
- Message Tags — https://support.mozilla.org/en-US/kb/message-tags
- Message Threading — https://support.mozilla.org/en-US/kb/message-threading-thunderbird
- Ignoring Threads — https://support.mozilla.org/en-US/kb/ignoring-threads
- Global Search (Gloda) — https://support.mozilla.org/en-US/kb/global-search
- Rebuilding the Global Database — https://support.mozilla.org/en-US/kb/rebuilding-global-database
- Quick Filter Toolbar — https://support.mozilla.org/en-US/kb/quick-filter-toolbar
- Using Saved Searches — https://support.mozilla.org/en-US/kb/using-saved-searches
- Message Filters — https://support.mozilla.org/en-US/kb/organize-your-messages-using-filters
- Filelink — https://support.mozilla.org/en-US/kb/filelink-large-attachments
- Junk/Spam — https://support.mozilla.org/en-US/kb/thunderbird-and-junk-spam-messages
- Automatic Account Configuration — https://support.mozilla.org/en-US/kb/automatic-account-configuration
- Manual Account Configuration — https://support.mozilla.org/en-US/kb/manual-account-configuration
- Account Hub — https://support.mozilla.org/en-US/kb/account-hub-thunderbird-desktop
- Custom OAuth — https://support.mozilla.org/en-US/kb/tb-custom-oauth
- Microsoft OAuth and Thunderbird in 2026 — https://support.mozilla.org/en-US/kb/microsoft-oauth-authentication-and-thunderbird-202
- Thunderbird and Exchange — https://support.mozilla.org/en-US/kb/thunderbird-and-exchange
- Microsoft EWS→Graph migration — https://support.mozilla.org/en-US/kb/microsoft-ews-graph-migration
- Unify POP accounts with a global inbox — https://support.mozilla.org/en-US/kb/unify-your-pop-email-accounts-global-inbox
- Glossary of Terms (account types, Outbox, Send Later, offline) — https://support.mozilla.org/en-US/kb/glossary-terms-including-types-accounts
- Forum answer on unified folders view — https://support.mozilla.org/en-US/questions/1566028
- Forum answer on global inbox — https://support.mozilla.org/en-US/questions/1094874

Microsoft (learn.microsoft.com / support.microsoft.com):
- Authenticate an IMAP, POP or SMTP connection using OAuth — https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth
- Deprecation of Basic authentication in Exchange Online — https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-basic-authentication-exchange-online
- POP, IMAP, and SMTP settings for Outlook.com — https://support.microsoft.com/en-us/office/pop-imap-and-smtp-settings-for-outlook-com-d088b986-291d-42b8-9564-9c414e2aa040

Google (developers.google.com / support.google.com):
- Gmail IMAP Extensions — https://developers.google.com/workspace/gmail/imap/imap-extensions
- Choose your IMAP email client settings for Gmail — https://support.google.com/mail/answer/78892?hl=en
- Transition from less secure apps to OAuth — https://support.google.com/a/answer/14114704
- Sign in with app passwords — https://support.google.com/mail/answer/185833?hl=en
- Restricted scope verification — https://developers.google.com/identity/protocols/oauth2/production-readiness/restricted-scope-verification
- OAuth app state overview — https://developers.google.com/identity/protocols/oauth2/production-readiness/overview

comm-central source (searchfox.org/comm-central):
- `mailnews/base/src/OAuth2Providers.sys.mjs` — OAuth issuers/scopes/endpoints
- `mailnews/imap/src/nsImapCore.h` — IMAP capability/auth flags
- `mailnews/imap/src/nsImapIncomingServer.cpp` — `useCondStore`, `SetIsGMailServer`
- `mailnews/imap/src/nsImapServerResponseParser.cpp` — `X-GM-EXT-1` capability detection
- `mailnews/db/msgdb/src/nsMsgDatabase.cpp` — threading prefs + `X-GM-LABELS` storage
- `mailnews/mailnews.js` — default prefs (strict/correct threading, TCP keepalive, biff)
- `mailnews/base/docs/folder_storage.md` — folder/database/store architecture, JMAP note

IETF RFCs (referenced by the above; not re-read line-by-line):
- RFC 3501 IMAP4rev1 / RFC 9051 IMAP4rev2; RFC 1939 POP3; RFC 5322 message format; RFC 2177 IDLE; RFC 4315 UIDPLUS; RFC 4978 COMPRESS; RFC 5161 ENABLE; RFC 5258 LIST-EXTENDED; RFC 6154 SPECIAL-USE; RFC 6851 MOVE; RFC 6855 UTF8=ACCEPT; RFC 7162 CONDSTORE; RFC 7628 OAUTHBEARER/XOAUTH2.
