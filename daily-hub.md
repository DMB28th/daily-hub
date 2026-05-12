# Daily Hub

A live, cross-source personal dashboard built as a Cowork artifact. Pulls fresh data from Slack, Gmail, Calendar, Drive, Linear, and Cowork sessions on every load, synthesizes it with Haiku, and renders four views over the week.

The Hub is the **live view**. A companion markdown folder serves as the **curated memory**. The bridge between them: a JSON snapshot of `memory.md` is embedded directly inside the Hub's HTML, plus an auto-listed feed of meeting-transcript filenames.

## What it does

On every reload, the Hub fetches data from connected MCP tools and renders four tabs:

- **Timeline** — Gong-style horizontal time axis with a lane per source. Click any event for detail. Drill into a single day for hourly view.
- **Last week** — retrospective. Conversations, meetings, completed action items.
- **This week** — operational. Default tab. Active workstreams, recent decisions, recent meeting transcripts, ranked "Next: do this" recommendations, a Needs-reply queue (Slack + email combined), open Linear tickets, past and upcoming meetings, AI-suggested action items, and a "Completed (untracked)" section for work shipped but never logged.
- **Next week** — forward-only. Upcoming meeting agendas.

A floating **chat panel** (bottom-right) accepts natural-language questions and can take three actions on confirmation: create a Linear ticket, mark a ticket Done, or schedule a calendar meeting.

Last/Next week tabs are **lazy-loaded** — they show a dormant banner until you click into them. Once loaded, past weeks cache forever; the current week refreshes every 6 hours. Two refresh buttons per tab: **↻ AI** (re-runs Haiku synth against cached source data, fast) and **Refresh all** (clears source caches and re-fetches everything).

Every section is **collapsible** — click any header chevron to toggle. State persists per-section in localStorage.

## Setting up the memory folder

The Hub looks for an embedded JSON snapshot inside its HTML, sourced from a folder of markdown files you maintain by hand. This section is the canonical setup recipe — adapt the paths to wherever you want your memory to live.

### 1. Create the folder

Pick a location your shell can reach (`~/Documents/`, `~/Claude/`, anywhere). Inside, create:

```
~/your-memory/
  README.md          ← describes the folder for anyone (or future-you) opening it
  memory.md          ← the front-door index, hand-curated
  daily/             ← one file per day, dated YYYY-MM-DD.md
  refs/              ← one file per long-lived workstream, hand-maintained
  slack/             ← daily Slack summaries, auto-populated or hand-pasted
  transcripts/       ← meeting transcript files (see next section)
  people/            ← one file per colleague — comms style, project history
```

The folder works on its own — no app dependency. The Daily Hub reads from it but doesn't require it.

### 2. Structure `memory.md`

This is the front door. Aim for something you can scan in 60 seconds. The Daily Hub parses these sections:

```
# memory.md

## Right now
- 3–5 top-of-mind items, each one line.

## Active workstreams
- **Workstream name** *(slug)* — Status sentence. → `refs/<slug>.md`
  - **Next:** Concrete next action.

## Recent decisions
- **YYYY-MM-DD** — Decision text. *(workstream-slug)*

## People
### Exec stakeholders
- **Name** — Role, context.

### Immediate team
- **Name** — Role, working relationship.

## Active artifacts
- **Name** — link or path.
```

Keep section headers stable — the embedded JSON snapshot mirrors the structure. When you change `memory.md`, ask Claude in a Cowork session to "update the embedded memory in the Daily Hub" and it'll regenerate the snapshot inside the HTML.

### 3. Create the daily-backfill scheduled task

A scheduled task runs at 7am EDT daily, pulls the last 24h of Slack/Calendar/Drive activity, and writes to `slack/`, `transcripts/`, and `daily/`. It makes conservative additions to `memory.md` (Recent decisions, Active workstreams `last_touched`, Active artifacts) but **never touches the curated "Right now" or "How to work with me" sections**.

