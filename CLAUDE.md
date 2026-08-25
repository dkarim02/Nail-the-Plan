# Nail the Plan — Claude Code Project Memory

This file is read automatically at the start of every Claude Code session. Do not delete it.

---

## What this project is

Browser-based daily planning and bridge-note tool for Nordstrom DC499. Supervisors enter planned vs. actual volume, headcount, and financial figures for each department and shift. The app builds formatted EOD bridge notes and posts them to Microsoft Teams via Adaptive Cards.

**GitHub repo:** dkarim02/Nail-the-Plan
**Live (GitHub Pages):** dkarim02.github.io/Nail-the-Plan
**Local:** C:\projects\Side gigs\Nail the Plan

No backend, no build system — single self-contained HTML file. All JS, CSS, and data embedded in `index.html`. `dc499-nail-the-plan 31.html` is the versioned working copy; `index.html` is always kept in sync with it for GitHub Pages.

---

## Architecture

### State & Storage
- All data lives in browser `localStorage` under key `ntp_dc499_v1`
- `STATE` shape: `{ data, notes, procNotes, meta }`
- `STATE.data[area][process][shift|checkpoint]` = `{ plan_u, act_u, plan_f, act_f, plan_hc, act_hc, plan_ind_hc, act_ind_hc, recirc, shiftHours, totalHours, indirectHours, note_u }`
- `reconcile()` runs on load — handles area/process renames, fills missing keys from SEED defaults
- `resetPersonHoursIfNewDay()` wipes ALL data fields + notes + autoSyncedBlocks when the plan date rolls over (PST, before 3 AM counts as previous day)

### Areas & Processes
```
AREAS = ["Retail","Ecom","DC Shipping","DC Receiving","Returns","Item Prep"]
PROC_BRIDGE = ["Ecom","Retail","Item Prep"]  — bridge notes are per-process, not rolled up
```
Retail: Retail OUT, Retail IN
Ecom: Packing, Picking, Replen, Putaway, Sorting, FC Shipping
DC Shipping: DC Shipping (has Recirc %)
DC Receiving: DC Receiving
Returns: Returns
Item Prep: Flat, GOH, Shoes

### Checkpoints
`SHIFTS = ["1st","2nd"]`, `CPS = ["Midday","EOD"]` — keys are `"1st|Midday"`, `"1st|EOD"`, etc.

### PPH Scores (three per checkpoint)
- **Planned PPH** = `plan_f / (shiftHours × plan_hc)` — baseline target
- **Static PPH** = `act_u / (shiftHours × act_hc)` — actual units per HC hour
- **Fin PPH** = `act_f / totalHours` — financial units per total hours (manually entered)

`indirectHours` shows as a % of `totalHours`. `indirectHours` is entered per checkpoint (Mid/EOD), not per shift.

### Roll-up logic
- Per shift: uses EOD if both plan+actual entered, else Midday, else skipped
- Area roll-up: sums across processes
- Day roll-up: sums across areas
- Status: ≥100% = make, ≥90% = near, <90% = behind

---

## Shared Sync (JSONBin)

All departments use the same JSONBin bin to share numbers across browsers/devices.

**Bin URL:** `https://api.jsonbin.io/v3/b/6a740381f5f4af5e29f11192` (stored as `DEFAULT_JSONBIN_URL`)
**Master key:** stored as `JSONBIN_KEY` constant in the file

### Push (`pushToJsonBin(area)`)
- Fires automatically after any Teams post (`postAreaToTeams`)
- Also fires from the nav "Push" button (pushes current area)
- Reads existing bin, merges in this area's data, writes back
- Payload per checkpoint includes ALL fields: `plan_u, act_u, plan_f, act_f, plan_hc, act_hc, plan_ind_hc, act_ind_hc, recirc, shiftHours, totalHours, indirectHours, note_u`
- Only checkpoints with at least one non-null value are included

### Manual Sync (`showSyncModal()` → "Sync depts" button)
- Fetches bin, shows checkbox per area
- Areas with local actuals show "Has local data" badge — syncing them does a **force overwrite** (all fields)
- Areas without local data default checked — sync fills empty fields only
- Home dept (⚑ flag) is always immune regardless

