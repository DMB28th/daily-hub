# Daily Hub — Improvement Plan

Eight functional improvements, grouped in three phases by risk and dependency. Each phase ships as its own commit on `main` so any regression has a clean rollback point.

## Conventions for all phases

- The artifact is a single self-contained HTML file at `index.html`. No build step.
- Do **not** bump `CACHE_VERSION` for prompt-schema changes — the synth cache is content-hash-keyed (line ~1742), so payload-shape changes already produce new cache keys naturally. The `CACHE_VERSION` constant only invalidates source-data caches.
- Maintain the existing CSS conventions: classes like `.ms-<section>-<element>`, IDs like `#ms-<section>`.
- Preserve the async signature on `renderOptioMemorySections` — `loadWeek` does `.catch()` on its return.
- Memory render functions use `classList.add/remove("ms-empty")`, never `className =` (the old `className = "ms-list"` pattern silently wiped the `section-body` collapse class).
- After edits, commit + push with a descriptive message.

---

## Phase 1 — Low risk, purely additive

Three independent additions. None touch existing logic; each can be reverted in isolation.

### 1.1 Wire Stakeholder Pulse

**Why:** `renderMemoryStakeholderPulse` is dead code at line 3605. It computes "days since last mention" per exec from IndexedDB events and is never rendered. Useful for relationship hygiene — Michael Magnussen at 14d cold is the kind of thing you want surfaced.

**Where:**
- HTML: add a `<div class="memory-section" id="ms-pulse">` block after the `id="ms-meetings"` block (currently around line 828).
- JS: call `renderMemoryStakeholderPulse(memory)` inside `renderOptioMemorySections` (line 3659).
- CSS: the function expects classes `.ms-pulse-row`, `.ms-pulse-name`, `.ms-pulse-days` with stale/aging/fresh variants. Verify they exist; if missing, add them matching the existing `.ms-meeting-row` pattern.

**Approach:**
1. Grep for `.ms-pulse-` to confirm whether CSS already exists.
2. Add HTML section with label "Stakeholder pulse" and source hint `~/optio/memory.md exec_stakeholders`.
3. Add the function call after `renderMemoryMeetings(memory)`.
4. The pulse is await-ed inside `renderOptioMemorySections` so the IDB query completes before render.

**Risk:** Section will be empty on first load if IndexedDB has no events. That's the existing behavior — render shows "no mention in 90d" for each exec. Fine.

**Acceptance:** Reload Hub. "Stakeholder pulse" section appears under Recent meeting transcripts. Each exec from `exec_stakeholders` shows with a days-ago badge or "no mention in 90d".

---

### 1.2 One-line "week vibe" summary

**Why:** The Week So Far recap is 3 paragraphs. There's no super-fast scan element. A single italic sentence at the very top — "Heavy on BI vendor decisions; ARR work continuing; Linear quiet" — gives the at-a-glance read.

**Where:**
- Synth prompt at line 1746: add `week_vibe: string` to the output JSON spec, instructing the model to produce one sentence (≤25 words) derived from its own `section_summaries`.
- JS rendering: new function `renderWeekVibe(weekKey, vibeText)` that renders into a new element placed above `${weekKey}-summary`.
- HTML: add `<div class="week-vibe" id="${weekKey}-vibe"></div>` in each week tab's structure.
- CSS: italic, slightly larger than body text (~16px), muted color, ~12px vertical margin.

**Approach:**
1. Modify the synth prompt's STRICT JSON spec near line 1825 to include `"week_vibe":"..."`.
2. Add prompt instruction: "Then `week_vibe`: ONE sentence (≤25 words) capturing the dominant theme of the week. Derived from your `section_summaries`. Write in the same voice as `week_recap`."
3. Call `renderWeekVibe(weekKey, synth.week_vibe)` in `loadWeek` right before `renderWeekRecap`.
4. Create container `<div id="this-vibe">`, `<div id="last-vibe">`, `<div id="next-vibe">` inside the existing tab HTML — find them by reading the existing tab structure.
5. For `synthesizeAgendas` (next-week path at line 1853), add `week_vibe` to its output too.

**Risk:** Adds one more bit of model output. Cache key naturally regenerates. No structural risk.

**Acceptance:** Reload Hub on This week. A single italic sentence appears at top, distinct from the 3-paragraph Week So Far. Same on Last week (after click-to-load).

---

### 1.3 Aging on Needs Reply

**Why:** A thread that's been waiting 4 days for a reply looks identical to one waiting 4 hours. Visual aging surfaces urgency without manual triage.

**Where:**
- `renderUrgent` function — find it by grep (search for `function renderUrgent`).
- Each thread has `latestTs` (Unix epoch seconds) and each email has a normalized timestamp; both already used in `relTime()`.

**Approach:**
1. For each card rendered in Needs Reply, compute `daysWaiting = (now - latestTs) / 86400`.
2. Render a small badge like `<span class="urgent-age">${days}d</span>` aligned right on the card.
3. CSS classes by age bucket:
   - 0–1 day: `urgent-age` (default, neutral)
   - 2–4 days: `urgent-age aging` (amber)
   - 5+ days: `urgent-age stale` (red); also add `urgent-old` class to the parent card for a subtle red left-border.
