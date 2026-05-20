# SDF Gate Plan View

A year-round gate planning view for Louisville Muhammad Ali International (SDF / KSDF). Ingests Diio Mi daily-extract schedule files and renders an assignable gate plan with live occupancy scorecards. Derby is the stress test; the daily Tuesday is the product.

## Goal

Replace SDF's hand-maintained Excel gate plan with a tool that:

1. Reads the Diio daily-extract `.xlsx` SDF already pulls.
2. Auto-assigns flights to gates per the configured operational rules.
3. Lets ops scrub through any date in the loaded extract and see occupancy at any point in time.
4. Surfaces open gate slots (and gates about to open) so the planner can re-slot manually when needed.
5. Lets ops edit the gate ruleset (leased / common-use / inactive / equipment caps) without touching code.

## Project artifacts

| File | What it is |
|---|---|
| `Derby_2026_Gate_Plan.html` | Existing visual mock — **the design system**. Match its dark theme, type, color tokens, Gantt layout, controls bar, and tooltip patterns. |
| `Derby 2026 Gate Plan Official (revision 2.0).xlsx` | SDF's current manual plan. Reference only — captures gate-to-gate tow workflow, hardstand markers, and free-text Notes that v1 should preserve. |
| `Schedule_Daily_Extract_Report_36500.xlsx` | Sample Diio daily-extract. **Authoritative input format.** Bidirectional (~570 dep + ~566 arr over 7 days). |

## Data: Diio daily-extract schema

Header row lives on row 5 (0-indexed 4). Data starts row 6. Columns:

| Col | Field | Notes |
|---|---|---|
| A | Date | datetime, midnight of op-day |
| B | Mkt Al | marketing carrier (AA, DL, UA, WN, G4, MX, …) |
| C | Alliance | oneworld / SkyTeam / Star / — |
| D | Op Al | operating carrier — **this is what flies the airplane** (RPA, EDV, SKW, ENY, OH, MQ, etc. for regionals) |
| E | Orig | 3-letter; will be SDF or the away station |
| F | Dest | 3-letter; will be SDF or the away station |
| G | Kilometers | great-circle |
| H | Flight | numeric, no carrier prefix |
| I | Stops | usually 0 |
| J | Equip | 3-char IATA equipment code (E75, 319, 738, CR9, …) |
| K | Seats | numeric |
| L | Dep Term | terminal at origin |
| M | Arr Term | terminal at destination |
| N | Dep Time | "HHMM" string, local at origin |
| O | Arr Time | "HHMM" string, local at destination |
| P | Block Mins | numeric |
| Q | Arr Flag | 0 same-day, +1 next-day, etc. |
| R | Orig WAC | world-area code |
| S | Dest WAC | world-area code |

Rules for ingestion:
- A row is an **SDF arrival** when `Dest == "SDF"` and an **SDF departure** when `Orig == "SDF"`.
- Pair arrivals → departures into **turns** by `Op Al` + `Equip` + tightest temporal fit (dep after arr, gap within plausible ground-time band). Unpaired arrivals → **RON_IN**; unpaired departures → **RON_OUT**.
- Times in the extract are local-at-station. Normalize to SDF local (EDT/EST) for plotting; for SDF-side legs that's already correct.
- "HHMM" strings: pad/parse defensively — extract has `"705"` mixed with `"0705"`, plus a few cells stored as int.

### Overnight pairing (cross-day)

After same-day turn pairing, run a second pass across consecutive op-days:
- For viewed date D, pair `RON_IN[D]` arrivals with `RON_OUT[D+1]` departures by `Op Al + Equip + plausible overnight gap (60 min ≤ gap ≤ 16 h)`.
- For viewed date D, pair `RON_IN[D-1]` arrivals with `RON_OUT[D]` departures by the same rule.
- Each match represents one aircraft staying overnight — the same gate is occupied from `arr_time[D]` through `dep_time[D+1]`.
- Display: paired RONs get a small `⇌` icon and the tooltip surfaces the counterpart leg ("Arrived yesterday DL 3093 from ATL @ 21:19" / "Departs tomorrow DL 2407 to ATL @ 06:00").
- Unpaired RONs are flagged as **orphan** (dotted border, red tooltip note). Expect orphans on the first day (no prior data) and last day (no next data) of an extract, and a small number on middle days where an aircraft swaps equip / op carrier / or doesn't actually overnight.

## Features in scope (v1)

### 1. Date selector
Top-bar control: **Today** button + date picker. Range is bounded by the dates present in the loaded Diio extract. Selecting a date rebuilds the plan for that op-day.

