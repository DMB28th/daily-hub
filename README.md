# Daily Hub

A self-contained Cowork artifact that pulls fresh data from Slack, Gmail, Google Calendar, Google Drive, Linear, and your Cowork sessions on every load, synthesizes it with Haiku, and renders four views — Timeline, Last week, This week, Next week — plus a floating chat panel that can create Linear tickets and schedule meetings.

It doesn't try to replace your primary tools. It sits on top of them and gives you one place to start the day.

For architecture detail, see `daily-hub.md` in this repo.

## What it shows

- **Timeline** — horizontal time axis with a lane per source. Click any event for detail. Range selector (this week / last week / 14d / 30d / 90d). One-click Backfill loads the last 8 weeks.
- **This week** (default) — active workstreams + recent decisions pulled from your memory file, ranked "Next: do this" recommendations, a Needs reply queue (Slack + email combined), your open Linear tickets, past + upcoming meetings (with notes/agenda summaries), AI-suggested action items to file, and a "Completed (untracked)" section for work you shipped but never logged.
- **Last week** — retrospective. Lazy-loaded.
- **Next week** — upcoming meeting agendas only. Lazy-loaded.
- **Chat panel** (bottom-right floating button) — natural-language questions over your recent activity, with three confirmable actions: `create_linear_ticket`, `complete_linear_ticket`, `schedule_meeting`.

Every section is collapsible; state persists per-section in localStorage. Past weeks cache forever, the current week refreshes every 6h. Two refresh buttons per tab — `↻ AI` re-runs synthesis against cached data (fast), `Refresh all` clears source caches and re-fetches.

## Prerequisites

