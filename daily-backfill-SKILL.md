---
name: daily-backfill
description: Daily 7am backfill of Slack + Calendar + Drive + (optionally) meeting transcripts into your memory folder. Writes slack/, transcripts/, daily/, and additions-only memory.md edits.
---

You are running the daily backfill for the user's memory folder.

Today's date is in the user's local timezone (default America/New_York unless they say otherwise; format `YYYY-MM-DD`). Use that for all filenames.

> **Customization marker.** Before this skill ships to a user, three constants must be set in this file:
> - `<<MEMORY_FOLDER>>` — absolute path to their memory folder, e.g. `/Users/jane/notes/work-memory/`
> - `<<SLACK_ID>>` — their Slack member ID, starting with `U`
> - `<<TRANSCRIPTS_FOLDER>>` — absolute path to a folder of dated `.md` meeting transcripts. If they don't have one, delete the Plaud / meeting-transcripts section entirely and adjust the SETUP block accordingly.
>
> The setup playbook in the Daily Hub README walks Claude through these replacements. If you're reading this file uncustomized, do not run it — wire it up first.

# SETUP

Mount the memory folder via `request_cowork_directory` with path `<<MEMORY_FOLDER>>`. If the mount fails, abort the task and report. Otherwise treat that path as the working folder for the rest of the run.