The task definition lives at `~/Documents/Claude/Scheduled/daily-backfill/SKILL.md`. The Hub still works without it — the task just makes it cheaper to keep current. To create the task, ask Claude: "create a scheduled task to backfill my memory folder from Slack, Calendar, and Drive every morning at 7am EDT."

## Setting up the meeting transcripts folder

The Hub's "Recent meeting transcripts" section reads transcript filenames from a folder on disk. The original setup uses Plaud (ambient meeting recorder), but the integration is just a folder of markdown files — any tool that drops `.md` files in a predictable folder works (Granola, Fathom, manual exports from Gemini Notes, etc.).

### 1. Pick a folder + naming convention

The Hub reads from `~/optio/Meeting Transcripts/` by default. To use a different folder, edit the `MEETINGS_BASE` constant in the Daily Hub HTML (or ask Claude to point it elsewhere).

Recommended filename pattern: `YYYY-MM-DD_short-topic-summary_<hash>.md`. The Hub parses the leading date to sort and age the entry, and uses the topic portion as the display label.

### 2. Get transcripts into the folder

Three options, depending on your recording tool:

- **Plaud** — exports via their API. The `Meeting Transcripts/.plaud_sync/` directory has an `install.sh`, `sync.sh`, and `schedule.sh` for an hourly pull. Token sits at `Meeting Transcripts/.plaud_sync/token.txt` (gitignored).
- **Granola / Fathom / Otter** — most have export-on-finish flags or weekly bulk-export. Configure that to write into the chosen folder.
- **Manual** — paste Gemini auto-notes or call transcripts into a markdown file with the date-prefix naming convention. The Hub picks them up on next reload.

### 3. Embed the file list in the Hub

The Hub keeps a static `meeting_transcripts` array inside its embedded JSON snapshot (date + topic + filename). When new transcripts land in the folder, ask Claude in a Cowork session: "refresh the meeting transcript list in the Daily Hub" and it'll re-list the folder and update the embedded JSON.

The transcript rows link via `file:///` URLs back to the markdown source, so clicking opens the raw transcript in your default markdown viewer.

## Daily flow

1. Open the pinned Daily Hub from Cowork.
2. Land on **This week** by default. The top of the tab presents up to three ranked "Next: do this" recommendations. Dismiss what's stale, act on the rest.
3. **Active workstreams** + **Recent decisions** + **Recent meeting transcripts** show the curated memory pulled from `memory.md` and the transcripts folder. All collapsible — click the chevron to toggle.
4. **Needs reply** consolidates open asks across Slack and email — hover any card for an Ignore button.
5. **My open Linear tickets** — hover any row for ✓ Complete to close in place.
6. **Action items — open** — Create button posts a new Linear ticket to the selected team.
7. **Completed (untracked)** — one click creates the ticket and marks it Done in a single Linear API call, for work that was shipped but never tracked.
8. Use the chat panel for ad-hoc questions or to take an action.
9. Switch to **Timeline** for the chronological view.

## The four tabs

### Timeline

A horizontal time axis with four lanes: Slack (blue dots), Email (red dots), Meetings (purple time-blocks sized by duration; faded if declined), Linear (green dots — `+` for created, `✓` for completed). Today is marked in red. Dense days auto-cluster into `+N` badges.

Range options: Last week / This week / Last 14 days / Last 30 days / Last 90 days. The 30 and 90 day ranges query IndexedDB, populated by browsing and by a one-click **Backfill** button that fetches the last 8 weeks of data.

### Last / This / Next week

Each tab opens with a Haiku-generated week summary structured as *Worked on / Talked about / Planned*. Below the summary, sections appear in a fixed order, all collapsible with state persisted in localStorage.

For This week:

1. Week summary
2. Memory sections — Active workstreams, Recent decisions, Recent meeting transcripts
3. Next: do this (ranked recommendations)
4. Needs reply (Slack + email combined)
5. Action items — open
6. My open Linear tickets
7. Completed (untracked)
8. Conversations
9. Email threads
10. Past meetings (with Gemini-notes summaries where available)
11. Upcoming meetings (with agendas)

Next week strips to Upcoming meetings only.

### Chat panel

