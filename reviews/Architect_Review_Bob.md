# Architecture Review: Show Quote Capture

**Reviewer:** Bob (System Architect)
**Date:** 2026-05-19
**Artifact:** `/mnt/d/AI/projects/show-quote-capture/index.html`
**Adversarial Stance:** Yes — structural flaws, scalability traps, security risks, and data-model issues below.

---

## Findings (8 Total)

| # | Topic | Severity | Finding | Fix |
|---|-------|----------|---------|-----|
| 1 | **Security Model** | Critical | **Telegram bot token lives in client-side JS.** Any user can open DevTools, copy `CONFIG.TELEGRAM_BOT_TOKEN`, and abuse the bot (spam, revoke, impersonate). The token is also sent in plaintext over the wire in the fetch URL. | Move Telegram API calls to a backend proxy or serverless edge function (Cloudflare Worker, Vercel Edge, Supabase Edge Function). The client should POST `{chat_id, text}` to your own endpoint; the endpoint holds the real token. |
| 2 | **Security Model** | Critical | **PII stored in plaintext `localStorage`.** Names, phone numbers, email addresses, and physical addresses are written as unencrypted JSON. localStorage is accessible to any JS running on the origin and survives browser-close. If the tablet is stolen or shared, all customer data is trivially extractable. | Encrypt lead data at rest using `SubtleCrypto` (AES-GCM) with a device-specific key, or — better — persist to a backend with auth. Alternatively, use IndexedDB with the same encryption layer. |
| 3 | **Business Continuity** | Critical | **Single-point-of-failure data store.** Everything lives in one browser's `localStorage`. There is zero backup, export, or sync. Device loss, browser cache clear, iOS Safari's 7-day inactivity purge, or a toddler mashing "clear data" = total loss of all show leads. | Implement automatic export: write a JSON backup to device Downloads on every save, and/or POST to a lightweight backend (Supabase, Firebase, a simple SQLite backend on a laptop) after each lead capture. Provide a visible "Back Up Now" button. |
| 4 | **App Architecture** | High | **Single-file architecture is a maintenance time bomb.** ~30 KB of inline CSS, HTML string literals inside JS, and mixed business logic. Adding features (appointment scheduling, photo capture, multi-event tracking) will create merge conflicts, bloat the file past readability, and make testing nearly impossible. | Split into separate files: `styles.css`, `app.js`, `components/*.js`, and HTML markup using templates or `template` tags. Even vanilla JS should use ES modules (`import`/`export`) loaded via `<script type="module">`. |
| 5 | **Offline / Resilience** | High | **No offline-first strategy.** While data capture itself works offline because it's client-side, Telegram sends fail silently with only a toast. There is no service worker, no background sync queue, and no retry mechanism. A spotty venue Wi-Fi means reps think they sent leads but nothing actually reached Telegram. | Add a Service Worker that caches the shell for instant reload. Use Background Sync API (or a simple in-memory retry queue with exponential backoff) for Telegram outbound messages. Store failed sends in IndexedDB and retry when connectivity returns. |
| 6 | **Data Model** | High | **No schema versioning or migration logic.** `sqr_leads` stores raw objects with no `_schemaVersion`. Adding a new field (e.g., `photoUrls`, `assignedRep`, `eventId`) will yield `undefined` on older records, causing UI crashes or NaN in calculations. | Add `_schemaVersion: 1` to every lead. On `getLeads()`, detect version mismatch and run a migration function that backfills defaults and re-saves. Better yet, define a schema object and validate against it on load/write. |
| 7 | **Deployment / Host** | Medium | **No HTTPS means broken APIs and blocked fetches.** Opening an HTML file directly (`file://`) or on an unsecured local network will cause CORS problems with Telegram's API and prevent geolocation, camera, and some modern APIs. It also trains users to accept "unsecure site" warnings. | Host on a free static CDN with HTTPS: Cloudflare Pages, Netlify, GitHub Pages, or Vercel. The domain can be password-protected or restricted by IP for event-day use. Serve via `https://` so all APIs and caches work correctly. |
| 8 | **Security Model** | Medium | **No access control or multi-user isolation.** Anyone who picks up the tablet sees all leads, can change statuses, and sends quotes from the same Telegram bot. There is no audit trail of which rep took which action. | Add a lightweight PIN code or rep selector at app start. Log actions with `repId` and timestamp. In Telegram messages, prefix the sender name. If multi-tablet, each rep should have a local alias so data can later be reconciled. |

---

## Deep-Dive: Topics

### 1. App Architecture
The pattern is a "vanilla SPA in one file." That is fine for a weekend prototype, but this is being deployed at a home show where money changes hands. The file mixes:
- Presentation markup (HTML strings inside `init()`)
- Styling (`:root` CSS variables + 200+ lines of inline CSS)
- Business logic (quote math, status machine, Telegram API calls)
- Data access (`localStorage` read/write scattered across handlers)

