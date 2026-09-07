# PRD — Sentry Access Automation

## Problem

Submitting a Sentry IDC Enter/Leave access request for the MY88 datacenter requires manually
stepping through a multi-field form (Direction → Data Center → Campus → Building → Room →
Access Control Location → Document Association → Submit) every time someone enters or leaves
a room. Each field opens a search drawer that needs the right value typed and selected. This is
slow and repetitive for people who visit the same handful of rooms daily.

## Goal

A Tampermonkey userscript that automates filling in the form for a known room + direction
(Enter/Leave), leaving only the final Submit as an explicit user action unless Auto Submit is
turned on. Runs entirely client-side in the browser, on desktop and mobile.

## Non-goals

- Not a replacement for the Sentry portal — it drives the existing form via DOM automation,
  it does not call any API directly.
- Not built for datacenters/rooms outside the configured `CONFIG.tree` — unlisted rooms must be
  added to config before they can be automated.
- Does not automate the "Call Security" phone-call feature or other non-Enter/Leave workflows.

## Users

Engineers/ops staff who physically visit MY88 datacenter rooms and need to file an Enter or
Leave access request from their phone or laptop before/after each visit.

## Functional requirements

### 1. Floating Action Button (FAB)
- A round orange-bordered button fixed to the bottom-right corner of the page.
- Tapping/clicking it opens the room menu; tapping again (or tapping elsewhere that closes it)
  closes the menu.
- Also reachable via the Tampermonkey extension menu command **"Open Sentry Menu"**.

### 2. Room menu
Opened from the FAB, shows (top to bottom):

- **Quick Action** — repeat the last logged room with the opposite direction (if history exists
  a one-tap button flips Enter→Leave or Leave→Enter for the same room).
- **Favorites** — starred rooms, shown with Enter/Leave buttons.
- **All rooms**, grouped by building, each with a favorite star toggle and Enter/Leave buttons.
- **Recent** — last 6 history entries with relative timestamps ("14:32", "Yesterday 09:10", or a
  full date).
- **Settings** — three toggles (Fast waits, Auto Submit, Debug logging) and a Clear History
  button.

### 3. Workflow automation
When a room + direction is chosen, the script runs a step sequence, driving the live DOM:
1. **Load Form** — if the "direction" field isn't present (e.g. the automation was started from
   the app hub page instead of the actual form), navigate to the configured form URL and wait
   for the field to appear.
2. **Direction** — click the Enter/Leave radio.
3. **Data Center** — open drawer, search/select the configured datacenter (e.g. MY88).
4. **Park / Campus** — open drawer, select the campus tied to the room.
5. **Building** — open drawer, select the building tied to the room.
6. **Room** — open drawer, select the room code.
7. **Access Control Location** — open drawer, select the per-door label
   (`<room>.<datacenter>-<door>号门外`), falling back to the bare room code if that exact label
   isn't found.
8. **Document Association** — click "No".
9. **Submit** *(only if Auto Submit is enabled)* — click the Submit button.

Each step is shown live in a status panel (bottom-left) with pending/active/done/error icons.

### 4. Error handling & retry
- Each step gets one silent automatic retry on failure.
- If it fails twice, the status panel shows the error message with **Retry** (re-run from the
  failed step) and **Continue manually** (dismiss and let the user finish by hand) buttons.
- A **Stop** button is always visible while a run is in progress, cancelling the workflow
  before/during the next step.

### 5. History
- Every completed run (reaching the end of the steps, not necessarily Submit) is recorded:
  room, direction, timestamp. Capped at the last 20 entries.
- Drives the Quick Action shortcut and the Recent list in the menu.
- Can be cleared from Settings.

### 6. Favorites
- Any room can be starred/unstarred from the menu; favorited rooms get their own section at the
  top of the room list for faster access.

### 7. Add Room (in-menu configuration)
- A form in the menu lets you add a room without editing the script: Data Center, Campus,
  Building, Room, and an optional Access Control Location door number (defaults to 1). Campus
  doubles as both the searchable code and the menu's display label — no separate label field.
- Added rooms are persisted via `GM_setValue` (separate from the hardcoded `CONFIG.tree`) and
  merged into the same menu grouping and workflow resolution as built-in rooms — no visible
  difference once added.
- Duplicate room codes (already in `CONFIG.tree` or previously added) are rejected with an
  inline error.
- **Clear added rooms** in Settings wipes everything added this way (including any door
  overrides entered for them) — it does not touch the hardcoded `CONFIG.tree`.
- Each room added this way also gets **Edit** and **Delete** buttons next to its Enter/Leave
  buttons (wherever it's listed, including Favorites). Built-in rooms from `CONFIG.tree` never
  show these — they can only be changed by editing the script.
- **Edit** loads the room's saved values back into the Add Room form (now labeled "Edit Room")
  so any field — including the room code itself — can be changed; **Save Changes** replaces the
  old entry, and **Cancel** discards the edit. Renaming a room carries its favorite star over to
  the new code.
- **Delete** removes the room immediately, along with its door override (if any) and its
  favorite entry (if starred).

### 8. Settings (persisted via `GM_setValue`/`GM_getValue`)
| Setting | Default | Effect |
|---|---|---|
| Fast waits | On | Uses shorter timing between DOM steps. Turn off if the page feels laggy and steps start failing due to timing. |
| Auto Submit | Off | When on, the workflow clicks Submit automatically at the end instead of stopping before it. |
| Debug logging | Off | Logs each step's progress to the browser console, prefixed `[SentryAuto]`. |
| Clear history | — | Wipes the run history (and the Quick Action/Recent sections it drives). |
| Clear added rooms | — | Wipes rooms added via the Add Room form and their door overrides. |

## Configuration (developer-facing, in `CONFIG` at the top of the script)

- `tree` — datacenter → campus → building → room-code list. Add/remove rooms under an existing
  datacenter and nothing else needs to change. Add a whole new datacenter by adding a new
  top-level key with its own `campuses` map (see the commented example above `tree` in the
  script) — the menu automatically groups rooms by datacenter once it's added.
- `doorOverrides` — room code → door number, for rooms whose Access Control Location isn't the
  default door 1. Keyed by plain room code, so room codes must be unique across every
  datacenter in `tree`, not just within one.
- `tagOverrides` — room code → literal search text, for rooms whose Access Control Location
  label doesn't follow the standard `<room>.<datacenter>-<door>号门外` pattern. Same
  cross-datacenter uniqueness requirement as `doorOverrides`.
- `formUrl` — the URL of the actual Enter/Leave form, used as a recovery target if the workflow
  starts on a different page (e.g. the app hub).

## Constraints / known fragility

- Automation is DOM-selector-based (`.mt-radio-container`, `.mt-drawer-modal`,
  `.mt-list-item-title`, `div.mt-button--primary-solid`, etc.) — a redesign of the Sentry portal
  UI will break it and require selector updates.
- Access Control Location matching depends on the confirmed per-door label format; if the portal
  changes that format, `tagOverrides`/`fallbackTag` matching may need revisiting.
- `clickSubmitButton`'s selector is explicitly flagged as an assumption not yet verified against
  every form variant — Auto Submit should be spot-checked after enabling.

## Success criteria

- A user can go from "on the app hub page" to "form filled, ready for Submit" in under ~10
  seconds for a known room, with zero manual field entry (2 taps: open menu, tap Enter/Leave).
- Failures surface clearly in the status panel with a way to retry or fall back to manual entry,
  never silently fail.
