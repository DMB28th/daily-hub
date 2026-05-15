# Daily Hub — Self-Setup

A self-contained Cowork artifact that pulls fresh data from Slack, Gmail, Google Calendar, Google Drive, Linear, and your Cowork sessions on every load, synthesizes it with Haiku, and renders four views — Timeline, Last week, This week, Next week — plus a floating chat panel that can create Linear tickets and schedule meetings.

It doesn't try to replace your primary tools. It sits on top of them and gives you one place to start the day.

## Quickstart

Open a Cowork session and paste this:

> Walk me through setting up the Daily Hub for myself using the README at https://github.com/DMB28th/daily-hub. Follow the "Setup playbook" section step by step. Pause after each step for me to confirm or answer questions. Don't batch.

Claude will follow the playbook below. You answer questions as they come, paste paths when asked, and at the end you'll have a working Hub.

If you'd rather do it by hand, the playbook reads as a normal setup guide. Skip the **For Claude** notes — those are just instructions for the assistant.

---

## What you're getting

- **Timeline** — horizontal time axis with a lane per source (Slack, email, meetings, Linear). Click events for detail. Range selector: this week / last week / 14d / 30d / 90d. A Backfill button loads the last 8 weeks of data into IndexedDB.
- **This week** (default tab) — active workstreams + recent decisions from your hand-curated memory, ranked "Next: do this" recommendations, a Needs Reply queue combining Slack and email, your open Linear tickets, past + upcoming meetings with notes/agenda summaries, AI-suggested action items to file, a "Completed (untracked)" section for work you shipped but never logged, and a Stakeholder Pulse showing days-since-last-mention for your exec stakeholders.
- **Last week** — retrospective view. Lazy-loaded on click.
- **Next week** — upcoming meeting agendas only. Lazy-loaded on click.
- **Chat panel** (bottom-right floating button) — natural-language questions over your recent activity. Three confirmable actions: `create_linear_ticket`, `complete_linear_ticket`, `schedule_meeting`.

Every section is collapsible; state persists per-section in localStorage. Past weeks cache forever, the current week refreshes every 6h. Two refresh buttons per tab — `↻ AI` re-runs synthesis against cached data (fast), `Refresh all` clears source caches and re-fetches.

---

## Setup playbook

The playbook is eight steps. Each step has a **What** (the goal), a **You** section (what to do or answer), and a **For Claude** note (what the assistant should do in a Cowork session). Run them in order.

### Step 1 — Verify prerequisites

