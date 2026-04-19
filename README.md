# PANW Newsletter Generator

Monthly toolkit for building a PAN technical newsletter from:

- live NAM events scraped from `https://events.paloaltonetworks.com/`
- curated monthly content in `content/months/YYYY-MM.json`
- a reusable HTML renderer
- optional Gmail delivery using SMTP env vars

The project now has two jobs:

1. Scrape current event data into JSON / CSV snapshots.
2. Render a newsletter HTML file for a given month and optionally email it.

## Why scraping instead of a browser

The events page is a JavaScript-rendered SPA, but the event cards are backed by an internal JSON endpoint:

`https://events.paloaltonetworks.com/apps/pan/public/splash.getFilteredEvents.json`

This project calls that endpoint directly, which is much faster and more stable than browser automation.

## Install

Requires Node 18+.

```bash
cd panw-events-scraper
```

## Run

```bash
npm run scrape                 # pull latest NAM event data
npm run diff                   # compare latest event snapshot to previous snapshot
npm run generate:newsletter    # render newsletter for the current month
npm run email:newsletter       # email output/newsletter/latest.html
npm run monthly:newsletter     # scrape events + render newsletter
npm run monthly:newsletter:email
```

You can target a specific month:

```bash
node generate_newsletter.js --month=2026-04
```

## Monthly Content

Static / curated content lives here:

```text
content/
├── base.json
└── months/
    └── 2026-04.json
```

To build a new month:

1. Copy the previous month file in `content/months/`
2. Rename it to `YYYY-MM.json`
3. Update:
   - `didYouKnow`
   - `unit42`
   - `corporateBlogs`
   - `whatsNew`
4. Run `npm run monthly:newsletter`

## Output

Files land in `./output/`:

- `panw_events_nam.csv` — structured event export
- `panw_events_nam.json` — current normalized event data
- `newsletter_diff.md` — snapshot diff summary for event changes
- `snapshots/YYYY-MM-DD.json` — event snapshots used for diffing
- `newsletter/panw_newsletter_YYYY-MM.html` — rendered newsletter
- `newsletter/latest.html` — latest rendered newsletter
- `newsletter/latest.json` — latest render metadata

## Gmail Delivery

The email sender now prefers `gog` first, then falls back to the built-in Gmail paths.

Supported delivery modes:

1. `gog` CLI with existing Google auth
2. SMTP with an app password
3. Gmail API with Google OAuth refresh token
4. Google Workspace service account with domain-wide delegation

Supported env vars:

- `GOG_ACCOUNT` or `GOOGLE_EMAIL` (optional; only needed to force a specific `gog` account)
- `GMAIL_USER` or `SMTP_USER`
- `GMAIL_APP_PASSWORD` or `SMTP_PASS`
- `GMAIL_CLIENT_ID` or `GOOGLE_CLIENT_ID`
- `GMAIL_CLIENT_SECRET` or `GOOGLE_CLIENT_SECRET`
- `GMAIL_REFRESH_TOKEN` or `GOOGLE_REFRESH_TOKEN`
- `GOOGLE_SERVICE_ACCOUNT_EMAIL` or `GMAIL_SERVICE_ACCOUNT_EMAIL`
- `GOOGLE_PRIVATE_KEY` or `GMAIL_PRIVATE_KEY`
- `GOOGLE_IMPERSONATED_USER` or `GMAIL_IMPERSONATED_USER`
- `GMAIL_TO` / `NEWSLETTER_TO` (optional, defaults to the sender)
- `GMAIL_FROM` / `SMTP_FROM` (optional)
- `GOOGLE_EMAIL` (optional sender fallback for OAuth mode)

Example:

```bash
gog auth add you@gmail.com
export GMAIL_TO="you@gmail.com"
npm run email:newsletter
```

OAuth example:

```bash
export GMAIL_CLIENT_ID="..."
export GMAIL_CLIENT_SECRET="..."
export GMAIL_REFRESH_TOKEN="..."
export GOOGLE_EMAIL="you@gmail.com"
export GMAIL_TO="you@gmail.com"
npm run email:newsletter
```

Workspace service-account example:

```bash
export GOOGLE_SERVICE_ACCOUNT_EMAIL="mailer@your-project.iam.gserviceaccount.com"
export GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
export GOOGLE_IMPERSONATED_USER="you@yourcompany.com"
export GMAIL_TO="you@yourcompany.com"
npm run email:newsletter
```

You can also override at runtime:

```bash
node send_newsletter.js --file=./output/newsletter/latest.html --to=you@gmail.com
```

## Monthly Schedule

### macOS (launchd)

Create `~/Library/LaunchAgents/com.john.panw-events.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.john.panw-events</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>-lc</string>
    <string>cd /Users/johnshelest/Code/panw-events-scraper &amp;&amp; /usr/local/bin/npm run monthly:newsletter:email &gt;&gt; ./output/cron.log 2&gt;&amp;1</string>
  </array>
  <key>StartCalendarInterval</key>
  <dict>
    <key>Day</key><integer>1</integer>
    <key>Hour</key><integer>7</integer>
    <key>Minute</key><integer>0</integer>
  </dict>
</dict>
</plist>
```

Load it:
```bash
launchctl load ~/Library/LaunchAgents/com.john.panw-events.plist
```

Runs at 7am on the 1st of every month.

### Linux cron

```
0 7 1 * * cd /path/to/panw-events-scraper && /usr/bin/npm run monthly:newsletter:email >> ./output/cron.log 2>&1
```

## Known limitations

- **Monthly editorial content is still curated**: `content/months/YYYY-MM.json` is the source of truth for research, blogs, and "What's New" cards.
- **Gmail requires an app password**: standard account password auth will usually fail.
- **The event categories are heuristic**: the renderer groups live events by title/format patterns, so edge cases may need rule tweaks in `generate_newsletter.js`.
- **The sender is SMTP-only**: it uses Gmail SMTP from env vars, not the Gmail API.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Newsletter month file missing | Create `content/months/YYYY-MM.json` before rendering |
| `email:newsletter` fails auth | First confirm `gog auth add you@gmail.com --services gmail`; if needed set `GOG_ACCOUNT` to force the account; otherwise use SMTP, Google OAuth, or service-account vars |
| Event links are blank | Re-run `npm run scrape`; the scraper now prefers `redirectTarget` from the PANW API |
| Diff output looks wrong | Old snapshots may predate the `registrationUrl` fix; current diffing now falls back to `eventId` |

## Files

```
panw-events-scraper/
├── content/
│   ├── base.json
│   └── months/
│       └── 2026-04.json
├── generate_newsletter.js   # renders newsletter HTML
├── send_newsletter.js       # sends HTML via Gmail SMTP
├── scrape_panw_events.js    # main scraper
├── diff_snapshots.js        # snapshot diff + markdown summary
├── package.json
├── README.md
└── output/                   # created on first run
    ├── panw_events_nam.csv
    ├── panw_events_nam.json
    ├── newsletter_diff.md
    ├── newsletter/
    │   ├── latest.html
    │   └── panw_newsletter_2026-04.html
    └── snapshots/
        ├── 2026-04-17.json
        ├── 2026-05-01.json
        └── ...
```
# panw-events-scraper
