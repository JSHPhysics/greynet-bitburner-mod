# Greynet — work log

Pickup notes for Claude (and me, on another machine) resuming work on the
prototype. New session entries go at the top.

## Files in this repo

- **`index.html`** — the prototype, single self-contained file
- **`SPEC.html`** — technical walkthrough. Last refreshed before the
  splash + perf + animation work below; still useful for high-level
  structure but stale on those topics
- **`WORK-LOG.md`** — this file; chronological session notes

## Session 2026-05-20 → 2026-05-21

A long session covering UI rework, code cleanup, three rounds of
performance work, and animation polish. Shipped in four commits — the
final batch is in the commit that introduced this work log.

### Splash flow + handler character

- Replaced the dense ONBOARDING modal with a two-step splash:
  - **Difficulty picker** (Default / Hard) with an animated typed welcome
  - **Handler-voiced brief** — paragraphs type in one at a time, in a
    blue `.handlermsg` palette
- Splash gated by `SPLASH_VERSION` in the save blob. Existing players
  see it once on next visit; bump the version to force a re-show after
  a future rewrite of the briefing
- "change difficulty" and "replay intro" links live in the top strip

### Handler character

The handler is now a visually consistent character across the brief
modal and the cliff / flagged post-mortems. Post-mortems were rewritten
in handler voice and stripped of the explicit `cd` / `cat` command hints
— the player has to find the right doc themselves.

### Hard mode

Sidebar shows just "1. Stop the heart. 2. Don't trigger any alarms or
failsafes." instead of the five-clause checklist. `paintConditions`
bails when `hardMode` is true.

### Terminal niceties

- **Tab autocomplete** — first token completes command names, then
  context-aware arguments (sealed nodes for `breach`, breached for `cd`,
  files for `cat`/`nano`, editable files only for `nano`). One match
  completes + trailing space; many matches extend to the shared prefix
  and list the options above the prompt.
- **Up/Down command history** — 50-entry cap, dedups consecutive
  duplicates, preserves the in-progress draft when scrolling away and
  restores it on scroll back. Session-scoped (not persisted).
- `#termout` capped at 500 lines so the DOM doesn't grow unbounded over
  long sessions.

### Cybernetic sweep

"Mechanical heart" → "cybernetic heart implant" everywhere — code, docs,
modals, spec.

### Code cleanup

- `TUNING` constants block at the top of the script for scattered magic
  numbers (tick rate, budget, BPM thresholds, trace bands, etc.)
- `parseConfig(file, txt)` helper — collapses duplicate regex parsing
  between nano save and `ns.greynet.writeConfig`
- `requireBreachedFile` / `requireBreachedNode` helpers — collapse the
  "is this node breached, does this file exist" boilerplate in
  `cat` / `nano` / `cd` / `ls`
- `setScore` split into `setScore` + `paintBestLabels`
- Removed unused `PANEL_STATE.CLEARED`

### Performance — three passes

**First pass:**
- `drawBloodflow`: squared-distance check skips sqrt for ~99% of
  cell-vs-point pairs; `prevLit` list so clearing the prior frame walks
  ~50 cells instead of all ~3,500
- `flashFailsafe` toggles the rate panel's `.fighting` class directly
  (was going through `positionHotzones`, which forced layout and
  re-positioned all four panels on every failsafe correction)
- Gutter drag coalesces mousemoves into one update per RAF
- `paintTelemetry` caches last-written values, quantises bar widths to
  integer percent
- Typing animation rewritten on `requestAnimationFrame` (was bottle-
  necked by `setTimeout`'s ~4 ms minimum clamp + drift + per-character
  paragraph reflow)
- Handler-intro modal pre-measures its final body height so paragraphs
  fill pre-allocated space — no jerky modal growth

**Lag-over-time bug fix:**
- `checkSignature` was O(n) — it walked the entire stream every tick.
  At tick 2,400 that's 2,400 iterations × 18 Hz. Replaced with three
  latched flags (`sigBpmRose`, `sigBpmCliffed`, `sigHadHighVar`)
  updated incrementally as each frame is pushed. Now O(1). ~360× less
  work across a full 2,400-cycle run.

**Second pass (Factorio-mode):**
- Spatial index by row in `drawBloodflow` — `heartCellsByRow[r]` is
  built at render time so the trace can walk only the cells near each
  pulse (~5,250 visits per frame vs ~73,500)
- Pre-built 8-step palette of `rgb(...)` / `text-shadow(...)` strings —
  zero string allocations per frame in the hot loop
- Removed `.05s` CSS transitions on heart-cell glyphs (~900 transition
  lifecycles per second saved)