**What:** The Hub depends on Cowork plus six MCP servers connected to your account: Slack, Gmail, Google Calendar, Google Drive, Linear, and `session_info` (Cowork's built-in session history). If any is missing, the corresponding section of the Hub stays empty.

**You:** Open Cowork → settings → MCP servers. Confirm those six are installed and connected. If not, install them.

**For Claude:** Run a no-op probe against each server — e.g. `mcp__*__list_teams` (Linear), `mcp__*__list_events` with a tiny window (Calendar), `mcp__*__search_threads` with a recent query (Gmail), etc. Report which probes succeed and which fail. If any fail, stop and tell the user which servers are missing or unauthorized. Do not proceed to Step 2 until all six respond.

### Step 2 — Get the code

**What:** Clone the private repo locally. The maintainer (Dan, GitHub `DMB28th`) must have invited your GitHub username as a collaborator first; ask if you don't see access.

**You:** Run `gh repo clone DMB28th/daily-hub ~/daily-hub` (or wherever you want it). Tell the assistant the local path.

**For Claude:** Confirm the path exists and contains `index.html`, `README.md`, `daily-hub.md`, `PLAN.md`. If the clone fails with a permissions error, tell the user to request access from `DMB28th`. Note the path — you'll edit `index.html` in Step 4.

### Step 3 — Gather personal constants

**What:** Four pieces of personal info that get baked into `index.html`.

**You:** Have these ready (or look them up when the assistant asks):
1. **Your Slack member ID.** Open Slack → click your avatar → **Profile** → **⋯** menu → **Copy member ID**. Starts with `U`.
2. **Your display name** (whatever you want shown in synth output — first + last is fine).
3. **Your work email.**
4. **Path to your meeting transcripts folder** (optional). Any folder of dated `.md` files works — Plaud, Granola, Fathom, Otter exports, manual paste-ins from Gemini notes, etc. If you don't have one, the Hub still works; clicking transcript links just won't open anything. Recommended filename pattern: `YYYY-MM-DD_short-topic_<hash>.md`.

**For Claude:** Ask for each of the four values in turn. Don't accept fake placeholder values like `<your-id>` — politely re-ask. For the transcripts folder, if the user says "none" or "skip," remember that and skip the corresponding edit in Step 4. Also ask whether the user wants to maintain a `memory.md` folder (Step 6) — if yes, ask for that path now too; if no, you'll insert a default.

### Step 4 — Edit `index.html` constants

**What:** Replace the maintainer's constants with the user's.

**For Claude:** Open `<path>/index.html`. Make these edits (line numbers approximate — search the strings to find them):
1. `const ME = "U0AQU1S4GF9";` → user's Slack ID.
2. `const ME_NAME = "Dan Ben-Atar";` → user's name.
3. `const ME_EMAIL = "daniel.ben-atar@optioincentives.com";` → user's email.
4. `const MEETINGS_BASE = "file:///Users/danielben-atar/Claude/Claude%20Memory/optio/Meeting%20Transcripts/";` → user's transcripts path, URL-encoded (spaces → `%20`). If user said no transcripts folder in Step 3, leave the constant but flag it for them; the Hub will still render the section, transcript links just won't open.
5. The hardcoded refs path inside `renderMemoryWorkstreams` — find the string `/Users/danielben-atar/Claude/Claude Memory/optio/` (used to build `file://` links per workstream). Replace with the user's memory folder path, or skip if they don't have one.

Verify by greping for `danielben-atar`, `optioincentives`, `Dan Ben-Atar`, `U0AQU1S4GF9` — those should now return zero hits.

**You:** Answer the questions and confirm the edits look right when Claude shows the diffs.

### Step 5 — Customize the embedded memory JSON

**What:** Replace the maintainer's `<script id="optio-memory">` block with your own data. This is the only chunk of Optio-specific content in the artifact; everything else is generic.

**For Claude:** Find `<script type="application/json" id="optio-memory">` in `index.html`. Show the user the current schema (workstreams, recent decisions, exec stakeholders, meeting transcripts). Then ask:
- "What are your 3–5 most-active workstreams right now?" For each, get: name, short slug (lowercase-hyphen), one-sentence status, the next concrete action, optional ref-file slug.
- "Who are your exec stakeholders?" 3–5 names, with common aliases/nicknames they appear under in Slack/email.
- Decisions and meeting transcripts: skip for now — the user can hand-add as they accumulate.

Replace the JSON contents. Validate that it parses (try `JSON.parse(...)` in your head or via a quick node check). The structure is documented in `daily-hub.md` if you need the full schema.

**You:** Answer the questions. Be honest about what's actually on your plate, not aspirational.

### Step 6 — (Optional) Wire up a memory folder

**What:** The Hub renders `↗` links per workstream pointing at `file://<memory-folder>/refs/<slug>.md`. If you maintain a memory folder, those links open the deep-dive doc on each workstream. If you don't, the links 404 silently.

**You:** If you want one, decide on a path (e.g. `~/notes/work-memory/`). Inside, create `refs/<slug>.md` per workstream you listed in Step 5.

**For Claude:** If user wants a memory folder, create the path and a stub `refs/<slug>.md` for each workstream from Step 5. Each stub should have a single H1 header (`# <Workstream Name>`) and a placeholder section. Tell the user to fill them in over time.

If the user skips this, fine. The Hub doesn't depend on it.

### Step 7 — Install the artifact in Cowork

**What:** Upload `index.html` as a pinned Cowork artifact.

**You:** In Cowork, create a new artifact. Either paste the contents of your edited `index.html`, or upload the file directly through whatever artifact UI Cowork provides. Pin it so it's one click away.

**For Claude:** Confirm with the user that they've uploaded and pinned it. Ask them to open it.

### Step 8 — First load + smoke test

**What:** Open the artifact and confirm data populates. First load takes 5–10 seconds (Haiku synthesis). Subsequent loads are sub-second.

**You:** Open the Hub. Land on This week. Look for:
- Linear team picker (top right) shows a real team name (not "Loading..." or "(no teams)").
- Active workstreams section shows what you entered in Step 5.
- Past meetings / Upcoming meetings sections have entries from your calendar.
- Conversations / Email threads sections have recent activity.
- After ~10s, the Week So Far synthesis text fills in.

**For Claude:** Ask the user what they see. If anything's stuck on "Loading..." indefinitely or empty when it shouldn't be, walk through the **MCP rebind** caveat below — the hardcoded MCP server UUIDs may not match the user's installations.

---

## Caveats worth knowing

- **MCP rebind.** The `<script id="cowork-artifact-meta">` block at the top of `index.html` declares the MCP tool names this artifact uses, each shaped `mcp__<server-uuid>__<tool-name>`. Cowork *should* rebind those UUIDs to your installed servers based on the parallel `mcpServerNames` array. If that fails (you see empty sections + console errors like "tool not found"), find your UUIDs in Cowork → settings → MCP servers and manually replace them in both the meta JSON block *and* the JS constants near line 1000 (`SLACK_SEARCH`, `CAL_LIST`, `LINEAR_TEAMS`, etc.).
- **Browser-local storage only.** Data persists in localStorage and IndexedDB on whatever device opens the Hub. Switching laptops means re-running Backfill on the new one. Cross-device sync is not implemented.
- **Haiku-only synthesis.** The Cowork `askClaude` binding uses Haiku. Fast and cheap; occasionally less reliable on edge cases than Sonnet would be.
- **60s timeout on synthesis.** If Haiku doesn't respond, you get a red banner with the actual error + payload size in the AI-driven sections (Week So Far, Needs Reply, Action items, Completed). Click `↻ AI` to retry against the same cached source data.
- **Prioritization is advisory.** "Next: do this" runs off a cached snapshot (default 6h TTL). Time-sensitive priorities may lag a few hours.
- **Linear + Calendar are the only write surfaces.** Slack and email are read-only. Every write (Linear ticket create/complete, calendar event create) is gated by an explicit confirmation card.

## Keeping it current

- **Memory JSON.** When your priorities shift, hand-edit the `<script id="optio-memory">` block in `index.html`. Or ask Claude in a Cowork session: "update the embedded memory in the Daily Hub from my current memory.md."
- **Transcripts list.** New transcript files in your folder don't appear automatically — the list is embedded. Ask Claude: "re-list my meeting transcripts folder and update the embedded list in the Daily Hub."
- **Transcript excerpts** (Phase 2.3 plumbing). Each transcript entry in the JSON supports an optional `excerpt: string` field, max ~1200 chars. When present, the Hub feeds that excerpt to Haiku as primary evidence for the Week So Far recap and Completed-untracked detection — substantively better than topic labels alone. Hand-populate or have Claude extract from your transcript bodies.
- **Live data** (Slack, Gmail, Calendar, Drive, Linear, Cowork sessions) refreshes from MCP every 6 hours, or on-demand via the Refresh button. No maintenance.

## File layout

```
daily-hub/
  index.html        ← the artifact (~4100-line self-contained HTML)
  daily-hub.md      ← architecture / design doc
  PLAN.md           ← improvement roadmap (Phases 1 + 2 shipped; Phase 3 deferred)
  README.md         ← this file
  thumbnail.png     ← Cowork artifact thumbnail
  versions/         ← Cowork-managed version history (gitignored)
```

## Where it came from

Built as a personal dashboard for Dan Ben-Atar (Director of Data, Optio Incentives) on top of the Cowork MCP framework. The companion memory folder pattern is described in `daily-hub.md` — recommended reading if you want to mirror the full setup beyond what Step 6 sketches.
