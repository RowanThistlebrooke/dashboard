# Tutorial prompt — paste this into Claude

Build a single self-contained HTML file called `dashboard.html`. It must work when double-clicked from the desktop or opened via VS Code Live Server — no build step, no npm, no external CSS or JS. All styles inline in `<style>`, all JavaScript inline in one `<script>` tag at the bottom.

The file has four components stacked vertically: a **Page title** at the very top, then a **Goal Ticker** strip, then a **Day Ring**, then a **To Do List** section.

---

## Visual style (whole file)

- Dark theme. Page background `#050506` with two soft radial washes layered on top: a warm orange wash `rgba(224, 118, 88, 0.16)` at 82% across / 14% down, and a cool grey wash `rgba(180, 180, 200, 0.06)` at 18% across / 90% down. Both blurred 40px and slowly drifting via a 36s alternating animation.
- A second `body::after` layer adds a tiny film-grain dot pattern (3px × 3px tile, white at ~1.4% opacity) so the dark never looks plastic.
- Body font: Apple system stack — `-apple-system, BlinkMacSystemFont, "Inter", "Segoe UI", Roboto, Helvetica, Arial, sans-serif`. Mono font (used for numbers, times, dates): `ui-monospace, "SF Mono", Menlo, Consolas, monospace`.
- Text colors as CSS variables: `--text-primary: #FAFAFA`, `--text-secondary: #B8B6B0`, `--text-tertiary: #76746E`.
- Semantic colors: `--success: #6BE3A4`, `--warning: #F2C063`, `--danger: #FF6B6B`.
- Card chassis: `rgba(255, 255, 255, 0.04)` background, no visible border, 16px radius, 18–22px padding, `backdrop-filter: blur(24px) saturate(1.2)`, soft shadow `0 12px 40px rgba(0,0,0,0.45)`.
- Body is centered, max-width 1100px, with safe-area-aware top padding.

---

## Component 1 — Page title

A single `<h1 class="dash-title">My Dashboard</h1>` at the very top of the body. Styled as:

- `font-size: 28px`, `font-weight: 700`, `letter-spacing: -0.025em`.
- A vertical white-to-soft-grey gradient masked into the text — set `background: linear-gradient(180deg, #FFFFFF 0%, #C7C4BC 120%)`, then `-webkit-background-clip: text; background-clip: text; -webkit-text-fill-color: transparent;` so the gradient shows through the letterforms.
- 14px bottom margin so it spaces nicely from the ticker that follows.
- On screens ≤ 480px: shrink to `font-size: 22px`.

---

## Component 2 — Goal Ticker (NASDAQ-style strip)

A horizontal strip below the title that cycles through today's pending goals one at a time, every 5 seconds, with vertical slide-in / slide-out animations. Looks like a stock ticker / LED board.

**Structure** (inside a wrapper `.ticker-row` with 18px bottom margin, 4px gap, single column):

```
<div class="goal-ticker" id="goalTicker" aria-live="polite" aria-atomic="true">
  <div class="goal-ticker-led"><span class="goal-ticker-led-dot"></span></div>
  <div class="goal-ticker-label">GOALS</div>
  <div class="goal-ticker-stage" id="goalTickerStage">
    <div class="goal-ticker-row">
      <span class="goal-ticker-status" data-status="">—</span>
      <span class="goal-ticker-text">Loading…</span>
    </div>
  </div>
  <div class="goal-ticker-meta" id="goalTickerMeta">0/0</div>
</div>
```

**Styling**

- `.goal-ticker` — flex row, 10px gap, 7px×12px padding, 12px radius, dark glass background using a layered gradient + repeating scan-line texture for the LED-board feel:
  ```
  background:
    linear-gradient(180deg, rgba(0,0,0,0.42) 0%, rgba(0,0,0,0.30) 100%);
  background-image:
    linear-gradient(180deg, rgba(0,0,0,0.42) 0%, rgba(0,0,0,0.30) 100%),
    repeating-linear-gradient(0deg, rgba(255,255,255,0.025) 0, rgba(255,255,255,0.025) 1px, transparent 1px, transparent 3px);
  box-shadow: inset 0 1px 0 rgba(255,255,255,0.04);
  position: relative; overflow: hidden;
  ```