- 1024-entry pulse curve LUT replaces 2× `Math.exp()` per `drawHeart`
- `checkSignature` reuses a single result object — zero alloc per tick
- Lazy idle — skip drawing when any modal covers the heart
- `paintTelemetry`'s inner `set` helper hoisted to module scope

### Animation decoupling

Heart visual now drives off a `requestAnimationFrame` render loop —
runs at display refresh rate (60+ Hz). Sim state still updates at 18 Hz
via `stepHeart`, but `beatPhase` is advanced by real elapsed milli-
seconds in the render loop. Smooth motion between sim ticks. Skipped
while a modal covers the heart (`lastRenderTs` reset to `null` on resume
so the heart doesn't lurch by the modal's duration).

### Live frame-time HUD

Small `X.Xms` readout in the top strip between meta-links and zoom
controls. EWMA-smoothed; the displayed value is repainted at 5 Hz so
the last digit doesn't jitter. Measures `renderLoop` cost — the actual
"is the animation smooth" number. No toggle, no label — reads as part
of the system chrome.

### Polish

- **Heart-rate match**: the `beatPhase` formula was ~1.8× too fast
  (`hw.bpm/600` per 55 ms tick worked out to ~131 visual BPM when the
  readout said 72). New formula `dt*hw.bpm/60000` makes the heart beat
  at exactly the displayed BPM.
- **Palette bump**: `--ink` (`#3fae4a` → `#5fd070`) and `--dim`
  (`#1f6e2a` → `#3da050`) brightened for readability. Hierarchy
  unchanged (hi > bright > ink > dim).
- `CHARS_PER_MS` settled at `0.18` (~180 chars/sec, comfortable read).

## Reference

### Voice + tone conventions

- **Player-facing text**: handler voice — direct, slightly cynical
  contractor, plain English, occasional dark joke. No spelling out
  solutions in post-mortems; nudge toward the right doc.
- **Game terms**: "cybernetic heart implant" (never "mechanical"),
  "contract", "operative", "the unit", "the implant". No coding jargon
  in player-facing strings.
- **Code comments**: explain *why*, not *what*. Note non-obvious
  decisions, leave the obvious to the code.
- **Visual character**: blue `.modal.handlermsg` for the handler; green
  (default) for system/unit voice; red for alarms (cascade).

### Key tuning constants

All at the top of the script:

- `TUNING.TICK_MS = 55` — sim tick rate (ms)
- `TUNING.TICK_BUDGET = 2400` — cycles per run before timeout
- `TUNING.HEALTHY_BPM = 72`, `TUNING.ARREST_BPM = 42`
- `TUNING.CLIFF_DROP = 14`, `TUNING.SMOOTH_VAR = 0.15`
- `TUNING.TRACE_SCRUTINY = 33`, `TUNING.TRACE_FLAGGED = 66`
- `TUNING.TRACE_FLAG_TICKS = 60` — ticks of held FLAGGED before run ends
- `CHARS_PER_MS = 0.18` — typing animation pace
- `SPLASH_VERSION = 1` — bump to force splash re-show for everyone

## Things deferred / open

- **Stream array elimination.** `stream.push` still allocates per tick.
  After the O(1) signature refactor, only `s[0].bpm` (could be a
  `firstBpm` variable) and `s[n-2].bpm` for cliff detection (could be
  `prevBpm`) are still read. Removing the array would zero out the last
  per-tick allocation. Worth doing if memory growth ever becomes a
  concern; bigger refactor than it sounds.
- **Path completion across nodes** in tab-complete (e.g.
  `cat n_rate/r<Tab>` from root). Player can just `cd` in first; the
  helper for path-as-prefix completion isn't there.
- **Frame cap on high-refresh displays.** Render loop runs at display
  rate, so on a 144 Hz monitor the per-frame budget is ~7 ms. If the
  HUD shows numbers creeping toward half that, throttle to 60 Hz with
  `if (ts - lastRenderTs < 14) return;` near the top of `renderLoop`.
- **The RAM score axis is still mock** (`2.0 + lines * 0.15`). Doesn't
  trade off against the other three axes the way a real model would.
  Design call to make.
- **SPEC.html refresh.** Predates everything in this work log. Still a
  useful architecture overview; treat its specifics as a snapshot from
  before this session.

## Picking up

1. Clone https://github.com/JSHPhysics/greynet-bitburner-mod
2. Skim the last few commits for context
3. Read this file
4. Open `SPEC.html` for the high-level structural walkthrough (with the
   staleness caveat above)
