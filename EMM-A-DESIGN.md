# EMM-A System Design Document

## 1) Purpose and Scope

EMM-A is a two-script Space Engineers automation system:

- **NOOK-Ca**: carrier/coordinator script (state authority, data persistence, route and live dock telemetry publisher).
- **NOOK-Pa**: pathfinder/miner script (operator-facing setup, path recording, autopilot execution, dock/undock behavior).

This document describes the system as implemented in the current repository version.

---

## 2) High-Level Architecture

### 2.1 Roles

- **Carrier (Ca)**
  - Stores setup data in block `CustomData` INI sections.
  - Resolves asteroid/zone context.
  - Accepts setup and workflow commands via argument + IGC.
  - Broadcasts confirmations, return paths, speed settings, live dock data.

- **Pathfinder (Pa)**
  - Captures operator positions/orientations for setup.
  - Drives autopilot routines (auto-return, undo-return, undock, dock, automine path follow).
  - Requests path and live dock data from Ca.
  - Displays status via LCD/light/sound feedback.

### 2.2 Communications

Both scripts support dual channels for compatibility:

- `EMM-A Ca_PathFinder_Channel`
- `ANOUK-Ca_PathFinder_Channel` (legacy)

Ca and Pa each broadcast to both and listen on both.

---

## 3) Block Naming Contracts

## 3.1 NOOK-Ca expected blocks

- Remote Control: `EMM-A Ca Remote Control` (legacy fallback: `ANOUK-Ca Remote Control`)
- Field DB timer: `EMM-A Ca A-Field DB` (legacy fallback: `ANOUK-Ca A-Field DB`)
- Carrier connector: `EMM-A Ca Connector` (legacy fallback: `ANOUK-Ca Connector`)
- Asteroid DB timer prefix: `EMM-A Ca <Zone> A-DB HD...` (legacy prefix accepted)
- LCD prefix: `EMM-A Ca LCD Data`

## 3.2 NOOK-Pa expected blocks

- Remote Control: `EMM-A PF Remote Control`
- Connector: `EMM-A PF Connector`
- Sensor: `EMM-A PF Sensor`
- Indicator light: `EMM-A PF Indicator Light`
- Sound block: `EMM-A PF Sound Block`
- LCDs: `EMM-A PF LCD 1`, `EMM-A PF LCD 2`

---

## 4) Persistent Data Model (Ca)

Ca persists asteroid/site data into the active asteroid DB timer `CustomData` INI section.

Typical keys:

- `Status`: `SetUp`, `pending`, etc.
- `OreTypes`
- `ExclusionZone`
- `currentSiteN`
- `ExcavationSiteEntry1` (first-site flow)
- `ExcavationSitePath{N}_{i}`
- `ExcavationSiteStagger{N}`
- `ExcavationSite{N}`

Value conventions:

- Entry/path/stagger store `GPS|orientation` where orientation is `yaw,pitch,roll` in degrees.
- Final site stores `orientation|cornerA|cornerB`.

---

## 5) IGC Message Schemas

## 5.1 Ca -> Pa

- `Confirmed_Entry|<fieldKey>|<gps>|<orient>`
- `Path|<gps>|<orient>|<close>|<gps>|<orient>|<close>|...`
- `PathfinderMaxSpeed|<number>`
- `LiveDockData|<entryGps>|<entryOrient>|<connectorGps>|<entryForwardVec>|<entryUpVec>|<connForwardVec>|<connUpVec>|<approachDistance>`

## 5.2 Pa -> Ca

- Setup/workflow commands as plain strings, including payload variants:
  - `ExcavationSiteEntry|...`
  - `ExcavationSitePathAdd|...`
  - `ExcavationSiteStagger|...`
  - `ExcavationSiteSetUpComplete|...`
- Control requests:
  - `PathfinderReturnRequest`
  - `PathfinderMaxSpeedRequest`
  - `LiveDockDataRequest`
  - `AsteroidSetUpComplete`
  - `AsteroidSetUpCancel`
  - `ExcavationSitePathUndo`
  - `Reset`

---

## 6) Command Reference

## 6.1 NOOK-Ca commands

### Direct/argument commands

- `SendConfirm`
- `PathfinderMaxSpeedRequest`
- `LiveDockDataRequest`
- `Reset`
- `AsteroidSetUpStart`
- `AsteroidSetUpCancel`
- `AsteroidSetUpComplete`
- `ExcavationSiteNextPath`
- `ExcavationSiteCancel`
- `ExcavationSitePathUndo`
- `PathfinderReturnRequest`
- `SetExclusionZone=<int>`
- `Ore<CommaSeparatedList>` (e.g. `OreIron,Nickel`)
- `ExcavationSiteEntry|<gps>|<orient>`
- `ExcavationSitePathAdd|<gps>|<orient>`
- `ExcavationSiteStagger|<gps>|<orient>`
- `ExcavationSiteSetUpComplete|<orient>|<cornerA>|<cornerB>`

### IGC-dispatched command coverage

Ca maps incoming IGC strings to the same handlers for all setup/control messages above (including `Reset`, `PathfinderReturnRequest`, and `LiveDockDataRequest`).

## 6.2 NOOK-Pa commands

### Setup and recording

- `PF_ExcavationSiteSetUpComplete` (alias: `EMMA_ExcavationSiteSetUpComplete`)
- `PF_ExcavationSitePathAdd` (alias: `EMMA_ExcavationSitePathAdd`)
- `PF_ExcavationSiteStagger` (alias: `EMMA_ExcavationSiteStagger`)
- `PF_ExcavationSiteCancel` (alias: `EMMA_ExcavationSiteCancel`)
- `PF_ExcavationSitePathStartRecording` (alias: `EMMA_ExcavationSitePathStartRecording`)
- `PF_ExcavationSitePathStopRecording` (alias: `EMMA_ExcavationSitePathStopRecording`)