- A `::after` pseudo-element drifts a soft white sweep across the strip every 8s (left -40% → 110%) — `linear-gradient(90deg, transparent, rgba(255,255,255,0.04), transparent)`, 30% wide, full height.
- `.goal-ticker-led-dot` — a 7×7 green dot (`#6BE3A4`) with `box-shadow: 0 0 8px rgba(107,227,164,0.7)` and a 1.6s pulse animation that drops opacity to 0.45 and scales to 0.85 at the midpoint.
- `.goal-ticker-label` — the word `GOALS`. 9.5px mono, weight 800, 0.18em tracking, tertiary text.
- `.goal-ticker-stage` — flex 1, 22px tall, `position: relative; overflow: hidden;` so rows can slide in/out within it.
- `.goal-ticker-row` — flex row inside the stage. 22px tall, 8px gap, 12.5px mono, weight 600, tabular nums, primary white text. `white-space: nowrap`.
- `.goal-ticker-status` — 18px wide flex slot for the status glyph. `data-status="done"` → `#6BE3A4`, `pending` → tertiary, `empty` → tertiary. Glyphs: `done` = `✓`, `pending` = `○`, otherwise `·`.
- `.goal-ticker-text` — flex 1, ellipsis on overflow.
- `.goal-ticker-meta` — the small right-side counter pill (`0/3`, `2/5`, etc.). 11px mono weight 700, tabular, secondary text, 0.04em tracking, 3px×8px padding, fully rounded, `rgba(255,255,255,0.04)` background.
- Two animations for the row swap:
  ```
  .goal-ticker-row.is-leaving  → ticker-leave 0.45s cubic-bezier(0.55, 0, 0.55, 1) forwards
  .goal-ticker-row.is-entering → ticker-enter 0.45s cubic-bezier(0.22, 1, 0.36, 1) forwards
  ```
  `ticker-leave`: opacity 1 → 0, translateY 0 → -100%. `ticker-enter`: opacity 0 → 1, translateY 100% → 0.
- On screens ≤ 480px: padding 9px×12px, label 9px / 0.14em, row text 12px, meta 10px / 2px×7px.

**Behavior**

- The ticker reads today's goal list (the same `goals:YYYY-MM-DD` localStorage key the To Do List uses) and builds an `items` array:
  - If `total === 0`: a single placeholder item `{ status: 'empty', text: 'No goals set for today — add one to get rolling.' }`.
  - Else if every goal is done: a single celebration item `{ status: 'done', text: '✓ All goals done — solid day.' }`.
  - Else: one item per pending (unchecked) goal — `{ status: 'pending', text: g.text }`. Done goals are skipped — once you check it off, it drops out of the rotation.
- The right-side `meta` pill always shows overall progress as `done/total` (e.g. `2/5`), even when only pending goals are rotating through, so completed work is still visible.
- `tick()` shows the current item, advances `cycleIdx`, and updates `meta`. Each call swaps the row: stamp the existing row with `is-leaving` (and remove it from the DOM after 460ms), append a fresh row with `is-entering`. On the very first render there's no leaving row, so just drop the entering one in without the animation class.
- `start()` runs `tick()` immediately, then `setInterval(tick, 5000)`.
- **Stay in sync with edits.** When the To Do List `storeSet`s any `goals:`-prefixed key, fire a custom event:
  ```
  window.dispatchEvent(new CustomEvent('goals-changed'));
  ```
  The ticker listens for `goals-changed` and re-runs `tick()` immediately (resetting `cycleIdx` to 0) so adding/checking/deleting/reordering a goal updates the strip instantly instead of waiting up to 5 seconds.

---

## Component 3 — Day Ring

A circular SVG progress ring that fills throughout the day, paired with a text column on its right.