### Auto-sync (`autoSyncFromJsonBin()`)
- Fires once per block deadline window (0–10 mins after block closes)
- Uses **fill-only** (`force=false`) — never overwrites fields the local user has entered
- Home dept always immune
- Tracks fired blocks in `STATE.meta.autoSyncedBlocks` (keyed by block key + date)

### Home dept flag
- Set by clicking ⚑ on a nav item — stored in `STATE.meta.homeDept`
- That area is NEVER overwritten by sync (manual or auto), no matter what

---

## Posting to Teams

### Webhooks
All webhook URLs are static constants baked into the file — `DEFAULT_WEBHOOK_1ST`, `DEFAULT_WEBHOOK_2ND`, `DEFAULT_WEBHOOK_SUBMIT`, `DEFAULT_WEBHOOK_SEND`, `DEFAULT_WEBHOOK_CHECK`. They are never stored in `STATE` or localStorage. The Webhooks ▾ dropdown is read-only (shows "Configured" for each entry). To update a webhook, change the constant in the source and redeploy.

`sendReport()` must use `DEFAULT_WEBHOOK_CHECK` (not `STATE.meta.checkStatusWebhook`) for the status check — always fall back to the constant or leaders without legacy localStorage will see all depts as Pending.

### Posting flow
`postAreaToTeams(area)` → builds Adaptive Card via `buildAdaptiveCard()` → posts to the current shift's webhook → silently calls `pushToJsonBin(area)`. Post Late also calls `pushToJsonBin(current)` after sending.

`postToTeams` treats HTTP 200/202 as confirmed success. Status 0 (CORS-blocked response) shows "confirm it arrived in Teams" rather than a false success toast.

### Adaptive Card
Built by `cardBlocksForBlock(block, areas)` → `cpBlocks(area, proc, sh, cp)`. Includes volume columns, HC FactSet, financial FactSet, PPH line, note. No emoji in card text.

### Block timer / Post Late
- `BLOCKS` = [{1st Mid: 3:00–10:30}, {1st EOD: 10:31–14:10}, {2nd Mid: 15:00–20:00}, {2nd EOD: 20:01–1:30}] (PST/PDT)
- Countdown shows in nav bottom block; turns amber at <10 min, red/pulsing at <5 min
- If outside all windows, "Post Late" button appears — opens modal to pick a reason before sending

---

## Day Reset

`resetPersonHoursIfNewDay()` runs on page load. When plan date rolls to a new day (PST, before 3 AM = previous day):
- Wipes all `plan_u, act_u, plan_f, act_f, plan_hc, act_hc, plan_ind_hc, act_ind_hc, recirc, shiftHours, totalHours, indirectHours, note_u` across all areas
- Clears `STATE.notes`, `STATE.procNotes`, `STATE.meta.autoSyncedBlocks`
- Sets `STATE.meta.lastResetDate = today`
- Sets `_dayResetFired = true` → a toast fires 400ms after load: "New day — all entries cleared for YYYY-MM-DD"

SEED data (historical example entries) is the fallback default — loaded when localStorage is empty. It does NOT reset to SEED on a new day; fields are wiped to null.

---

## File structure

- `index.html` — the entire app, single canonical file. Edit this directly.
- `.nojekyll` — tells GitHub Pages to skip Jekyll and serve files directly (no Docker/build step).
- `README.md` — public-facing feature list and usage notes
- `CLAUDE.md` — this file

No versioned copies. `index.html` is what GitHub Pages serves at `dkarim02.github.io/Nail-the-Plan`.

---

## Status codes / thresholds

| Status | Condition | Color |
|--------|-----------|-------|
| make | ≥ 100% | Green |
| near | 90–99% | Amber |
| behind | < 90% | Red |
| na | No plan or no actual | Gray |

Recirc (DC Shipping only): ≤ 10% = make, > 10% = behind.

---

## Disclaimer (required on all pages)

```
This tool measures throughput only and may not be used to evaluate, coach, or hold team members accountable on performance.
```

---

## How this project works day-to-day

- Supervisors open the live GitHub Pages URL on their phone or PC
- Each dept sets their home dept flag (⚑) so their data is never overwritten
- They enter Mid-Day and EOD numbers, then post to Teams — this auto-pushes to the shared store
- Other depts auto-sync after block deadlines, or manually sync via "Sync depts"
- Data is local to each browser; export JSON to back up or transfer a day's data