**Verdict:** Split it. At minimum: `index.html` shells the layout, `styles.css` handles theming, `app.js` orchestrates routing, `store.js` handles persistence, `telegram.js` wraps the outbound API, `components/` contains reusable renderers. Use ES modules to enforce boundaries.

### 2. Data Model
Current model per lead:
```js
{ id, name, phone, email, address, city, state, zip, notes,
  source, roofType, urgency, sqft, quote, status, createdAt }
```

**Problems:**
- No schema version, no validation, no constraints.
- `quote` is a computed value stored as state; if `RATE_PER_SQFT` changes later, historical quotes are silently altered on recalculation (luckily the code does not recalculate on load, but there's no explicit protection).
- Status is a free-text string with no state machine enforcement — any JS console can set `status = 'Hacked'`.

**Recommendation:** Freeze historical quotes and store rate metadata:
```js
quote: { rate: 1.65, sqft: 2400, total: 3960.00, deposit: 250, computedAt: "..." }
```
Add a status enum and transition rules (e.g., "Quoted" can only follow "Contacted").

### 3. Security Model
Two critical vectors, already summarized in findings 1 and 2. An additional concern: the app pulls in external calculators (`fixr.com`) via plain links. If that domain were compromised or if a man-in-the-middle attack were mounted on the venue Wi-Fi, reps could be redirected to a phishing form. Mitigation: pin the URL or embed the calculator flow inside a sandboxed iframe, or skip third-party calculators entirely and bring sqft estimation into the app.

### 4. Offline / Resilience
The app is "accidentally offline" for reads/writes, but "intentionally offline" for sync. At a trade show, Wi-Fi is notoriously flaky. The correct pattern is:
1. Service Worker caches app shell (instant load).
2. Lead capture writes to IndexedDB immediately.
3. A sync queue attempts Telegram/API sends; failures are retried.
4. UI shows a persistent "synced / pending / failed" indicator per lead.

### 5. Deployment Options
Currently: open HTML on a tablet. Better options:
- **Static host + QRCode:** Host on Cloudflare Pages (free, HTTPS, global CDN). Print a QR code on a stand; reps scan and open. No install needed.
- **PWA mode:** Add a Web App Manifest and Service Worker so the tablet can "Add to Home Screen" and run fullscreen like a native app.
- **Localhost fallback:** If the venue has no internet, run `npx serve` from a laptop on the local network, or use an `ngrok` tunnel.

### 6. Extensibility
Adding features to a single file is painful. If Daniel asks for:
- **Photo capture** (roof images): Requires `<input capture="environment">` + blob storage. In a single file, base64 strings bloat `localStorage` (5 MB limit danger).
- **Multi-event tracking** (different home shows): Requires an `eventId` field and UI changes scattered across every page.
- **Offline quote PDF generation:** Requires a PDF library inlined or loaded via CDN, further bloating the file.

With modular ES modules and a component pattern, each feature is an isolated module. Use `import` and a simple event bus or `Map`-based store.

### 7. Business Continuity
The most likely failure modes at a show:
- **Tablet dies / battery runs out:** No backup = lost day. Fix: auto-export JSON after every save; sync to a laptop on the same network.
- **Browser crashes / OS update:** localStorage wiped. Fix: Service Worker + IndexedDB + background sync.
- **Telegram bot blocked / rate-limited:** No fallback notification channel. Fix: add a secondary webhook target (email or Slack) and a visible retry queue in the UI.

---

## Summary Table

| Area | Grade | Primary Risk |
|------|-------|--------------|
| App Architecture | D+ | Single-file monolith, no module boundaries |
| Data Model | D | No schema, versioning, or integrity |
| Security Model | F | Exposed token + plaintext PII |
| Offline Strategy | D+ | Accidentally offline, no sync/recovery |
| Deployment | C | Works, but not robust for live events |
| Extensibility | D | New features require rewriting the monolith |
| Business Continuity | F | Zero backup; one accident = total data loss |

---

## Recommended Immediate Actions (in order)

1. **Move Telegram calls server-side.** This is non-negotiable; an exposed bot token is an immediate security failure.
2. **Encrypt localStorage and add a JSON export.** Protect customer PII and give reps a tangible backup file.
3. **Split into ES modules.** Start with `store.js`, `telegram.js`, and `components/LeadForm.js`.
4. **Add schema versioning.** A single `_schemaVersion` field and a 10-line migration function buys future compatibility.
5. **Deploy via HTTPS static host.** Cloudflare Pages or Netlify, free tier is sufficient.
6. **Add a Service Worker + sync queue.** Retry Telegram sends upon reconnection.
7. **Add a rep PIN / alias.** Enable accountability if multiple reps share a tablet.

---

*Review complete. Architecture recommendation: treat this as a prototype. The next revision should be a modular PWA with a lightweight backend proxy and encrypted at-rest storage.*