**Structure**
- A flex container, centered, with 26px gap, items wrap on narrow screens.
- LEFT: a 168×168px square containing an SVG ring (viewBox 0 0 120 120) plus an absolutely-positioned text overlay centered inside it.
- The SVG has two circles, both `cx=60 cy=60 r=52`:
  1. Track: `fill: none; stroke: rgba(255,255,255,0.06); stroke-width: 8;`
  2. Fill: same geometry but `stroke-linecap: round`, with `stroke-dasharray` and `stroke-dashoffset` set by JS, rotated `-90` around `(60, 60)` so 0% starts at the top, and a soft `feGaussianBlur` glow filter applied.
  3. Both stroke color and offset have a 0.7s `cubic-bezier(0.22, 1, 0.36, 1)` transition.
- Inside the ring (centered, stacked vertically, `pointer-events: none`):
  - **Percentage**: 40px, weight 800, tabular-nums, `letter-spacing: -0.04em`.
  - **Phase label**: 9.5px mono uppercase, weight 800, `letter-spacing: 0.16em`, tertiary text, 5px above margin.
  - **Live clock**: 10.5px mono, tertiary text.
- RIGHT (max-width 280px column, 6px gap):
  - **Status line**: 14px primary text, weight 700.
  - **Remaining time**: 12px mono, secondary text.
  - **Hours range**: 11px mono, tertiary text — static text `8:00 AM – 12:00 AM`.
- On screens ≤ 480px: ring shrinks to 144×144, percentage to 32px, status to 13px.

