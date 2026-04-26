# the-followup

A small prototype that ingests meeting notes, parses action items with named owners and due dates, and publishes them to a static dashboard.

Live: <https://srkamp.github.io/the-followup/>

## What it does

1. You drop meeting notes (`.md` or `.txt`) into `inbox/`.
2. You ask Cowork to ingest them. Cowork sends each note to Claude with a parsing prompt and gets back structured JSON: meeting metadata plus a list of action items with owner, description, and due date.
3. Cowork appends to `data.json` (the single source of truth).
4. `index.html` reads `data.json` on page load and renders three views: Master, By person, and Due in 3 days.
5. Reassignment in the dashboard updates `data.json`; pushing to GitHub publishes the change to GitHub Pages.

## File layout

```
the-followup/
├── index.html        # single-file dashboard (vanilla HTML/CSS/JS)
├── data.json         # source of truth — meetings + action items
├── people.json       # allowlist for owner auto-resolution
├── inbox/            # drop new meeting notes here
│   ├── 2026-04-22_acme-travel-q2-review.md
│   └── 2026-04-23_internal-sales-marketing-sync.md
└── README.md
```

## Parsing rules

These are enforced by Cowork during ingestion:

- **Owner allowlist.** `people.json` is a strict allowlist. Each entry has a canonical `name` and optional `aliases` (single-token first names). Cowork only auto-resolves names that match the allowlist exactly or via an alias. Anything else flags as `UNRESOLVED: <name as written>` and surfaces in the dashboard for manual fix.
- **Ambiguous owners.** Phrases like "the team will follow up" or "someone needs to handle" produce an action item with `owner = "UNASSIGNED"` and a flag in the dashboard.
- **Default due date.** If no due date is mentioned, default to **3 business days** from the meeting date (Mon–Fri, skipping weekends).
- **Relative dates.** "by end of week" resolves to the Friday of the meeting week. "within two weeks" = meeting date + 14 calendar days. "next Friday" is interpreted as the Friday of the *following* week unless the same-week Friday is more than a couple of days away (this is judgment-call territory; flag it in the dashboard's `dueDateNote` so the user can override).
- **Transparency.** Every action item carries `dueDateSource` (`explicit`, `explicit-relative`, `default-3bd`) and a free-text `dueDateNote` so you can see *why* a date was chosen.

## Workflow

### Adding a new meeting

```
# 1. drop the note in
cp ~/Downloads/2026-05-01_meeting.md inbox/

# 2. ask Cowork to process it: "ingest the new file in inbox/"
#    Cowork parses, writes to data.json, leaves the inbox file alone.

# 3. push
git add . && git commit -m "Ingest 2026-05-01 meeting" && git push
```

### Reassigning an item

The dashboard is static (GitHub Pages), so reassignments live in your browser session until you commit them:

1. Open the dashboard, change the owner dropdown on a row.
2. The "unsaved changes" bar appears at the top. Click **Download updated data.json**.
3. Replace `data.json` in the repo with the downloaded file.
4. `git add data.json && git commit -m "Reassign foo to bar" && git push`.

A future version could automate the push via a small local helper script (see "Design-only" below).

## Running locally

```
cd the-followup
python3 -m http.server 8000
# open http://localhost:8000
```

Anything that serves the folder as static files works (`npx serve`, `caddy`, etc.). Don't open `index.html` directly via `file://` — `fetch("data.json")` won't work without an HTTP origin.

## Default seed data

The two files in `inbox/` are sample meeting notes that ship with the prototype so the dashboard isn't empty on first run. They produce 10 action items, including:

- 7 cleanly resolved owners
- 2 ambiguous owners ("Team to follow up", "Someone needs to handle") flagged as `UNASSIGNED`
- 1 single-name reference ("Samson") resolved via the allowlist alias to "Samson Kampler"
- A mix of explicit dates ("by May 1"), explicit-relative ("by end of week", "within two weeks", "by next Friday"), and 3-business-day defaults

## What's built (v1)

- Ingestion from a local folder
- Parsing with Claude (via Cowork)
- JSON storage
- Three-view HTML dashboard with sort, search, and reassignment UI
- Mark-done toggle
- Manual download/replace publish loop

## Design-only (not built in v1)

These are documented as future integrations:

- **Google Drive connector.** Real-time pull from a watched Drive folder instead of a local `inbox/`. Cowork already has a Drive MCP available; the only piece missing is a small daemon that polls and shells the result into Cowork.
- **Auto-push on reassignment.** A tiny local helper (`publish.sh` or `publish.py`) the user runs once that watches `data.json` for changes and runs `git add data.json && git commit && git push`. Static Pages can't push from the browser, so this needs a host-side process.
- **Slack notifications.** Post to a channel when new action items are created and when items hit T-1 day. Needs a Slack MCP or webhook.
- **HubSpot logging.** For client-facing meetings (`type === "external"`), log action items to the HubSpot deal record.
- **Asana sync.** For internal owners, mirror action items as Asana tasks; mark complete bidirectionally.
- **Multi-user.** Right now `people.json` is a single hardcoded allowlist. Multi-user would mean per-user views, auth, and per-org allowlists.

## Resume framing

> Built a prototype that ingests meeting notes, parses action items with named owners and due dates, and publishes them to a dashboard. Auto-publishes via GitHub Pages. Designed extensibility for Slack, HubSpot, and Asana integration.