4. **Do not** modify the synth prompt in Phase 1. Auto-promotion of >5d items to `next_actions` is Phase 2.

**Risk:** Pure UI decoration; if `latestTs` is missing for any item, fall back to no badge rather than NaN.

**Acceptance:** Open This week. Each Needs Reply card shows a small "Xd" or "Xh" badge. Anything >2d has amber color; anything >5d has red border on left.

---

## Phase 2 — Medium risk, moderate scope

Three changes with light state migration or prompt-engineering.

### 2.1 Snooze instead of binary ignore

**Why:** Today's `IGNORED_KEY` is a Set of IDs — once you click Ignore, the item never comes back. Most "ignore" actions are really "not now." Snooze respects that.

**Where:**
- `IGNORED_KEY = "wd:ignored"` constant; `lsGetSet` / `lsSetSet` helpers.
- Every Ignore button on cards: Needs Reply, Action items open, Linear tickets (Complete is different, leave alone).

**Approach:**
1. Change storage shape: `wd:ignored` becomes `{ [id]: { until_iso: string } }` instead of a flat set. Add a one-time migration: if the value is an array, convert each entry to `{ until_iso: "9999-12-31" }` so existing ignores still work.
2. Replace the single Ignore button on each card with a small dropdown: **1d / 3d / 1w / Forever**.
3. On click, write `{ until_iso: <computed-iso> }` for that id.
4. In each render function that filters out ignored items, check `Date.now() < new Date(entry.until_iso).getTime()` — show items past their snooze date again.
5. Add helper `lsGetIgnoreMap()` and `lsSetIgnoreEntry(id, untilIso)`.

**Risk:** State migration. Test with a populated ignore list (manually seed one in localStorage before testing).

**Acceptance:** Click Ignore on a thread, pick "1d". Reload — thread is gone. Wait 24h (or manually edit localStorage to backdate `until_iso`) and reload — thread is back.

---

### 2.2 Coarser synth cache key

**Why:** Current cache key is `djb2(JSON.stringify(payload))` at line 1742. Every 1-char change in any of 20 truncated Slack snippets busts the cache, forcing a fresh Haiku call. Coarser keying means more cache hits, fewer redundant calls.

**Where:** Line 1742, the `synthCacheKey` construction.

**Approach:**
1. Build a structural signature instead of raw-bytes hash:
   ```js
   const sig = JSON.stringify({
     week: weekKey,
     threadCount: payload.threads.length,
     emailCount: payload.emails.length,
     pastMeetingCount: payload.past_meetings.length,
     upcomingMeetingCount: payload.upcoming_meetings.length,
     coworkCount: payload.cowork_sessions.length,
     latestThreadTs: Math.max(0, ...payload.threads.map(t => (t.recent_messages?.[t.recent_messages.length - 1] ? Date.now() : 0))),
     // Include a hash of just the thread channelIds + latest_from_is_me flags — captures "who replied last where"
     threadSig: payload.threads.map(t => `${t.id}:${t.latest_from_is_me ? 1 : 0}`).join("|"),
     emailSig: payload.emails.map(e => `${e.id}:${e.latest_from_is_me ? 1 : 0}`).join("|"),
     workstreamSig: (memory.active_workstreams || []).map(w => `${w.slug}:${w.last_touched}`).join("|")
   });
   const synthCacheKey = `synth:${weekKey}:${djb2(sig)}`;
   ```
2. Test by reloading without any data change — should hit cache. Then change one workstream's `last_touched` in the embedded JSON and reload — should miss cache.

**Risk:** Too coarse → stale synth output. The signature MUST include anything that affects synth output meaningfully. Test thoroughly: change `last_touched` on a workstream, change `latest_from_is_me` by sending a new Slack message, change ticket counts — each should bust the cache.

**Acceptance:** Reload Hub three times with no underlying changes — exactly 1 Haiku call (the first). Visible via the `Synthesized in Xs` diagnostic line — second and third should not show a fresh time.

---

### 2.3 Meeting transcript excerpts in the embedded JSON

**Why:** Right now the Hub has filenames + topics for meetings but never the BODY. So "what did we decide about Sigma?" can't be answered — the model is guessing from Slack and calendar metadata. Loading 800–1200 char excerpts from this week's transcripts dramatically improves week-recap and completed-actions accuracy.

**Where:**
- The embedded `<script id="optio-memory">` JSON's `meeting_transcripts` array (lines 941–991).
- `synthesize()` payload (line 1694) — include excerpts when present.

**Approach:**
1. Extend the `meeting_transcripts` schema: add an optional `excerpt: string` field per entry, max 1200 chars, ideally pulled from the transcript's "Summary" or "Decisions" section if present.
2. The Hub already has the topic; new field is purely additive.
3. In `synthesize()`, build a `meeting_excerpts` array on the payload only for transcripts dated within the current week range. Cap to 6 transcripts, 1000 chars each (≤6KB added to payload).
4. Add prompt instructions: "Plus `meeting_excerpts` — body excerpts from this week's meeting transcripts. Use these as primary evidence for week_recap.worked_on and completed_actions. Direct quotes are OK when relevant."
5. The Hub does NOT read transcripts at runtime in Phase 2. The user (or a scheduled task) populates `excerpt` fields by hand or via the daily-backfill skill.
6. As a separate ergonomic helper, add a small inline comment in the JSON schema documenting the new field.