**Behavior**
- The "awake window" runs 8:00 AM → midnight (16 hours = 100%). Define `WAKE_HOUR = 8` and `SLEEP_HOUR = 24` as constants at the top so they're easy to tweak.
- Compute current hours as `now.getHours() + now.getMinutes()/60 + now.getSeconds()/3600`.

  | Time | Behavior |
  |---|---|
  | Before 8 AM | Empty ring, dim slate stroke `#4D4B47`, percentage shows `—`, phase `SLEEPING`, status `😴 Still sleeping`, remaining shows `Xh Ym until wake-up`. |
  | 8 AM – midnight | Fill the ring proportionally. `percent = (hours - 8) / 16 * 100`. Stroke color is interpolated from a 9-stop sun-cycle palette (see below). Phase + status by quartile. Remaining shows `Xh Ym awake time left`. |
  | After midnight (shouldn't normally hit before resetting) | Ring full, stroke `#E25D7A`, phase `PAST BEDTIME`, status `⚠️ Past bedtime`, remaining shows `Sleep!`. |

- **Sun-cycle palette stops** — interpolate linearly between the two adjacent stops based on the current percent:
  ```
  0%    [255, 216, 158]   morning gold
  12.5% [255, 205, 121]   warm gold
  25%   [255, 227, 143]   bright midday
  37.5% [255, 183, 106]   peach
  50%   [255, 149,  89]   amber
  62.5% [243, 111,  79]   sunset orange
  75%   [226,  93, 122]   sunset pink
  87.5% [123,  91, 176]   twilight purple
  100%  [ 47,  58, 102]   deep night blue
  ```

- **Phase + status by quartile** (when in awake window):
  - `< 25%` → `MORNING` / `☀️ Morning — fresh start`
  - `< 50%` → `MIDDAY` / `⚡ Midday — keep moving`
  - `< 75%` → `AFTERNOON` / `🔥 Afternoon — push it`
  - `< 90%` → `EVENING` / `⏳ Evening — wrap up`
  - else → `BEDTIME` / `🌙 Bedtime soon`

- Live clock format: 12-hour with AM/PM, e.g. `9:49 AM` (no leading zero on hour, two-digit minutes).
- Ring circumference = `2 * Math.PI * 52`. Set `stroke-dasharray` to that. Set `stroke-dashoffset` to `C * (1 - percent/100)`.
- Run the update once on load, then `setInterval(updateDayBar, 60 * 1000)`.

---

## Component 4 — To Do List

Wrap in `<div class="section">`. The section header is a small uppercase eyebrow that reads `To Do List` (NOT "Goalmaxxing" — the section title is literally the words "To Do List"). The eyebrow has a short dash before the text and a fading horizontal line after it:

```
.section-title {
  font-size: 10.5px; font-weight: 700;
  letter-spacing: 0.18em; text-transform: uppercase;
  color: var(--text-tertiary);
  display: flex; align-items: center; gap: 12px;
}
.section-title::before { content: ''; width: 18px; height: 1px; background: var(--text-tertiary); opacity: 0.6; }
.section-title::after  { content: ''; flex: 1; height: 1px; background: linear-gradient(90deg, rgba(255,255,255,0.08), transparent); }
```

Inside the section, two cards stacked: **TODAY** and **PLAN TOMORROW**.

### TODAY card

Header row (flex, space-between, wrap, 14px bottom margin):
- LEFT column:
  - Eyebrow: `Today — {Sat, May 9}` — formatted as `{weekday-short}, {month-short} {day}`. ID `todayLabel`. 10.5px uppercase tertiary, 0.18em letter-spacing, 6px bottom margin.
  - Progress row (flex, baseline-aligned, 6px gap):
    - Big number `gmProgressNum` — 42px, weight 700, `letter-spacing: -0.045em`, tabular-nums.
    - `gmProgressTotal` — `/ 0`, 18px mono, tertiary.
    - `gmProgressLabel` — small uppercase, 11px, weight 600, 0.10em tracking, tertiary. Reads:
      - `no goals yet` when total = 0
      - `complete` when in progress
      - `all done — solid day` when total > 0 and all done.
- RIGHT: streak pill `gmStreak`, an inline-flex pill with 6px gap, 8px×12px padding, fully rounded:
  - Default: `rgba(255,255,255,0.04)` background, tertiary text.
  - Active (streak count > 0): `rgba(242,192,99,0.10)` background, `#F2C063` text, `0.32` border-color, the bolt icon gets a `drop-shadow(0 0 6px rgba(242,192,99,0.6))`.
  - Contents: `⚡` (13px), then `gmStreakNum` (mono tabular, weight 700), then label `day streak` (uppercase, 0.10em tracking).

Segmented progress bar `gmBar` — flex row, 4px gap, 6px tall, 16px bottom margin. One `<div class="gm-bar-seg">` per goal. Done segments get `gm-bar-seg-done`: `background: #6BE3A4; box-shadow: 0 0 6px rgba(107,227,164,0.40)`. Empty bar (no segments) hides via `.gm-bar:empty { display: none; }`.

Goal list `<ul id="goalList" class="goal-list gm-list">`. Empty state `<div id="emptyState" class="empty-state">No goals for today yet — add one below.</div>` — 12px tertiary italic, 14px vertical padding, centered.

Each goal row is a flex item with 12px gap, 12px×14px padding, 6px bottom margin, `rgba(255,255,255,0.035)` background, 12px radius, hairline border `rgba(255,255,255,0.06)`. On hover: lighter background, drag handle and delete button fade in. Contents in this order:
1. **Drag handle** `⋮⋮` — 14px wide, hidden until hover (opacity 0 → 1), grab cursor, `letter-spacing: -2px` so the dots tighten.
2. **Custom checkbox** — 22px square, 7px radius, 1.5px border `rgba(255,255,255,0.18)`, dark inner background. When checked: `#6BE3A4` background, glow `0 0 12px rgba(107,227,164,0.40)`, and a ::after rotated checkmark that pops in via 0.28s `cubic-bezier(0.34, 1.56, 0.64, 1)` scale animation.
3. **Goal text** — flex 1, click to edit inline (sets `contentEditable="true"`, adds outline, Enter commits, Escape cancels).
4. **⚡ Queue button** `.gm-queue-btn` — toggles a "queued for productivity window" flag. Default: tertiary, 0.55 opacity. Active: `#F2C063` with a `drop-shadow(0 0 4px rgba(242,192,99,0.65))`. Tapping triggers a 0.48s flash animation `gm-queue-flash` (background pulses to `rgba(242,192,99,0.32)` and scales to 1.015).
5. **× delete button** `.goal-delete` — tertiary, hover red, 0.5 opacity until row hover.

Done rows: 0.45 opacity, green-tinted background `rgba(107,227,164,0.04)`, text gets `line-through` with `text-decoration-color: rgba(255,255,255,0.4)`.

Queued rows: yellow-tinted background `rgba(242,192,99,0.10)` with a `inset 3px 0 0 0 #F2C063` left accent stripe, text color `#FFE2A8`.

When all goals are checked: the card itself gets `.gm-all-done` — adds a soft green radial gradient at the top, `.gm-progress-num` and `.gm-progress-label` turn green.

If goals length > 5: render only the first 5, then a dashed-border "Show N more ▾" toggle row that expands to show the rest (and switches to "Show less ▴").

After the list:
- **Push remaining button** `gmPushBtn` — shown only when there's at least one unchecked goal. Full-width, dashed border, tertiary text, hover solidifies to primary. On click: confirm prompt, then move every unchecked goal into the tomorrow list (skip duplicates by exact text match), then strip them from today (keeping only the checked ones).
- **Quick-add row** `.goal-input-wrap.gm-input-wrap` with a 14px top border + 14px top padding so it feels separated from the list:
  - Text input `goalInput` — flex 1, 11px×14px padding, 12px radius, glass background, white-on-focus border, placeholder `Add a goal for today…`.
  - **+ Add button** `goalAddBtn` — primary white pill: `linear-gradient(180deg, #FFFFFF 0%, #E8E5DD 100%)` background, `#0A0A0B` text, weight 700, 11px×20px padding, multi-layer shadow including inset top highlight. Hover: lifts 1px and brightens.
  - **✨ Polish button** `goalPolishBtn` — secondary glass pill: `rgba(255,255,255,0.04)` background, primary text, 1px hairline border `rgba(255,255,255,0.10)`, otherwise same shape as Add.
- A 11px tertiary status line `polishStatus` for transient messages.

### PLAN TOMORROW card

Same chassis with class `gm-card gm-card-tomorrow`. The progress row is hidden (`.gm-card-tomorrow .gm-progress-row { display: none; }`). Header has:
- Eyebrow `tomorrowLabel` → `Plan tomorrow — {Sun, May 10}`.
- Below it, sub-text `Write tonight, locked until 6 AM.` — 12px tertiary.
- Right: count badge `gmTomorrowCount` reads `0 planned` / `3 planned` etc. — 11px mono uppercase tabular tertiary.

Goal list, empty state `Nothing planned for tomorrow yet`, and identical quick-add row (with `tomorrowInput`, `tomorrowAddBtn`, `tomorrowPolishBtn`, `tomorrowStatus`). The difference: tomorrow rows render in **read-only mode** — checkboxes are disabled with title `Activates at 6 AM tomorrow`, and the ⚡ button is disabled. Inline edit, drag-reorder, and × delete all still work.

---

## Logic & state

All persistence in `localStorage` only. Key shape: `goals:YYYY-MM-DD` → array of `{ text, done, doneAt?, queued? }`. Streak state under `goal_streak_v1` → `{ count, lastProcessedDate }`.

Helper functions:
- `storeGet(key)` → `JSON.parse` or null.
- `storeSet(key, value)` → `localStorage.setItem` JSON-stringified.
- `storeDelete(key)` and `storeListKeys(prefix)` over `localStorage.length`.

**Active date logic** — the dashboard treats 6 AM as the day boundary, not midnight. So:
- `getActiveDateString()`: if current `getHours() < 6`, subtract a day. Otherwise today's calendar date. Format `YYYY-MM-DD`.
- `getTomorrowDateString()`: if current hours < 6, return today's calendar date (because the active day is yesterday). Otherwise tomorrow's calendar date.
- `formatDate("YYYY-MM-DD")` returns `Sat, May 9` style — used for both eyebrows.

**Rollover** — runs once on load. Walks every `goals:` key strictly older than the active date, takes any undone goals and pushes them into today's list (deduped by exact text), then deletes the old day's record entirely.

**Streak check** — runs once on load. Walks every `goals:` key older than today, in date order, starting from `lastProcessedDate`. For each: if 0 goals → skip (don't break the streak); if all done → +1; else → reset to 0. Persist the new count and date.

**Render functions:**
- `renderTodayHeader()` — re-reads today's goals, updates the big number, total, label, fills the segmented bar, toggles `gm-all-done` on the card, shows/hides the push button.
- `renderStreak()` — updates the streak number and toggles `gm-streak-active`.
- `renderTomorrowCount()` — updates the `N planned` badge.
- `renderListInto(goals, listEl, emptyEl, key, readOnly)` — clears the `<ul>`, builds rows via `buildGoalRow`, applies the show-more collapse if > 5, calls the appropriate header renderer at the end.
- `loadToday()` and `loadTomorrow()` — re-read storage and call `renderListInto`.

**Row interactions** are wired in `buildGoalRow`:
- Checkbox change → set `goals[idx].done = cb.checked`, stamp `doneAt = Date.now()` when checking, delete it when unchecking, save, re-render.
- Inline edit (`makeInlineEdit`) — click sets `contentEditable=true`, focuses, places caret at end. Blur or Enter commits (saves only if text changed and isn't empty); Escape cancels.
- Drag (`wireDragReorder`) — HTML5 drag/drop. On drop, splice from-index out, splice into to-index, save, re-render. Add a top-border indicator on the drag-over row.
- Queue button click → toggle `queued`, save, add `is-queue-flashing` class on the row, then re-render after 480ms so the user sees the pulse before the row rebuilds.
- Delete → splice out at idx, save, re-render.

**Add + Polish** wired by a shared `makeAddHandlers(input, addBtn, polishBtn, key, statusEl, reload)`:
- Add: trim input, push `{ text, done: false }`, save, clear input, reload.
- Polish: at the very top of the JS, declare `const ANTHROPIC_API_KEY = '';`. If empty → fall back to plain Add and show a brief tertiary message in `statusEl` saying `Polish needs an Anthropic API key — added as-typed.` for 3.5 seconds. If a key is set → POST to `https://api.anthropic.com/v1/messages` with headers `Content-Type: application/json`, `x-api-key: <KEY>`, `anthropic-version: 2023-06-01`, `anthropic-dangerous-direct-browser-access: true`. Body: model `claude-sonnet-4-5`, `max_tokens: 1000`, single user message asking the model to clean up exactly ONE goal and return it as a one-element JSON array of strings (no preamble, no fences). Parse, push, save, clear input. On any error: add the raw text and show `Polish failed — added as-typed.` in red for 3.5s.
- Enter in the input fires Add (not Polish).

After both handlers wired: call `loadToday()` and `loadTomorrow()`. Run `renderStreak()` once. Run the day-ring update once and start the 60-second interval.

---

## Acceptance checklist

- File runs from a `file://` URL or VS Code Live Server with no errors in console.
- The page opens with a gradient `My Dashboard` headline at the top.
- The goal ticker shows a green pulsing LED, the word `GOALS`, the current pending goal in mono, and a `done/total` pill on the right. It cycles to the next pending goal every 5 seconds with a vertical slide animation. Adding, checking, deleting, or reordering a goal updates the ticker immediately (not on the next 5s cycle).
- When all goals are checked, the ticker shows `✓ All goals done — solid day.` instead of an empty rotation. With zero goals, it shows `No goals set for today — add one to get rolling.`
- Day ring shows the correct percentage at the current time, with the right phase label and color.
- Adding a goal to Today shows it in the list; checking it greens out the row, fills its segment, and updates the counter.
- Hitting all goals turns the whole card green-tinted and changes the label to `all done — solid day`.
- Inline-edit, drag-reorder, queue-flash, delete, push-remaining, and tomorrow-list (read-only checkboxes) all work as described.
- Refreshing the page restores all state from localStorage.
- The section title reads exactly `To Do List` — capital T, capital D, capital L — not "Goalmaxxing".