### Setup lifecycle controls

- `PF_AsteroidSetUpComplete` (alias: `EMMA_AsteroidSetUpComplete`) -> forwards `AsteroidSetUpComplete`
- `PF_AsteroidSetUpCancel` -> forwards `AsteroidSetUpCancel`

### Navigation / data requests

- `PF_ReturnRequest` (alias: `EMMA_ReturnRequest`) -> `PathfinderReturnRequest`
- `PF_MaxSpeedRequest` (alias: `EMMA_MaxSpeedRequest`) -> `PathfinderMaxSpeedRequest`

### Autopilot and utility

- `PF_Undock`
- `PF_Dock`
- `PF_AutoMineStart`
- `PF_AutoMineStop`
- `PF_ExcavationSitePathUndo`
- `PF_ExcavationSitePathUndoAndReturn`
- `PF_Reset` (also relays `Reset` to Ca)

---

## 7) Workflow Sequences

## 7.1 Asteroid/site setup flow

1. Operator starts setup on Ca (`AsteroidSetUpStart`).
2. Ca creates/sets setup context and auto-creates `ExcavationSiteEntry1` for first site.
3. PF sends path points via `PF_ExcavationSitePathAdd`.
4. PF sends stagger via `PF_ExcavationSiteStagger`.
5. PF runs 3-step `PF_ExcavationSiteSetUpComplete` capture and sends final site payload.
6. Ca confirms saves via `Confirmed_Entry|...` broadcasts.

## 7.2 Auto return after setup complete (PF)

After step-3 completion, PF starts auto-return to last stagger point while holding locked setup orientation. On arrival it restores pilot control and starts recording mode.

## 7.3 Undo and reverse return (PF)

`PF_ExcavationSitePathUndoAndReturn`:

1. Sends `ExcavationSitePathUndo` to Ca.
2. Removes last local checkpoint.
3. Builds reverse chain from local path history to stagger (or Entry1 fallback).
4. Drives checkpoint-by-checkpoint, aligning to stored orientation per checkpoint.

## 7.4 Dock/undock flow

### Undock

`PF_Undock`:

- Disconnects connector if needed.
- Moves straight away for configured approach distance.
- Then transitions toward Entry1 if available.

### Dock

`PF_Dock`:

- Requests `LiveDockData` from Ca.
- Requires being within Entry1 50m bubble.
- Goes to a staging point in front of live connector, then final approach.
- Attempts `connector.Connect()` when within `CONNECTOR_APPROACH_CLOSE` and status is connectable.

---

## 8) AutoMine (Current Implementation)

Current `PF_AutoMineStart` behavior:

1. Sets pending state.
2. Requests return path from Ca (`PathfinderReturnRequest`).
3. On `Path|...` receipt, starts waypoint follow.

Current `ApplyAutomineFollow` is **path follower only**:

- Uses drill-center offset compensation (`GetDrillCenterOrRemote`).
- Faces movement direction.
- Drives toward waypoint with per-point close distance.
- Stops when path is complete.

### Important: what AutoMine is **not** yet

- No lane generator from excavation rectangle.
- No depth stepping automation.
- No drill-inventory ore continuation logic.
- No built-in end-of-path auto PF_Dock handoff.

---

## 9) Known Gaps / Technical Debt

1. Mission-state logic is spread across many booleans (disjoint mode management risk).
2. AutoMine and dock/undock are not unified under one mission state machine.
3. Ca path payload ordering may not match all PF handoff expectations.
4. PF does repeated full block scans (`thrusters`, `drills`) in active loops (instruction/perf pressure on larger grids).

---

## 10) Recommended Refactor Direction

1. Introduce a single PF mission enum state machine:
   - `Idle`, `Undock`, `TransitToEntry`, `Mine`, `Return`, `Dock`, `Abort`.
2. Make AutoMine completion transition to dock state automatically.
3. Split Ca route intents (transit path vs mining path) or tag path points by role.
4. Add lane/depth mining planner using stagger-side rectangle normal and 5m depth steps.
5. Add cached block registries with periodic refresh rather than per-tick global scans.

---

## 11) Command Examples

## 11.1 Typical setup and mission sequence

1. On Ca:
   - `AsteroidSetUpStart`
2. On PF:
   - `PF_ExcavationSitePathAdd`
   - `PF_ExcavationSitePathAdd`
   - `PF_ExcavationSiteStagger`
   - `PF_ExcavationSiteSetUpComplete` (run 3 times)
3. On PF:
   - `PF_AsteroidSetUpComplete`
4. Mission:
   - `PF_Undock`
   - `PF_AutoMineStart`
   - (optionally) `PF_Dock`

## 11.2 Maintenance / control examples

- Cancel current site: `PF_ExcavationSiteCancel`
- Undo last path point: `PF_ExcavationSitePathUndo`
- Undo + reverse return: `PF_ExcavationSitePathUndoAndReturn`
- Reset both scripts from PF: `PF_Reset`

---

## 12) Troubleshooting Checklist

- Confirm block names exactly match configured constants.
- Verify both scripts are running and listening on same channel pair.
- Check Ca has valid field DB and zone DB timers.
- Check `Path|...` is actually received (`Path stored.` echo on PF).
- For docking, verify `LiveDockData` receipt and Entry1 bubble gate.
- If autopilot cancels unexpectedly, check pilot input noise thresholds.

---

## 13) Version Note

This document reflects the script behavior currently present in this repository at the time of writing, including dual legacy channel compatibility and the existing AutoMine-as-path-follow implementation.
