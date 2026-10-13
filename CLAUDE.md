# Rocks Progress Scorecard

Live widget embedded in Notion that pulls departmental rock data from the Notion API and displays a branded progress dashboard. Companion project to the Orbit visualization (both are Notion-embedded widgets).

## Architecture

```
Notion embed (/embed block)
  -> index.html on GitHub Pages
    -> Cloudflare Worker (API proxy)
      -> Notion API (reads Departmental Quarterly Rocks database)
```

Three components, two files:

| Component | Location | File |
|-----------|----------|------|
| Frontend widget | GitHub Pages: `mariaofmars/rocks-scorecard` | `index.html` |
| API proxy | Cloudflare Workers: `rocks-scorecard-api.maria-011.workers.dev` | `worker.js` (deployed via Cloudflare dashboard editor) |
| Data source | Notion: Mars Men workspace | Database ID `c00a913d20fd44b19a30731444082dff` |

Live URLs:
- Widget: `https://mariaofmars.github.io/rocks-scorecard/`
- Worker: `https://rocks-scorecard-api.maria-011.workers.dev`

## Notion Database

**Database:** Departmental Quarterly Rocks
**Database ID:** `c00a913d20fd44b19a30731444082dff`
**Integration:** "Rocks Scorecard" (internal, read-only)
**API token:** stored as Cloudflare Worker secret `NOTION_API_KEY` (never in client code)

### Properties used by the widget

| Property | Type | Values |
|----------|------|--------|
| Rock | Title | Rock name |
| Team | Select | Growth, eCommerce & CRO, Retention, CX, Creative, Partnerships, Operations, Finance, Marketplaces, Brand, Legal, People |
| Status | Status | See scoring table below |
| Quarter | Select | Q1, Q2, Q3, Q4 |
| Year | Number | 2026, 2027, etc. |
| Owner | Rich text | Rock owner name |
| Company Priority | Select | P1 Channel Expansion, P2 Net Subscriber Growth, P3 Launch Ignite, P4 AI / Automations Rollout |

### Status scoring

Status names are case-sensitive and must match the Notion database exactly, including spaces around slashes.

| Status | Score | Notes |
|--------|-------|-------|
| Not Started | 0 | |
| Scoping | 12.5 | |
| Planning | 25 | |
| Kickoff | 37.5 | |
| In Progress | 50 | Was previously "In build" |
| Review / Feedback | 62.5 | |
| Finalizing / QA | 75 | |
| Launched / Implemented | 87.5 | |
| Complete | 100 | Was previously "Done" |
| On Hold | 25 | Flat stalled score |
| Blocked | 25 | Flat stalled score |
| Dropped | removed | Filtered out of the scorecard entirely: no card, no counts, no breakdown bar |
| Missed | excluded | Not counted in average, but still shown in counts and the breakdown bar |

Dropped and Missed are handled differently (Maria, 2026-10-13): a Dropped rock is removed from the scorecard entirely, including the total rock count, as if it never existed. A Missed rock still appears in the status breakdown bar and total rock count (footer shows "N missed"), but is excluded from the score calculation so it doesn't drag the department average down.

### Breakage risk

If someone renames, removes, or adds statuses in the Notion database, the widget will silently score unrecognized statuses as 0. This has happened once already (In build -> In Progress, Done -> Complete). When the scorecard looks wrong, check the database schema against the `SCORES` and `S_COLORS` objects in `index.html`.

## Widget (index.html)

Single self-contained HTML file. No build step, no dependencies beyond Google Fonts (JetBrains Mono).

### Key code sections

**`SCORES` object (~line 334):** Maps status names to numerical scores. Update here when statuses change.

**`EXCLUDED_STATUSES` array (~line 341):** Statuses excluded from score calculation. Rocks with these statuses still display but don't affect averages.

**`S_COLORS` object (~line 343):** Maps status names to hex colors for the status breakdown mini-bar on each department card.

**`process()` function (~line 380):** Groups rocks by team, calculates weighted averages, sorts teams by score (highest first). Removes Dropped rocks and rocks whose team is missing/'Unknown' (no Team selected in Notion) from the entire scorecard, including counts and averages; Missed rocks stay in counts but are excluded from scoring via `EXCLUDED_STATUSES`.

