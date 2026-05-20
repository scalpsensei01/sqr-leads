# Shingle Roof Renewal — Show Quote Capture

Single-file web app for capturing roofing leads at home shows/events. No backend, no Telegram bot — runs entirely in the browser with localStorage.

## Deploy to Vercel (2 minutes)

```bash
# 1. Go to project folder
cd /mnt/d/AI/projects/show-quote-capture

# 2. Push to GitHub (create repo first, or use existing)
git init
git add index.html README.md
git commit -m "v2 — no Telegram, exports, validation"
git branch -M main
git remote add origin https://github.com/YOURNAME/sqr-leads.git
git push -u origin main

# 3. On Vercel dashboard:
#    - Import GitHub repo
#    - Framework: Other (static)
#    - Deploy
#    - Done — you get a https:// URL
```

## Features (v2)

- **Lead capture form** — name, phone, email, address, roof type, urgency, source, notes
- **Phone validation** — requires 10 digits, warns on duplicates
- **ZIP validation** — requires 5 digits
- **Roof quote calculator** — 2 modes (dimensions or living area) + pitch/garage/overhang
- **Quote storage** — $1.65/sqft rate, $250 deposit, frozen quote object with timestamp
- **Status pipeline** — New → Contacted → Quoted → Follow-up → Closed Won/Lost
- **Export** — JSON backup + CSV for spreadsheets
- **Copy quote to clipboard** — with fallback for older browsers
- **Searchable leads dashboard**
- **Works offline** — localStorage persists between sessions
- **Unsaved form warning** — beforeunload handler

## Pricing Config

Edit the `CONFIG` block at the top of `index.html`:

```js
const CONFIG = {
  RATE_PER_SQFT: 1.65,   // change to your rate
  DEPOSIT_AMOUNT: 250,    // change to your deposit
};
```

## How to Use at a Home Show

1. Open the Vercel URL on a tablet/phone
2. Tap **Capture** — fill customer info, hit **Save Lead**
3. On success screen, tap **Enter Roof Sq Ft**
4. Use the calculator (enter house dimensions or living area) or type sq ft manually
5. Tap **Calculate Roof & Quote** → **Save to Lead**
6. Tap **Copy Quote** to paste into your other Telegram bot or text the customer
7. Update status as you progress: **Contacted** → **Quoted** → etc.
8. At end of day, go to **Leads** → **Export CSV** for your records

## Data Safety

- All data stays in the browser's localStorage
- Export JSON after every show as backup
- 500 lead cap (oldest auto-trimmed)
- Schema v2 with migration — old data auto-upgrades

## Files

- `index.html` — entire app in one file (~45 KB)
- No build step, no dependencies, no backend

## Security Notes

- App is hosted on HTTPS (Vercel)
- No API tokens in client code
- No user authentication — rely on URL obscurity + only 4 trusted users
- For extra lock, add Vercel password protection in project settings