**Risk:** Payload size — strict 6×1000 cap keeps it under 10KB added. Haiku's window is fine. Watch the diagnostic line — should still synthesize in <10s.

**Acceptance:** Hand-add `"excerpt": "..."` to 2-3 transcript entries in the embedded JSON. Reload. The Week So Far text references specific things said in those meetings (not just topic labels).

---

## Phase 3 — Deferred (not currently scoped)

The two items below were considered but deferred. The function-calling chat is real work for marginal day-to-day benefit given how the Hub is actually used. Cross-device sync only matters if the Hub is opened on multiple devices, which isn't a current need. Kept here for reference if priorities shift.

### 3.1 Function-calling chat panel

**Why:** Today the chat is "look up answers in a pre-bundled snapshot." It can't dynamically pull a specific Slack thread or transcript. Letting the LLM request reads turns it from a static-context lookup into an actual research assistant.

**Where:** `chatSend()` at line 3892.

**Approach:**
1. Extend the prompt's action list with READ actions in addition to the existing WRITE actions:
   ```
   - search_slack: { query, days_back? } → returns thread summaries
   - get_thread: { channel_id, ts? } → returns full message history
   - read_transcript: { file } → returns meeting transcript body
   - search_drive: { query } → returns recent files
   - get_ticket: { id } → returns ticket detail
   ```
2. Define each tool as an MCP wrapper: search_slack → `slack_search_public_and_private`, get_thread → already exists, read_transcript → needs a new MCP tool for local-file reads (use the existing file MCP `read_file_content` if it supports `file://` paths; if not, fall back to fetching the file via Drive search).
3. Modify the chat loop: parse model response, if `actions` contains read tools, execute them, append results as a system/tool message to history, send next turn to model with augmented context. Cap to 5 iterations to avoid runaway loops.
4. Visual: read actions don't render confirmation cards (auto-execute); only writes do.
5. Add a typing-indicator that shows "Looking up <tool_name>..." between turns.

**Risk:** Multi-turn conversation state. Cap iterations strictly. Show clear error if a read fails. Test with: "what did Magnus say about Sigma last Tuesday?" — should fire search_slack with relevant query, get a result, and answer with specifics.

**Acceptance:** Open chat. Ask a question that requires data not in the pre-bundled context (e.g., a Slack thread from 30 days ago). The chat fires a read action visibly, then answers with concrete details — not "I don't have access to that."

---

### 3.2 Cross-device sync via Drive

**Why:** localStorage + IndexedDB are per-device. Switching laptops loses Backfill, ignore lists, chat history. A tiny Drive JSON file as a sync target solves it without a backend.

**Where:** A new sync layer wrapping localStorage writes; initialization step in `main()`.

**Approach:**
1. Define a known Drive file name: `daily-hub-state.json` in the user's My Drive root.
2. State to sync: ignore map, dismissed actions, expanded sections, created actions, team selection, chat history. Skip cache entries (those are recomputable per device).
3. On init, attempt to read `daily-hub-state.json` from Drive. If present and newer than local state, merge in. If absent, no-op.
4. After any state-changing localStorage write, debounce 5 seconds, then upload state to Drive. Use the Drive MCP `create_file` with overwrite semantics, or look for an `update_file` tool — verify available tools.
5. Add a footer indicator: "Synced 2m ago" with last-upload timestamp; click to force a sync.
6. Conflict resolution: latest-write-wins by ISO timestamp embedded in the JSON.

**Risk:** Drive MCP latency could feel sluggish if synchronous. Debounce + fire-and-forget is essential. If Drive call fails, log to console and continue — don't block UI.

**Acceptance:** On laptop A, dismiss a Needs Reply card and create a Linear ticket. On laptop B (or after clearing local storage on A), reload Hub. The dismissed card stays dismissed; the created ticket shows in the local "created" list.

---

## Phase ordering rationale

- **Phase 1** is pure addition: zero impact on existing renders, zero state migration, no prompt restructuring. Three commits, each independently revertable.
- **Phase 2** introduces a state migration (snooze) and a prompt-payload change (meeting excerpts). Cache-key coarsening is the only one that could mask other bugs — keep it last in Phase 2 so issues elsewhere surface against the existing cache first.
- **Phase 3** is real engineering. Function-calling chat is its own product. Cross-device sync touches every storage write. Ship Phase 1 and 2 first to derisk the touchpoints.

## Execution

Each phase runs as a single coding-agent invocation against the artifact at `/Users/danielben-atar/Documents/Claude/Artifacts/daily-hub/`. The agent reads this PLAN.md, executes the phase's three items, verifies via grep + a final read of touched regions, commits, and pushes to `origin/main` on `DMB28th/daily-hub`.