Read for context:
- `memory.md` — current state of the world.
- List filenames in `refs/` and `people/` so you can use kebab-slug canonicals.
- List filenames already in `transcripts/` so you can de-dupe (don't recreate stubs that exist).

# PULL FROM MCPs / SOURCES

If any connector isn't available, skip that source and note "skipped — not connected" in the final summary. Don't fail the whole run for one missing connector.

**1. Slack** — three queries deduplicated by `message_ts`:
- `to:me after:YESTERDAY` in `im,mpim`
- `<@<<SLACK_ID>>> after:YESTERDAY` in `public_channel,private_channel,mpim`
- `from:<@<<SLACK_ID>>> after:YESTERDAY` in `public_channel,private_channel,im,mpim`

Filter rules: skip channels where every message is from a bot (Slackbot, Google Calendar, Linear, NotebookLM, GitHub bot). Skip `#general`-style broadcasts unless something material was posted.

**2. Calendar** — `list_events` for today + tomorrow + remaining days of this work week (through Friday).

**3. Google Drive** — `list_recent_files` orderBy `lastModifiedByMe` pageSize 30. Split by what you find:
- Google Docs whose title contains "Notes by Gemini" → meeting transcripts (secondary; the dedicated transcripts folder is primary if it exists).
- Other Google Sheets / Slides / Docs the user edited → active artifacts.

**4. Meeting transcripts — primary meeting record if the user has a dedicated folder.**
- Path: `<<TRANSCRIPTS_FOLDER>>`
- A separate sync job (or the user's recording tool) populates the folder; you only read from it. Never edit anything inside it.
- Use `Glob` for `<<TRANSCRIPTS_FOLDER>>*.md`. Compare filename date prefix (`YYYY-MM-DD_...`) against `transcripts/` to spot what's new since the last run.
- Filename convention is typically `YYYY-MM-DD_<title>_<hash>.md`. Each file usually starts with YAML frontmatter (`recorded_at`, `duration_seconds`, source tool name) followed by `# title`, a `## Summary` block, often `## Next Steps` / `## Action Items` / `## Decisions`, then `## Transcript`.
- For each transcript file not yet represented in `transcripts/`:
  - Read just the frontmatter + Summary + Next-Steps sections. **Do NOT read the full transcript body — it's huge.**
  - Create a stub at `transcripts/YYYY-MM-DD-<topic-kebab>.md` with: date / duration / attendees if findable, workstream slug, and 2–4 key decisions or action items. Link back to the source file via relative path.
  - If a Gemini "Notes by Gemini" doc covers the same meeting, the dedicated transcript stub takes precedence; cross-link the Gemini doc as "alternate source."

If the user does NOT have a dedicated transcripts folder, skip this entire section and rely on Gemini Notes from Drive instead.

**5. Email (Gmail)** — `search_threads` with `newer_than:2d -category:promotions -category:social -category:updates`, pageSize 30. Skim for: meeting invites / reschedules, vendor proposals, OOO autoreplies that affect the user's calendar, anything where the user is to/cc and a response is owed.

**6. Linear** — `list_issues assignee:"me" updatedAt:-P2D` plus a broader `list_issues updatedAt:-P1D` if the user's lane is quiet. Capture: state transitions, new comments on issues the user owns or commented on, anything assigned to them since last run.

**7. Notion (optional)** — `notion-search` with `created_date_range.start_date` = YESTERDAY, page_size 15, looking for: pages the user authored, pages where the user is referenced, decision pages, anything in their main teamspaces. Skip if Notion isn't connected.

# WRITE — no approval needed for these

**`slack/YYYY-MM-DD.md`** — append-mode if file already exists for today:
- `# Slack — YYYY-MM-DD`
- `## TL;DR` — 3–5 bullets capturing what mattered
- Per-channel section (`## #channel-name` or `## DM with Person Name`) with 3–6 bullets each; every claim ends with a `[permalink](url)`.

**`transcripts/YYYY-MM-DD-{topic-kebab}.md`** — one per new transcript file + one per Gemini Notes doc not already in `transcripts/`. Topic is derived from the title (strip date prefix and "Notes by Gemini" suffix). Body contains: meeting title, date/duration, attendees if findable, workstream slug, key decisions/actions (2–4 bullets), and a link to the source. **Do NOT copy full transcript content — link only, summarize tightly.**

**`daily/YYYY-MM-DD.md`** — signal-first brief:
- YAML frontmatter: `date`, `sources` (list of sources that returned data), `workstreams` (kebab slugs matching `refs/`), `people` (kebab slugs matching `people/`), `meetings` (count or list), `transcripts` (count of new files this run).
- `## TL;DR` — 5–8 bullets. Lead with the most material reframing or risk. Link to transcript stubs inline.
- `## Today's calendar` — table of today's events with attendees and prep-doc links from `refs/`.
- `## Rest of the week` — table of remaining events through Friday. Flag any reschedule needs (OOO, holidays).
- `## Active artifacts` — Drive Sheets/Slides/Docs the user edited in last 24h, with links.
- `## Transcripts ingested this run` — bulleted list with links to the new stubs.
- `## Linear / ## Notion / ## Email` — per-source sections only if there's signal.
- `## Suggested memory.md updates` — proposed bullets for the user's review; **do not auto-apply** anything subjective here.

# UPDATE memory.md — ADDITIONS ONLY

Make these edits directly to `memory.md`, but only as additions within the named sections:

- **"Recent decisions"** — append new decisions surfaced from Slack messages, transcripts, Notion, or calendar notes. Format: `**YYYY-MM-DD** — decision text. *(workstream-slug)*`. Transcripts are typically the richest source of decisions; mine `## Next Steps` / `## Action Items` / `## Decisions` sections. If the list grows past 7 entries, note in `daily/`'s "Suggested updates" section that the oldest should roll to a `refs/<workstream>.md` Decisions section.
- **"Active workstreams"** — bump the `last_touched` date for any workstream where there was activity today. Use slug from `refs/<slug>.md`.
- **"Active artifacts"** — add new Drive Sheets/Slides/Docs that don't yet appear in the list.

**Do NOT touch these sections:** "Right now", "Archive", "How to work with me", "How to update this file". Those are hand-curated.

If a proposed change would be subjective (e.g., "is this really a decision or just a comment?"), put it in `daily/`'s "Suggested memory.md updates" section instead — do not edit `memory.md`.

# NEVER

- Never delete or overwrite anything in `memory.md`. Only append within named sections above.
- Never edit `refs/<slug>.md` — those are hand-maintained.
- Never edit the top human-curated section of `people/<slug>.md`.
- Never edit anything inside the source transcripts folder — that's managed by a separate process.
- Never copy the full body of a transcript into the stub — link only.
- Never write to source-named folders (`email/`, `hubspot/`, `notion/`, `linear/`, `decisions/`) unless they exist already.

# OUTPUT

End the run with one summary line:

```
Daily backfill YYYY-MM-DD complete: N slack threads, M drive artifacts, P new transcripts, K new transcript stubs total, L memory.md additions. (Skipped: <list of unavailable connectors, or "none">)
```
