# Engineer Review — Show Quote Capture (Alex)
**Artifact:** `index.html` (single-file web app)  
**Date:** 2026-05-19  
**Reviewer:** Alex (Full-Stack Engineer)

---

## TOP FINDINGS (8)

### 1. TELEGRAM BOT TOKEN EXPOSED IN CLIENT-SIDE CODE
**Severity: CRITICAL**

The `CONFIG` object is client-side JavaScript. A Telegram bot token in plaintext HTML/JS is trivial for any user to extract via View Source or DevTools. Once leaked, an attacker can read all messages sent to the bot, impersonate the bot, and use Telegram's API to spam or phish users.

**Fix:** Move `sendTelegram()` to a backend (e.g., a tiny Cloudflare Worker, Vercel Edge Function, or simple Node/Express endpoint). The client POSTs the lead data to your endpoint; the server holds the token securely (env variable) and forwards to Telegram. Never ship tokens to browsers.

---

### 2. NO INPUT SANITIZATION — XSS VIA LEAD DATA
**Severity: HIGH**

The `esc()` function sanitizes text for safe HTML insertion, but it is NOT applied consistently:
- Line 383: `renderSuccess(lead)` sets `succ-name` via `textContent` — safe.
- Line 388: `document.getElementById('succ-name').textContent = lead.name` — safe (textContent).
- BUT Line 389: `succ-fixr.href = ...encodeURIComponent(lead.zip)` — `lead.zip` is user-controlled text injected into an href. Malicious ZIP like `javascript:alert(1)` would execute (though `encodeURIComponent` does NOT block `javascript:` schemes).
- Line 390 `onOpenFixr()` builds a URL with `lead.zip || '33073'` — same vector.

More critically, the `esc()` function uses `document.createElement('div').textContent = s; return d.innerHTML`. This is a correct sanitizer when used, but the developer has multiple paths where raw values hit the DOM (e.g., in the FIXR URL construction above). The risk is LOW because the attacker would need to poison their own localStorage, but if data is ever synced/shared, this becomes a stored XSS vector.

**Fix:** Validate ZIP as numeric (`/^\d{5}(-\d{4})?$/`) before any URL construction. Never put raw user strings into `href` or `src` without scheme validation. Consider using `URL` constructor instead of string interpolation for URLs.

---

### 3. NO FORM VALIDATION BEYOND HTML5 `required`
**Severity: MEDIUM**

- Phone accepts any string — no format validation (`(954) 555-1234`, `abc`, etc.)
- Email accepts empty string (HTML5 `type="email"` helps but `noValidate` is not set — actually this is fine, but client-side email regex is weak).
- ZIP accepts any text — no 5-digit validation.
- No duplicate detection: saving the same lead 5 times creates 5 entries silently.
- No confirmation on discard: if user partially fills form and navigates, data is lost.

**Fix:** Add regex validation for phone (naïve: `/[\d\s\-\(\)\.]+/`) and ZIP (`/^\d{5}(-\d{4})?$/`). Add a `beforeunload` handler to warn on unsaved form changes. Add duplicate detection by phone number before unshift.

---

### 4. NO DATA PERSISTENCE OR BACKUP STRATEGY — TOTAL DATA LOSS ON BROWSER CLEAR
**Severity: MEDIUM**

All state lives in `localStorage` under key `sqr_leads`. No export, no cloud sync, no backup. If the user clears cookies/site data, or if this is a PWA and iOS clears localStorage on low storage, all leads vanish permanently. This is a business-critical CRM use case.

**Fix:** At minimum, add an "Export to JSON / CSV" button so the user can backup. Better: POST leads to a backend DB. The Telegram integration already proves network capability — use it to also persist to a real database.

---

### 5. MEMORY LEAK AND UNBOUNDED localStorage GROWTH
**Severity: MEDIUM**

`getLeads()` and `saveLeads()` re-parse and re-serialize the entire array on every keystroke (search input triggers `renderDashboard()` which calls `getLeads()`). There is no maximum lead count. Uncapped localStorage growth causes:
- Performance degradation on load/render (O(n) scan for every search keystroke)
- localStorage 5MB limit hit, at which point `setItem` throws and the app breaks
- No pruning or archival strategy for old "Closed Lost" leads

**Fix:** Cap the array (e.g., keep last 500 leads). Add pagination or virtual scrolling for the dashboard list. Use `sessionStorage` or an in-memory cache for search filtering to avoid re-parsing JSON on every keystroke.

---

### 6. HTML SEMANTICS & ACCESSIBILITY DEFICIENCIES
**Severity: MEDIUM**