**`phase()` function (~line 402):** Maps score ranges to phase labels and colors:
- 0: Not started
- 1-25: Early stages
- 26-50: Building momentum
- 51-75: Strong progress
- 76-99: Near completion
- 100: Complete (cream #e8c9a0, added 2026-10-13 so finished departments stand out)

**`grad()` function (~line 410):** Returns CSS gradient for progress bars based on score range. Uses a warm-to-hot orange ramp from the Mars Men brand palette.

**Quarter picker:** Built into the header, auto-detects current quarter from system date. URL params `?quarter=Q2&year=2026` override the default.

### Design

Mars Men branded. Do not reference Protocol brand (separate personal project).

**Brand colors:** Orange `#FF5600`, Rust `#963821`, Black `#000000`, White `#FFFFFF`

**Extended palette for the scorecard:**
- Cream (score numbers): `#e8c9a0` -- always used for scores so they stay readable
- Amber (near-completion): `#ff8c42`
- Deep burnt: `#5c2210`
- Darkest: `#3d1508`

**Background:** `#191919` (Notion dark mode exact match, not transparent -- Notion embeds render in a light iframe so transparent shows white)

**Typography:** Helvetica Neue for body, JetBrains Mono for numbers/data, uppercase headers with wide letter-spacing, sharp 2px border-radius throughout

**Phase tag colors** follow the progress gradient ramp (darker at low scores, brighter at high). This is intentional even though darker tags are harder to read -- the cream-colored score number provides the readable reference.

### Responsive behavior

- Desktop: full-width, 2-column grid for department cards
- Mobile (<640px): company average full-width, department cards in horizontal scroll strip (75vw per card, scroll-snap)

### Notion embed notes

- Notion overlays its embed controls (comment, expand, menu buttons) on the top-right corner of the iframe. They live in Notion's own page, so no z-index inside the widget can beat them.
- Body padding is `46px 4px 8px`. The 46px top padding nudges the whole widget down so the header row (title + quarter picker) sits below Notion's overlay zone. This is Maria's preferred fix (restored 2026-10-12). History warning: this padding has been accidentally reverted to `8px 4px` twice during refactors, which silently re-breaks the picker. If the picker is covered again, check this value first.
- Cache busting: after deploying changes, the Notion embed URL may need a `?v=N` query param appended (increment N) to force a fresh load. Sometimes deleting and re-adding the embed block is needed.
- The widget is synced to ~32 other pages via Notion synced blocks.

## Worker (worker.js)

Deployed in the Cloudflare dashboard (Workers & Pages > rocks-scorecard-api > Edit Code). Not managed in the GitHub repo.

**What it does:** Receives GET requests with `?quarter=Q2&year=2026`, calls the Notion API with the stored `NOTION_API_KEY` secret, filters the database by Quarter + Year, and returns a lightweight JSON payload with just the fields the widget needs.

**Response shape:**
```json
{
  "rocks": [
    { "rock": "...", "team": "...", "status": "...", "owner": "...", "priority": "..." }
  ],
  "quarter": "Q3",
  "year": 2026,
  "fetched_at": "2026-10-12T..."
}
```

**Caching:** 5-minute `Cache-Control` header. Status changes in Notion take up to 5 minutes to appear.

**CORS:** Open (`Access-Control-Allow-Origin: *`) to allow Notion's embed iframe.

**The Worker does not need changes when statuses are renamed** -- it passes through whatever status names Notion returns. Only `index.html` needs updating for status changes.

## Deployment

### Updating the widget (index.html)
1. Edit `index.html` in the `mariaofmars/rocks-scorecard` GitHub repo
2. Commit to `main`
3. GitHub Pages deploys automatically (~1 minute)
4. If Notion still shows old version: delete the embed block and re-add with `?v=N` incremented

### Updating the Worker (worker.js)
1. Go to dash.cloudflare.com > Workers & Pages > rocks-scorecard-api
2. Click "Edit Code"
3. Make changes
4. Click "Deploy" (takes effect immediately)

### Changing the Notion API key
1. Cloudflare dashboard > Workers & Pages > rocks-scorecard-api > Settings > Variables and Secrets
2. Update the `NOTION_API_KEY` secret

## Common maintenance tasks

**Statuses were renamed or added in Notion:**
Update three things in `index.html`: the `SCORES` object, the `S_COLORS` object, and `EXCLUDED_STATUSES` if the new status should be excluded from scoring. Status names must match exactly.

**New teams were added:**
No code changes needed. New teams are picked up automatically. They'll use the default left-border color (#5c2210). To assign a specific accent color, update the `leftBdr()` function.

**Score weighting needs to change:**
Update the values in the `SCORES` object.

**Phase labels or thresholds need to change:**
Update the `phase()` function.

**Widget shows wrong data or errors:**
1. Check the Worker directly: `https://rocks-scorecard-api.maria-011.workers.dev?quarter=Q3&year=2026`
2. If Worker returns an error, check the Notion integration is still connected to the database and the API key is valid
3. If Worker returns good data but widget looks wrong, the issue is in `index.html` (likely a status name mismatch)

## Accounts

| Service | Account | Subdomain/Org |
|---------|---------|---------------|
| Cloudflare | maria@mengotomars.com | maria-011.workers.dev |
| GitHub | mariaofmars | mariaofmars.github.io |
| Notion | Mars Men workspace | Integration: "Rocks Scorecard" |