- **Cowork.** The Hub depends on `window.cowork.callMcpTool()` and `window.cowork.askClaude()`. It will not run in a plain browser — opening `index.html` directly shows `Cannot read properties of undefined (reading 'callMcpTool')` in the console. Expected.
- **These MCP servers connected to your Cowork account**: Slack, Gmail, Google Calendar, Google Drive, Linear, plus `session_info` (Cowork's built-in session history).
- A modern Chrome- or Safari-based browser (the Cowork artifact view).

## Setup for your account

Clone this repo somewhere on your machine:

```bash
gh repo clone DMB28th/daily-hub
cd daily-hub
```

Then open `index.html` and change four things.

### 1. Your identity (top of the `<script>` block, around line 993)

```js
const ME = "U0AQU1S4GF9";                                  // ← your Slack member ID
const ME_NAME = "Dan Ben-Atar";                            // ← your display name
const ME_EMAIL = "daniel.ben-atar@optioincentives.com";    // ← your work email
```

Find your Slack member ID via Slack → click your avatar → **Profile** → **⋯** → **Copy member ID**. It starts with `U`.

These constants are used to tag messages as "from me" (so Needs reply correctly excludes threads you sent last) and to flag meetings you scheduled.

### 2. Your meeting transcripts folder (around line 3646)

```js
const MEETINGS_BASE = "file:///Users/danielben-atar/Claude/Claude%20Memory/optio/Meeting%20Transcripts/";
```

Point this at wherever you keep meeting transcripts on disk. Any folder of dated `.md` files works — Plaud, Granola, Fathom, Otter exports, or manual paste-ins from Gemini auto-notes. The recommended filename pattern is `YYYY-MM-DD_short-topic_<hash>.md`; the Hub parses the leading date and uses the topic portion as the row label.

Note the URL-encoding: spaces become `%20`.

### 3. Your memory folder path (around line 3559)

```js
const refLink = w.ref ? `<a class="ws-ref" href="file://${escapeHtml("/Users/danielben-atar/Claude/Claude Memory/optio/" + w.ref)}" title="...
```

Change the path prefix to wherever you keep your memory folder. The Hub renders one `↗` link per workstream that resolves to `file://<your-path>/refs/<slug>.md`.

If you don't maintain a memory folder, leave this alone — clicking the link just won't open anything. The Hub still works.

### 4. The embedded memory JSON (lines 941–991)

This is the only chunk of personal data baked into the file: a snapshot of the maintainer's `memory.md`. Replace the contents with your own. The schema is:

```json
{
  "last_updated": "2026-05-12T08:00:00-04:00",
  "right_now": [
    "3–5 top-of-mind items, each one line."
  ],
  "active_workstreams": [
    {
      "name": "Display name",
      "slug": "lowercase-slug",
      "status": "One-sentence current state",
      "next_action": "Concrete next thing to do",
      "ref": "refs/lowercase-slug.md",
      "last_touched": "2026-05-11"
    }
  ],
  "recent_decisions": [
    {
      "date": "2026-05-11",
      "workstream": "lowercase-slug",
      "text": "What was decided.",
      "superseded": "2026-05-12"
    }
  ],
  "exec_stakeholders": [
    {"name": "Full Name", "aliases": ["Full Name", "Nickname"]}
  ],
  "meeting_transcripts": [
    {
      "date": "2026-05-11",
      "topic": "Short topic label",
      "file": "2026-05-11_full-filename.md"
    }
  ]
}
```

The `aliases` array on stakeholders is what powers the Stakeholder Pulse fuzzy-match against Slack/Gmail/Calendar event text — include common nicknames and formal-name variants.

You can hand-edit this JSON whenever your priorities shift. There's no rebuild step — save the file, reload the artifact.

## MCP server bindings — read this if tool calls start failing

The `<script id="cowork-artifact-meta">` block at the very top of `index.html` lists the MCP tool names this artifact uses. Each tool name has the shape `mcp__<server-uuid>__<tool-name>`. Those UUIDs are tied to a specific Cowork installation.

When you install the artifact in your Cowork, the harness *should* rebind the server UUIDs to your installed servers based on the parallel `mcpServerNames` array (Slack / Google Calendar / Google Drive / Gmail / Linear / session_info). If that rebind happens cleanly, you don't need to touch anything.

If something doesn't work — e.g. Slack data is empty, the Linear team picker is blank — open browser dev tools and look for `safeCall` errors. If you see "tool not found", you'll need to manually swap the UUIDs in both the `cowork-artifact-meta` JSON *and* the JS constants further down (`SLACK_SEARCH`, `CAL_LIST`, etc., around line 996) to match your servers' UUIDs. Find your UUIDs in Cowork → settings → MCP servers.

## Installing in Cowork

1. Open Cowork, start a session, and ask: "install this Daily Hub artifact" — paste the contents of `index.html`, or upload the file directly through Cowork's artifact UI.
2. Pin the artifact so it's one click away.
3. Open it. The first load takes longer (Haiku synthesis is ~5–10 seconds with no cache); subsequent loads pull from cache and finish in under a second.

## Keeping it current

- **Memory JSON.** When your priorities change, hand-edit the `<script id="optio-memory">` block. Or, if you have Claude in a Cowork session with access to your memory folder, ask: "update the embedded memory in the Daily Hub from my current memory.md."
- **Transcripts list.** New transcript files in your folder don't appear automatically — the list is embedded. Ask Claude: "re-list my meeting transcripts folder and update the embedded list in the Daily Hub." Or hand-edit the `meeting_transcripts` array.
- **Live data.** Slack, Gmail, Calendar, Drive, Linear, Cowork sessions all refresh from MCP every 6 hours (or on-demand via the Refresh button). No maintenance.

## Caveats worth knowing

- **Browser-local storage only.** Data persists in localStorage and IndexedDB on whatever device opens the Hub. Switching laptops means re-running Backfill on the new one. Cross-device sync was scoped out to keep the architecture self-contained.
- **Haiku-only synthesis.** The `askClaude` Cowork binding uses Haiku. No model parameter. Fast and cheap, but occasionally less reliable on edge cases than Sonnet would be.
- **60s timeout on synthesis.** If Haiku doesn't respond in 60 seconds, you get a red error banner in the AI-driven sections (Week So Far, Needs reply, Action items, Completed) with the actual error + payload diagnostics. Click `↻ AI` to retry against the same cached source data.
- **Prioritization is advisory.** "Next: do this" runs off a cached snapshot (default 6h TTL). Time-sensitive priorities may lag a few hours — manual review of Needs reply and open Linear tickets remains authoritative.
- **Linear + Calendar are the only write surfaces.** The Hub reads Slack and email but never writes back — no draft messages, no replies, no labels. Every write (Linear ticket create/complete, calendar event create) is gated by an explicit confirmation card.

## File layout

```
daily-hub/
  index.html        ← the artifact (one ~4000-line file, self-contained)
  daily-hub.md      ← architecture / design doc
  README.md         ← this file
  thumbnail.png     ← Cowork artifact thumbnail
  versions/         ← Cowork-managed version history (gitignored)
```

## Updating from upstream

Since this is private and shared among Optio colleagues, sync changes by pulling and merging. The maintainer (Dan) commits as features land. If you make your own changes, fork or branch — don't push to `main`.

## Where it came from

Built as a personal dashboard for Dan Ben-Atar (Director of Data, Optio Incentives) on top of the Cowork MCP framework. The companion memory folder pattern is documented in `daily-hub.md` — recommended reading if you want to mirror the full setup.