- `<html>` lacks `<!DOCTYPE html>` declaration (line 1 starts with `<html>`, not `<!DOCTYPE html>`).
- No `<main>`, `<header>`, `<nav>`, or `<section>` landmarks — screen readers get no structure.
- Form inputs have no `id`/`for` linkage with labels (line 554: `<label>Full Name</label><input name="name">`). Labels are not programmatically associated; screen readers may miss them.
- No `aria-live` region for toasts — screen reader users won't hear toast messages.
- No `aria-current="page"` on active nav button.
- The page relies entirely on `display:none` for routing; this is okay but `hidden` attribute or `aria-hidden` should accompany.
- Touch targets: nav buttons use `flex: 1` with `min-width: 110px` — acceptable but borderline for WCAG 2.5.5 (44x44px). The CSS `min-width: 110px` and padding give ~110x52px targets which is fine.

**Fix:** Add `<!DOCTYPE html>`. Replace label/input pairs with explicit `for` attributes (`<label for="name">`). Wrap nav in `<nav>`, content in `<main>`. Add `role="alert"` or `aria-live="polite"` to toast.

---

### 7. CROSS-SITE LINKING OPENS IN SAME ORIGIN CONTEXT + NO REL ATTRIBUTES
**Severity: LOW**

- Line 389/396: `window.open(url, '_blank')` opens FIXR in a new tab, but there is `window.opener` access risk. Not using `rel="noopener noreferrer"` on the `<a>` (line 608) leaves the app vulnerable to tab-napping if FIXR is compromised or the URL is hijacked.
- Line 608: `<a id="succ-fixr" href="#" target="_blank">` — the `href="#"` is replaced by JS, but the element doesn't get `rel="noopener noreferrer"`.

**Fix:** Add `rel="noopener noreferrer"` to all external links. For `window.open()`, use `window.open(url, '_blank', 'noopener,noreferrer')` (though `window.open` with `_blank` usually implies `noopener` in modern browsers, explicit is better).

---

### 8. NO ERROR HANDLING FOR CLIPBOARD API
**Severity: LOW**

Line 434: `navigator.clipboard.writeText(text).then(() => showToast('Copied to clipboard'));`
- No `.catch()` handler. On browsers/contexts where clipboard access is denied (HTTP site, iframe without permissions, mobile Safari in some modes), this promise rejects silently. The user sees nothing.

**Fix:** Add `.catch(() => showToast('Copy failed — please copy manually'))` or fallback to a hidden `<textarea>` + `document.execCommand('copy')`.

---

## ADDITIONAL OBSERVATIONS (BELOW TOP 8)

### Missing SEO / Meta Tags
No Open Graph, no favicon, no description meta. Fine for an internal tool, but bad if anyone shares/bookmarks it.

### No Service Worker / Offline Strategy
For a trade-show tool used on spotty venue WiFi, a minimal service worker with offline fallback would prevent complete failure when connectivity drops.

### `user-scalable=no` Accessibility Concern
Line 4: `maximum-scale=1.0, user-scalable=no` violates WCAG 1.4.4 (Resize text up to 200%). Users with low vision cannot zoom. Recommended: remove `user-scalable=no` and `maximum-scale=1.0`.

### Floating-Point Math for Currency
`sqft * CONFIG.RATE_PER_SQFT` (line 407) can produce JS float artifacts like `$4,125.000000000001`. The `formatCurrency` helper uses `toLocaleString` with `minimumFractionDigits: 2` which rounds to 2 decimals, so this is largely masked, but direct reads of `lead.quote` elsewhere could show the raw float.

### Dead / Unused CSS
`.btn:active` is defined twice (lines 160 and 169) with different properties. No functional bug, just maintenance debt.

### Status Class Mapping Fragility
Line 461: `const statusClass = 'status-' + l.status.toLowerCase().replace(/\s+/g, '');` relies on the exact strings matching CSS classes. If a new status is added in JS but not CSS, it silently gets no styling. No fallback class/default styling.

---

## SUMMARY TABLE

| # | Issue | Severity | Effort to Fix |
|---|-------|----------|---------------|
| 1 | Telegram token exposed client-side | CRITICAL | Medium (needs backend) |
| 2 | XSS via unvalidated ZIP in href | HIGH | Low |
| 3 | No input validation / duplicates | MEDIUM | Low |
| 4 | Total data loss risk (localStorage only) | MEDIUM | Medium |
| 5 | Unbounded localStorage growth | MEDIUM | Low |
| 6 | Accessibility / semantic HTML gaps | MEDIUM | Low |
| 7 | Tab-napping via `window.open` / `<a>` | LOW | Trivial |
| 8 | Clipboard API unhandled rejection | LOW | Trivial |

---

*Reviewed independently. No other reviews consulted.*