### 2. Gate plan Gantt
Match the existing mock: concourse A / B tabs, gate rows, 5-min-resolution time axis from 04:30 to 01:30 next-day, airline color tokens, RON bars at edges, turn bars in the middle, hover tooltip with route+equip+seats+turn duration. Click-to-pin detail panel as in the mock.

### 3. Occupancy scorecard (above the Gantt)
Live count tied to the **time cursor** (default = now if viewing today; otherwise noon of the selected date, with a slider to move it). Shows:
- Gates **occupied** (count + list on hover)
- Gates **open** (count + list on hover)
- Gates **inactive** (per rules tab)
- Optional: % utilization

Updates as the cursor moves.

### 4. Open-slots scorecard (below the Gantt)
Two adjacent panels:
- **Available now** — gates currently open. For each: gate id, time it's been open, time until next occupancy (next-arrival ETA). Sorted by longest-open-duration descending.
- **Opening soon** — gates that will free up within the next N minutes (configurable, default 60). For each: gate id, current occupant, time until departure, expected duration of the open window.

Both panels rebuild as the time cursor moves.

### 5. Gate rules tab
Separate tab/route in the same shell. Lists every gate with editable attributes:

| Attribute | Type | Example |
|---|---|---|
| Gate ID | string (immutable) | `A3`, `B16A` |
| Concourse | enum | `A`, `B` |
| Status | enum | `active`, `inactive`, `maintenance` |
| Use type | enum | `leased`, `common-use`, `preferential` |
| Leaseholder | carrier code (if leased) | `DL` |
| Max equipment | IATA equip code or size class | `B739`, or `narrowbody` |
| Has jet bridge | bool | true |
| Hardstand / remote | bool | false |
| Notes | free text | matches the legacy "Notes:" column |

Edits persist to a local JSON config (v1 — no backend). The auto-assigner re-runs when rules change.

### 6. Auto-assignment
Deterministic pass that maps flights → gates given the ruleset. v1 heuristic:

1. **Filter** gates to those whose status is `active` and whose `max equipment` is compatible with the flight's equipment.
2. **Prefer** leaseholder match (leased gate → that carrier's flights only; common-use → any).
3. **Pack** turns greedily by earliest-arrival, into the first gate where the turn fits without conflict (arrival - buffer >= prior departure + buffer).
4. **RON** flights park at their last-used gate when possible, otherwise the lowest-utilization compatible gate.
5. **Unassigned** flights surface in a "needs attention" panel rather than failing silently.

Buffer / min-ground-time defaults editable in the rules tab.

## Out of scope (v1, revisit later)

- Live status feeds (ASDI/SWIM, FlightAware, ACARS)
- Tail-number tracking
- Charter / GA / military / cargo ops
- Multi-user editing, auth, audit log
- IROPs replan / what-if mode
- SSIM ingest for season-level planning
- Export back to the legacy Excel format
- FIDS / GIDS / airline-portal integrations
- Mobile / tablet layouts (desktop only for v1)

## Tech & layout conventions

- **Single-file HTML** like the existing mock — vanilla JS, no build step, no framework. Add a sibling `gate_rules.json` for the editable ruleset and a `data/` folder for dropped-in Diio `.xlsx` files.
- **Excel parsing in-browser** via SheetJS (`xlsx.full.min.js`) loaded from CDN. User drops or selects a file; no upload to a server.
- **Design system**: re-use the CSS custom properties from `Derby_2026_Gate_Plan.html` (`--bg`, `--accent`, the per-carrier `--c-*` tokens, JetBrains Mono / Space Grotesk, the `.btn` / `.tab` / `.stat` patterns). Don't redesign.
- **No backend** in v1. Rules persist to `localStorage` and can be exported/imported as JSON.

## File layout

```
/Users/willlivsey/Desktop/SDF/
├── CLAUDE.md                                     ← this file
├── Derby_2026_Gate_Plan.html                     ← design reference
├── Derby 2026 Gate Plan Official (revision 2.0).xlsx   ← reference only
├── Schedule_Daily_Extract_Report_36500.xlsx      ← sample Diio input
└── (to build)
    ├── index.html                                ← the app
    ├── gate_rules.default.json                   ← seed ruleset
    └── data/                                     ← Diio drops live here
```

## Working agreements

- Don't generalize past v1. Three concrete features (Gantt, two scorecards, rules tab) plus a date picker. No premature abstractions for the out-of-scope list.
- Don't refactor the mock's CSS — copy it forward and extend.
- Time math is in **SDF-local minutes-since-midnight** as the mock already uses (`bar_start`, `bar_end`). Keep that convention.
- Test against the provided Diio file; if the schema drifts in a real SDF extract, surface a clear parse error instead of guessing.
- Ask before adding dependencies beyond SheetJS.