Each message builds fresh context from aggregated KPI metrics for the current view, the last 14 days of events from IndexedDB, the top 30 open Linear tickets by priority, the most recent week recap, and a compact memory summary (workstreams + decisions + meeting topics inlined into the prompt). Three confirmable actions: `create_linear_ticket`, `complete_linear_ticket`, `schedule_meeting`. Each action renders as an amber confirmation card before execution.

## Data sources

| Source | What gets pulled |
|---|---|
| Slack MCP | DMs, @-mentions, sent messages |
| Gmail MCP | Inbox threads with active engagement, noise-filtered |
| Calendar MCP | Past, today, and upcoming meetings |
| Drive MCP | Recent files; Gemini meeting notes paired with calendar events |
| Linear MCP | Open tickets + recent created/completed |
| session_info MCP | Recent Cowork sessions as completion evidence |
| `memory.md` | Embedded as JSON snapshot in the HTML |
| Meeting transcripts folder | Filenames + parsed dates, embedded as JSON list |

## How synthesis works

The AI synthesis call sends a slim payload (capped at 20 Slack threads, 15 emails, 12 past meetings, 10 cowork sessions) plus inline memory context to Haiku. Output includes per-thread summaries, per-meeting summaries, ranked next actions, completed-untracked candidates, and a *Worked on / Talked about / Planned* week recap.

Hard 60s timeout. If Haiku doesn't respond in time, a red error banner shows the actual error message + payload size, and the **↻ AI** button retries against existing cached source data without re-fetching from MCP.

Synthesis output is cached by a content hash of the payload — if the same Slack/email/meeting data is there, no fresh AI call is made on reload.

## Caching architecture

Two layers, both browser-local and per-device:

**localStorage** (synchronous, ~5–10MB):
- Per-source data caches — 6h TTL for current/next week, persistent for past weeks
- Synthesis output keyed by content hash
- UI state (active tab, collapsed sections, ignored items, dismissed actions, team selection)

**IndexedDB** (asynchronous, much larger):
- Flat event log spanning all sources — powers long-range Timeline views
- Weekly metrics snapshots — powers KPI aggregation
- Linear ticket snapshots — powers diff computation

A per-tab Refresh button clears the localStorage cache for that week and re-fetches. A footer Clear cache option wipes localStorage; Shift-click also clears IndexedDB.

## Design choices worth calling out

- **Browser-local storage only.** Data persists in localStorage and IndexedDB on the device where the Hub is opened. Switching devices means re-running Backfill. Cross-device sync was scoped out to keep the architecture self-contained.
- **Linear and Calendar are the only write surfaces.** Slack messages and email drafts are intentionally read-only. Every write is gated by an explicit confirmation card.
- **The Hub doesn't replace primary tools.** It views and acts on data from Google Calendar, Slack, Linear, etc., but doesn't try to be the primary management surface for any of them.
- **Prioritization is advisory.** The "Next: do this" card is generated from a cached snapshot (6-hour TTL by default), so highly time-sensitive priorities may lag a few hours. Manual review of Needs reply and open Linear tickets remains authoritative.
- **Memory is hand-curated.** The daily-backfill task only makes conservative additions. Curated sections (`Right now`, `How to work with me`) are never touched by automation.

## Updating the Hub

The HTML lives at `~/Documents/Claude/Artifacts/daily-hub/index.html` — fully self-contained. It can be edited directly or via Claude in a Cowork session.

A `CACHE_VERSION` constant in the JavaScript layer is bumped whenever synthesis schemas change, automatically invalidating stale synthesis caches. **Be careful with this** — bumping it wipes source caches too, forcing a full MCP re-fetch on next load.

To version-control the Hub:

```bash
cd ~/Documents/Claude/Artifacts/daily-hub
git init
echo ".DS_Store" > .gitignore
git add index.html .gitignore
git commit -m "Initial: Daily Hub"
gh repo create daily-hub --private --source=. --remote=origin --push
```

A private repository is recommended — the embedded `memory.md` JSON contains names, ticket IDs, and links to internal threads.
