# Scrapyard `game/` Architecture Refactor Plan

> **PLAN ONLY.** Nothing in `aasumitro/bbmvc` was modified. This document is the blueprint for a later, incremental implementation.
>
> - **Repository analysed:** `aasumitro/bbmvc`, branch `main`, commit `f28082e` (2026-09-30, "Ask before Google Analytics…"). The product in the code is called **Scrapyard**. The brief calls it "BBMV".
> - **Scope:** `game/src/` (the client) and `game/server/` (the authoritative game server), both one npm package.
> - **Method:** I read the source, the repo's own architecture docs in `.claude/work/arch/`, `net/`, `mm/` and `nakama-mm/`, and both `AGENTS.md` files. A script built the import graph over all 119 `.ts`/`.tsx` files. I ran the full baseline (`lint`, `tsc -b`, `npm run check`, including `server:check`) on a local clone.

---

## Table of contents

0. [Read this first](#0-read-this-first)
1. [Current architecture](#1-current-architecture)
2. [God objects](#2-god-objects)
3. [Dependency graph](#3-dependency-graph)
4. [Architectural layers](#4-architectural-layers)
5. [Contracts and interfaces](#5-contracts-and-interfaces)
6. [Plug-and-play content](#6-plug-and-play-content)
7. [Game mode architecture](#7-game-mode-architecture)
8. [Client / simulation / rendering separation](#8-client--simulation--rendering-separation)
9. [Networking architecture](#9-networking-architecture)
10. [Server architecture](#10-server-architecture)
11. [Definition vs runtime state](#11-definition-vs-runtime-state)
12. [Registry architecture](#12-registry-architecture)
13. [Testing architecture](#13-testing-architecture)
14. [Dependency enforcement](#14-dependency-enforcement)
15. [File-by-file migration map](#15-file-by-file-migration-map)
16. [Migration phases](#16-migration-phases)
17. [Behaviour that must not change](#17-behaviour-that-must-not-change)
18. [High-risk areas](#18-high-risk-areas)
19. [Avoiding over-engineering](#19-avoiding-over-engineering)
20. [Target directory tree](#20-target-directory-tree)
21. [Architecture decision summary](#21-architecture-decision-summary)
22. [Final recommendation](#22-final-recommendation)
- [Appendix A: Import graph (current)](#appendix-a-import-graph-current)
- [Appendix B: Reproducing the analysis](#appendix-b-reproducing-the-analysis)

---

## 0. Read this first

### 0.1 The main finding: the premise is partly out of date

The brief assumes the codebase needs to be *restructured into* a layered architecture. The evidence says most of the requested boundaries **already exist in code**, but not in folder names.

A refactor on 27 Sept 2026 (`.claude/work/arch/REFACTOR_REPORT.md`) already did the following:

- Split a monolithic 838-line `match.ts` into:
  - `simulation.ts`: authoritative, headless, player-agnostic.
  - `view.ts`: Three.js, audio and camera; implements `SimEvents`.
  - `pilot.ts`: the local control source.
  - `feed.ts`: the kill feed and announcer.
- Replaced about 12 `mode === 'tdm'` branches with a `MatchMode` contract and a `MODES` registry. The engine no longer branches on the mode.
- Introduced id-keyed registries: `WEAPONS`, `VEHICLES` + `MODELS`, `MAPS`, `MODES`.
- Seeded every gameplay random stream from the match seed, so a bots-only match replays bit for bit.
- Moved the runtime out of React (`runtime.ts`). React only mounts and disposes it.

The online work then built on those seams:

- `MatchSource` lets practice and online share one "player side" (`playMatch`).
- The server runs **the same** `simulation.ts`, modes and bots.
- Replays re-run a room to the bit.

Moving everything into the proposed 9-layer tree (`app/ application/ domain/ engine/ networking/ presentation/ platform/ content/ shared/`) would therefore mostly **relabel boundaries that already hold**. The cost would be about 65 file moves (all of `src/game/`) plus import rewrites in 32 other files, 7 architecture docs and 2 `AGENTS.md` code maps invalidated, the balance probes and a deploy smoke script broken, and `git blame` harder to follow. **I do not recommend a big-bang restructure.** [High confidence: I read the refactor report and checked every claim against the current source.]

### 0.2 What actually blocks "plug-and-play", in priority order

| # | Problem (evidence) | Why it matters | Verdict |
|---|---|---|---|
| 1 | **No cross-commit behaviour pinning.** `simulation.check.ts` compares two runs *in the same process* (`first === replay(7)`), and `server.check` compares a room with the bare simulation *in the same build*. A refactor that changes gameplay deterministically passes every check. Only arena layout is pinned across commits (`server/digests.json`). | Every refactor phase needs a tripwire first. | **REFACTOR (Phase 0)** |
| 2 | **Boundaries are enforced by convention only.** The server imports `src/game/*`, and a DOM shim (`server/headless.ts`) keeps the arena builders alive in Node. `audio.ts` calls `window.addEventListener` *at import time*: one wrong import into a server-reachable module crashes the server at startup. `MODULE_BOUNDARIES.md` says the graph was "checked with an import-graph scan", but no such script exists in the repo or in CI, so it was a one-off manual check. | Multi-developer and multi-agent work will regress this. | **REFACTOR (Phase 1)** |
| 3 | **Weapon identity is recovered from the turret model.** `weaponId = WEAPONS.find(w => w.model === spec.model)` (`net/protocol.ts:175`), because bots carry a *scaled copy* of the spec (`ai.ts botGun`). Two weapons sharing a turret model would be reported as the same weapon on the wire, in match records, in replays and in the HUD. | A latent correctness bug that the next weapon will hit. | **REFACTOR (Phase 2)** |
| 4 | **Adding a weapon edits core files.** The turret is chosen by a ternary (`vehicle/vehicle.ts:195`: `weapon === 'rocketPod' ? buildRocketPod() : buildMinigun()`). The HUD icon is two hard-coded SVGs switched by `data-kind` (`hud/Hud.tsx:235–241`). `WeaponSpec.model` is a closed union type. | Contradicts the primary goal directly. | **REFACTOR (Phase 2)** |
| 5 | **Mode-specific UI is concentrated in two files, plus one server branch.** `Hud.tsx` has 42 lines that reference `ffa`/`tdm`; `Results.tsx` has 26. `server/room.ts:219` has `if (kind !== 'tdm') return ''` (team chat). `MapSelect` already shows a disabled "Custom — coming soon". | A third mode edits `Hud.tsx`, `Results.tsx` and `room.ts`. | **REFACTOR (Phase 4)** |
| 6 | **The wire-event codec is split across client and server, with positional indices.** The encoder is in `server/recorder.ts` and the decoder in `net/client.ts` (`f[11]`, `f[0] === me`…). | A new `SimEvents` callback touches 3 files, and a silent index mismatch is possible. | **REFACTOR (Phase 3)** |
| 7 | **The server ignores the chosen vehicle.** `room.join` keeps the roster's vehicle. `server/server.check.ts:365` *deliberately fails* if `VEHICLES` gets a second entry. Bots always use `BOT_VEHICLE`. The garage has no vehicle pager. The `ro` wire message and the replay `join` line carry no vehicle. | A second vehicle is a **feature** (it needs a `PROTOCOL` bump), not a refactor. | **Postpone (Phase 7)** until a second vehicle is scheduled |
| 8 | **`src/game/` is a 67-file folder mixing five concerns:** pure rules, the Rapier simulation, Three.js presentation, browser runtime, and the online adapter. `game/runtime.ts` and `game/online.ts` import `net/`, while `net/client.ts` imports `game/`. This is a folder-level inversion, not a file cycle. | Navigation; knowing what the server may import. | **REFACTOR, mechanical (Phase 5)**, after the fixes above |

Everything else is either **KEEP** (and good) or **REVIEW** (questionable, not clearly harmful). Details follow.

### 0.3 Baseline (measured on commit `f28082e`, Node 22.22, local clone)

| Command | Result |
|---|---|
| `npm run lint` | pass; 1 pre-existing warning (`screens/Drawer.tsx:24`, react-hooks exhaustive-deps) |
| `npx tsc -b` | clean |
| `npm run check` | **all pass**: router ok · ffa 143 · tdm 163 · simulation 28 · loading 23 · protocol 70 · chat 14 · matchmaker 63 · fairplay 10 · bots 21 (kills a minute: easy 7.8, normal 13.8, hard 17.8) · arena 64 (digests `b0ce6b61` / `9896c223`) · server 133 · client 32 · netplay 33 |
| Server bundle | main shared chunk 4,435 kB, because the server bundles Three.js and every visual arena builder (see §3) |
| Import cycles | **0**, including type-only imports (script in Appendix B) |
| Code size | about 19.2k lines of production TS/TSX (`src/` + `server/`, excluding checks); about 3.9k lines of `.check.ts` |

CI runs Node 24 (`.github/workflows/ci.yml`); I ran Node 22.22. Everything passed on 22.22, so the Node version has no bearing on today's results. It can matter for the golden fingerprints proposed in Phase 0 (§18).

### 0.4 Labels used

- **KEEP**: sound as it is; no refactor needed.
- **REVIEW**: questionable but not clearly harmful; investigate before changing.
- **REFACTOR**: change it, for the concrete reason given.
- **[High / Medium / Low confidence]**: based on what I read or measured. "Needs verification" means I have not proven it.

---

## 1. Current architecture

### 1.1 Folder overview

```
game/
├── src/
│   ├── main.tsx, App.tsx, analytics.ts, index.css   app shell: screen state machine, loadout, pick, online seat
│   ├── screens/   (13 files)  React screens: Loading, MainMenu, Garage, MapSelect, GameCanvas (gameplay screen),
│   │                          Results, Matchmaking overlay, Settings/PatchNotes panels, Menu/Drawer/Notice primitives
│   ├── hud/       (3 files)   Hud.tsx (per-frame imperative DOM writes), minimap.ts (canvas), Chat.tsx
│   ├── game/      (67 files)  everything else about the game (below)
│   └── net/       (11 files)  protocol, socket, client mirror, prediction, interpolation, matchmaking store,
│                              Nakama session, chat
└── server/        (22 files)  authoritative game server: door + loop, lobby, matchmaker, room, recorder,
                               rewind, fair-play, records, replay, headless arena builds, auth
```

### 1.2 What each area owns today

| Area | Owns | Key dependencies | Assessment |
|---|---|---|---|
| `src/` root | `App.tsx`: screen state (`loading → menu → garage → map-select → game`), `Loadout`, the mode/map pick, the online `Link`. `main.tsx`: mounts React and lazily imports `App`. | screens, `game/loadout`, `game/maps`, `game/modes`, `net/matchmaking` | **KEEP**. Small; React state holds only UI choices. |
| `src/game/` sim part | `simulation.ts` (combatants, fixed step, firing, rockets, damage, wrecks, respawn, stuck recovery), `combat.ts` (weapon specs + trigger + hitscan), `physics.ts` (Rapier init, static world), `vehicle/drive.ts` (ray-cast car), `ai.ts` (bots), `scoring.ts`, `rng.ts`, `mode.ts` (contract), `roster.ts`, `loadout.ts` | Rapier; Three.js **math** (`Vector3`/`Quaternion`) | **KEEP** the design. Headless and seeded; proven by 3 check suites. |
| `src/game/` modes | `ffa/`, `tdm/`: config, pure rules, adapter (`mode.ts`), tactics/items, checks; `modes.ts` registry | sim contract, `scoring.ts` | **KEEP** the design; UI leaks and one server leak remain (§7). |
| `src/game/` content | `arena/*` (contract + 2 maps + kit), `vehicle/*` (spec registry, model, parts, drive), `materials/*` (library, GPU bake, recipes, facade shader), `geometry.ts`, `maps.ts` | Three.js, the material library → `renderer.ts` | **KEEP** mostly; registration leaks for weapons (§6). |
| `src/game/` presentation | `view.ts`, `effects.ts`, `audio.ts`, `sounds.ts`, `camera.ts`, `environment.ts`, `postprocessing.ts`, `renderer.ts`, `turntable.ts`, `feed.ts`, `pilot.ts`, `input.ts`, `settings.ts` | Three.js, WebAudio, DOM | **KEEP**. `view` hears the sim only through `SimEvents` and state reads. |
| `src/game/` runtime | `runtime.ts` (`startGame`: loading steps, scene, composer, loop, dev globals), `match.ts` (`playMatch` + `MatchSource` + practice `createMatch`), `online.ts` (online `MatchSource`), `loading.ts` (task runner + `STARTUP`) | everything above, **plus `net/`** | **KEEP** the design; **REFACTOR** its location (the game↔net folder inversion). |
| `src/net/` | `protocol.ts` (messages, quantisation, binary snapshot, `parseClient` validation, `PROTOCOL`, `BUILD`), `connection.ts` (socket + `Link`), `client.ts` (DOM-free mirror of a server match), `prediction.ts`, `snapshots.ts`, `matchmaking.ts` (page store), `session.ts` (Nakama), `chat.ts` + `chatCommand.ts` | sim (`enlist`, `createWorld`, `MODES`, `drive.ts`), content registries | **KEEP**; the event codec is split (§9). |
| `src/screens/` | Screens. `GameCanvas.tsx` (391 lines) starts/disposes `startGame` and owns the loading overlay, pause menu, exit confirm, results, keys and chat wiring. | runtime, `Match` type, registries, `net/chat`, `net/matchmaking` | **KEEP** the structure; **REVIEW** `GameCanvas` size. |
| `src/hud/` | `Hud.tsx` (723 lines): markup rendered once (`memo`), then `update(match, camera)` writes the DOM every frame. Holds both modes' panels, the scoreboard and effect chips. | `Match`, `ffa/*`, `tdm/config`, `modes` | **REFACTOR**: per-mode panels (§7). |
| `server/` | `main.ts` (env, arenas up front), `server.ts` (HTTP health, ws door: origins, auth, rate limits, strikes, backlog; one fixed-step loop; dev latency simulator), `lobby.ts` (rooms, seating, one connection per user), `matchmaker.ts` (pure), `room.ts` (one match: seats, input queues, sim step, snapshots, records, journal, fair-play watch, chat channel names), `recorder.ts` (SimEvents → wire), `rewind.ts` (lag compensation), `fairplay.ts` (pure), `records.ts` (disk), `replay.ts`/`replay-main.ts`, `arenas.ts` + `headless.ts` + `digests.json`, `auth.ts` (HS256) | `src/game/*` sim/modes/content, `src/net/protocol.ts` | **KEEP** flat; light extractions from `room.ts` (§10). |

### 1.3 Good decisions (KEEP; do not undo)

1. **React is out of the loop.** `renderer.setAnimationLoop(frame)` drives `match.frame`, and the HUD writes the DOM through an imperative handle. React re-renders only on loading steps and player-view phase changes (`GAME_LOOP.md`, verified in `runtime.ts` and `GameCanvas.tsx`).
2. **One simulation for practice, server and replay.** `createSimulation` is called by practice (`match.ts`), by the room (`server/room.ts`) and by the replay (through `createRoom`). `server.check` asserts that "a room is the practice simulation, bit for bit".
3. **`SimEvents` as a direct-call interface, not a bus.** Headless runs pass no-ops. The server passes the recorder. The browser passes the view.
4. **Controls are plain data written by interchangeable control sources**: the pilot, a bot brain, a server input queue, a replay's `forced` input.
5. **Pure, event-queue mode rules** with 143 and 163 checks, and an adapter (`MatchMode`) around them. `share()`/`mirror()` carry rules state online without the browser ever ticking rules.
6. **Seeded randomness with separate streams**: simulation `seed ^ 0x9e3779b9`, bot guns `seed ^ 0x2545f491`, FFA rules `createRng(seed)`. Presentation (particles, pitch, shake) is deliberately unseeded.
7. **`MatchSource`**: the player's side (`playMatch`) is identical for practice and online. Only the step source differs.
8. **Protocol discipline**: quantised integers, binary snapshot frames, `PROTOCOL` bumps, a content-hash `BUILD` id, server-side `parseClient` clamping. The client sends intent only.
9. **Arena digests** held across browser, server, CI and the deploy smoke test.
10. **The `.check.ts` convention**: plain-node self-checks with no test framework, plus Vite-bundled server checks that use real sockets.
11. **Session caches with explicit ownership** (`STATE_OWNERSHIP.md`): materials, baked textures, arenas and sound buffers live for the session; geometries are disposed per match.

### 1.4 Problem decisions (REFACTOR or REVIEW)

| Problem | Evidence | Verdict |
|---|---|---|
| Weapon identity by turret model | `net/protocol.ts:175`, `ai.ts:71` (`botGun` copies the spec) | REFACTOR |
| Closed-set turret and icon selection | `vehicle/vehicle.ts:195`, `hud/Hud.tsx:235–241`, `combat.ts:9` | REFACTOR |
| Mode UI concentrated in HUD/Results; `room.ts` team-chat branch | `Hud.tsx:456–457, 676–677`; `Results.tsx:41, 106`; `room.ts:219` | REFACTOR |
| `MatchMode.show(camera)` puts rendering in the gameplay contract | `game/mode.ts`; `modes.ts` injects `createPickups(scene)`, so the **server bundle includes pickups visuals** | REFACTOR (small) |
| Wire-event codec split; positional indices | `server/recorder.ts` ↔ `net/client.ts` `play()`/`mine()` | REFACTOR |
| Duplicate domain types | `Life` ×3 (`mode.ts`, `ffa/rules.ts`, `tdm/types.ts`); `Point {x,z}` ×3 (`mode.ts`, `tdm/types.ts`, `ffa/items.ts`); two unrelated `Seat` types (`mode.ts`, `net/protocol.ts`) | REFACTOR (type-only) |
| Loadout validation duplicated | `loadout.ts savedLoadout()` vs `protocol.ts pick()` | REFACTOR (small) |
| game↔net folder inversion | `game/runtime.ts → net/connection.ts`, `game/online.ts → net/client.ts`; `net/client.ts → game/*` | REFACTOR (move) |
| Arena gameplay layout derived from the visual scene graph | `props.ts solid()` stores colliders in `userData`; `arena.ts collectColliders()` reads `matrixWorld`. The server must build the visual arena under a DOM shim, then `strip()` it. | **REVIEW → postpone** (digests guard it; §18) |
| View reads Rapier internals | `view.ts` `wheelIsInContact`, `poseWheels` (suspension from the controller) | REVIEW (documented debt; fine while clients run a local world) |
| `Garage.tsx → vehicle/drive.ts` for `drivePerformance` (pure maths inside a Rapier module) | `screens/Garage.tsx:6` | REVIEW (harmless; move if convenient) |
| Unused Vitest dev dependency | `package.json`; `AGENTS.md` says "installed, unused" | REVIEW (owner decision) |
| Legacy generators kept as reference | `VehicleGenerator.ts`, `ArenaGenerator.ts`, `proceduralTexture.ts` (no importers) | REVIEW (owner decision; `AGENTS.md` says keep) |

---

## 2. God objects

Size alone is not a defect. `city.ts` (833 lines) and `recipes.ts` (690) are big, single-purpose content files. The files below were judged on **responsibilities**.

### `game/match.ts` (318 lines): **KEEP** (one small split)

| | |
|---|---|
| **Current responsibilities** | `playMatch`: the player's side of any match (pilot wiring, the fixed-step accumulator, interpolation, player-view phase machine, pause/restart/lose, HUD-facing state object, debug lines). `createMatch`: the practice `MatchSource` (seeds, roster, world, view, mode, sim, feed). Types: `MatchSource`, `MatchParts`, `Match`. |
| **Should remain** | `playMatch`, `MatchSource`, `Match`. |
| **Should move** | `createMatch` → `runtime/practice.ts`, symmetric with `online.ts`. |
| **Destination** | `runtime/match.ts` + `runtime/practice.ts` |
| **Reason** | Clarity: the two sources would sit side by side. No behavioural reason. |
| **Risk** | Low. Pure move; the seed pick (`freshSeed`, `Math.random`) moves with it. |

### `game/simulation.ts` (395 lines): **KEEP**

| | |
|---|---|
| **Current responsibilities** | Combatants (`enlist`, `reset`), one fixed step (think → drive → fire → `world.step` → read poses → crash → stuck → rockets → rules tick → respawn), hitscan (with the server's optional `castRound` hook), rockets and blasts, damage, wrecks, respawn via rules, stuck recovery, `standDown`, `restart`. |
| **Should remain** | All of it. It is the authoritative core, and its step order *is* the game's determinism. |
| **Should move** | Nothing now. **Only if** a third firing behaviour (beam, homing, mine) is scheduled: turn `if (c.weapon.spec.rocket) return launch(c)` into a small table keyed by `spec.kind`. |
| **Destination** | `sim/simulation.ts` (folder move only) |
| **Reason** | Cohesive; well-checked. |
| **Risk** | High if touched: step order, RNG draw order, Rapier call order (§18). |

### `game/runtime.ts` (160 lines): **KEEP** (move folder)

| | |
|---|---|
| **Current responsibilities** | Match loading tasks, renderer settings, scene/sky/sun, arena borrow, practice-or-online match creation, composer + live settings, frame, resize, release, dev globals, online arena/digest validation. |
| **Should remain** | All of it. This is the application-level composition. |
| **Should move** | The folder: `game/` → `runtime/`, removing the `game → net` inversion. The session arena cache `loadArena` could move in from `maps.ts`. |
| **Risk** | Low. |

### `game/ai.ts` (556 lines): **REFACTOR** (split by responsibility; Phase 6)

| | |
|---|---|
| **Current responsibilities** | (1) Tuning (`AI`, `TACTICS`). (2) Difficulty (`Skill`, `DIFFICULTIES`, `armBot`, `botGun`). (3) The agent model (`Agent`, `Brain`, `Plan`, `createBrain`, `provoke`). (4) Perception (`feel`, `canSee`, `open`, `hiddenFrom`, `pickHideout`, `nodesInView`). (5) Navigation (`routesTo`, `travel`). (6) Targeting (`pickTarget`, `leadTarget`). (7) The decision loop (`think`). |
| **Should remain together** | `think` + targeting (they share per-call scratch state). |
| **Should move** | `sim/ai/skill.ts` (Skill, DIFFICULTIES, armBot, botGun); `sim/ai/brain.ts` (Agent, Brain, Plan, createBrain, provoke); `sim/ai/perception.ts`; `sim/ai/navigation.ts`; `sim/ai/think.ts` (+ targeting). |
| **Reason** | Five responsibilities in one file. AI tuning per weapon and new behaviours will land here; smaller files reduce merge conflicts for parallel agents. |
| **Risk** | **Medium-high.** (a) The **order of `random()` draws** must not change. (b) Module-level scratch vectors (`goal`, `circling`, `weaving`, `lead`, `aimAt`, `from`, `along`, `ray`, `spot`, `errandAt`, …) must not become *shared* between functions that are live at the same time after the split. `bots.check` (behaviour ranges) plus the golden fingerprints (Phase 0) catch both. |

### `game/view.ts` (291 lines): **KEEP**

Cohesive. It builds models, implements `SimEvents` (effects, sound, shake, feedback pulses), handles interpolation, turret laying, ambient effects, engine audio, `refit`, restart and dispose. Only change: the fire sound cue is chosen by `c.weapon.spec.rocket ? 'launch' : 'shot'`. That becomes `spec.cue ?? …` when a weapon needs its own sound (§6).

### `server/server.ts` (310 lines): **KEEP** (optional extraction)

| | |
|---|---|
| **Current responsibilities** | HTTP `/health`; the upgrade door (path, origin, per-address cap); per-socket protocol (hello deadline, version/build check, auth, rate tokens, strikes, backlog cut-off); routing to lobby or room; **dev latency simulator** (`held()`: lag, jitter, stalls, order-preserving); the fixed-step loop with catch-up cap; shutdown. |
| **Should move (optional)** | `held()` + `stallEnd` → `server/netsim.ts` (dev/check-only concern). |
| **Reason** | Separates a development tool from the production door. Low value; do it only when touching the file anyway. |
| **Risk** | Low; `netplay.check` exercises it. |

### `server/room.ts` (567 lines): **REFACTOR, light** (Phase 6)

| | |
|---|---|
| **Current responsibilities** | Seats and humans (`join`, `leave`, `freeSeat`, `takeWheel`); **input queue policy** (queue, drain window, stale/idle, repeats/drops/depth stats); applying `Given` inputs; the **replay journal format** (`ReplayLine`, `Given`, `note()` row encoding) that `replay.ts` must mirror; the **fair-play watch** geometry (`watch`, eye/camera points); the first-match hold; the step orchestration; the **match record** (`MatchRecord`, `SeatRecord`, `matchRecord()`); snapshot/state broadcast; chat channel names; restart. |
| **Should remain** | `step()` orchestration **in its exact order**; seats; broadcast; lifecycle. |
| **Should move** | `server/journal.ts`: `ReplayLine`, `Given`, `DRIVING/STANDING/COASTING`, row encode/decode, shared by `room.ts` and `replay.ts` (today the format is implicit in both). `server/inputs.ts`: the per-person queue/drain/stale policy (pure, unit-checkable). `server/matchRecord.ts`: record types + builder (optional). |
| **Reason** | The journal format is a persisted, versioned artefact that lives inside a 567-line file. The input policy is the most tuned code on the server (see `NET_LOG.md`) and deserves its own check. |
| **Risk** | **Medium-high:** replay determinism and the journal's line semantics (`ahead()` in `replay.ts`). Guarded by `server.check`'s replay test and the golden fingerprints. |

### `server/matchmaker.ts` (278 lines): **KEEP**

Pure, clock-injected, with 63 checks. Cohesive. No refactor needed.

### `server/lobby.ts` (224 lines): **KEEP**

Room lifecycle and seating policy. Cohesive.

### `hud/Hud.tsx` (723 lines): **REFACTOR** (Phase 4, then Phase 6)

| | |
|---|---|
| **Current responsibilities** | All HUD markup; element collection by `data-hud`; the per-frame update for compass, minimap, **FFA score/board/lead line**, **TDM team score/momentum**, clock/overtime (branching per mode), banner/countdown, feed, speed/health, **effect chips (FFA effects)**, weapon panel (**hard-coded icons**), crosshair, markers, hints, scoreboard (**mode-specific ordering, headers, labels**), debug. |
| **Should remain** | The shared HUD (compass, clock, banner, feed, vitals, weapon, crosshair, markers, debug) and the imperative-write pattern. |
| **Should move** | Mode-specific panels → `modes/<mode>/hud.tsx` via a client-only `MODE_VIEWS` registry (§7). Weapon icon → weapon spec data (§6). Optionally split the shared parts into `hud/*.ts` helpers (Phase 6). |
| **Reason** | This is the file a third mode must edit today. |
| **Risk** | Medium: per-frame allocations, `data-hud` collection scope, `memo`. Visual parity is checked by hand in the browser (no screenshot tests exist). |

### `screens/GameCanvas.tsx` (391 lines): **REVIEW**

Loading overlay, failure/retry, pause menu, exit confirm, results, keyboard handling, chat wiring. Splitting into `PauseMenu`, `ExitConfirm` and `LoadingOverlay` components would help readability. **Do not** touch the `startGame`/dispose effect or the link-closing semantics without the StrictMode double-mount scenarios in mind (§18).

### `net/client.ts` (420 lines): **REFACTOR, light** (Phase 3)

Cohesive (the mirror). Only the wire-event **decode** (`play`, `mine`, positional `f[n]`) moves into a shared codec. Reconcile/place/step stay.

### `net/protocol.ts` (477 lines): **REVIEW**

A cohesive "wire" module. Optional: move the binary snapshot frame (`packCars`, `packSnapshot`, `unpackSnapshot`) to `net/snapshotFrame.ts`. **Do not move the file itself.** `scripts/match-smoke.mjs` reads `PROTOCOL` from the path `game/src/net/protocol.ts` with a regex.

### `ffa/rules.ts` (536) and `tdm/rules.ts` (335): **KEEP**

Pure, event-sourced, heavily checked. Only change: import the shared `Life`/`Point` types instead of redeclaring them.

---

## 3. Dependency graph

### 3.1 Folder-level graph (current, value + type imports)

```
                 main.tsx ──(dynamic import)──► App.tsx
                                                  │
              ┌───────────────────────────────────┼─────────────────────────────┐
              ▼                                   ▼                             ▼
          screens/  ──────────────────────►  game/ (runtime.ts, match.ts, …)  ◄──── hud/
              │                                   │  ▲                          │
              │                                   ▼  │ (net/client → game/*)    │
              └───────────► net/ ◄────────────────┘  │                          │
                             │  (runtime.ts → net/connection; online.ts → net/client)
                             └───────────────────────┘
server/ ──► game/ (sim, modes, maps→arena builders→materials→renderer), net/protocol.ts
```

### 3.2 Edges that matter

| Edge | Kind | Verdict |
|---|---|---|
| `game/runtime.ts → net/connection.ts` (value: `NetError`, `REASONS`) | runtime → net | Fine for a **runtime** module, but it sits in `game/`, which `net/` also imports → **folder inversion**. Fix by moving the runtime (Phase 5). |
| `game/online.ts → net/client.ts` | runtime → net | Same. |
| `net/client.ts → game/{modes,simulation,physics,combat,vehicles,drive}` | net → sim | **KEEP** (correct direction). |
| `net/protocol.ts → game/{combat (WEAPONS), scoring (createStats), vehicles (VEHICLES), simulation (type Combatant)}` | net → sim/content | **KEEP**; the wire depends on the entity shape by design. |
| `server/* → src/game/*`, `src/net/protocol.ts` | server → shared | **KEEP**. |
| `server/lobby.ts, arenas.ts, main.ts → game/maps.ts → arena builders → materials/library.ts → renderer.ts` | server → visual code | Works because `library.ts` skips the GPU bake when `typeof window === 'undefined'` and `headless.ts` fakes `document`. **REVIEW**: it explains the 4.4 MB server chunk. Not a correctness bug. |
| `game/modes.ts → ffa/pickups.ts` (Three.js scenery) | registry → visuals | **REFACTOR**: the scenery factory belongs to the client (Phase 4). |
| `hud/Hud.tsx → ffa/config, ffa/items, ffa/rules (type), tdm/config` | UI → mode internals | By design today (narrowing on `mode.kind`); **REFACTOR** into per-mode panels (Phase 4). |
| `hud/minimap.ts → ffa/items (ITEMS colours)` | UI → mode data | Pass marks with colours from the FFA panel (Phase 4). |
| `screens/Results.tsx → tdm/config (TEAMS)` | UI → mode config | Per-mode results panel (Phase 4). |
| `screens/Garage.tsx → vehicle/drive.ts` (`drivePerformance`) | UI → physics module | REVIEW: harmless (pure function). Optionally move it next to the vehicle specs. |
| `arena/props.ts → vehicle/parts.ts` | content → content | Content cross-reference (tyres, lamps). Move `parts.ts` to a shared content location (Phase 5). |
| `materials/library.ts → renderer.ts` | content → GPU context | Real, lazy dependency (baking). Resolve by folder placement (§4: `render/`). |
| `feed.ts → view.ts` (type `Feedback`) | presentation ↔ presentation | KEEP. |

### 3.3 Unwanted-dependency audit (what the brief asked for)

| Unwanted dependency | Present? | Evidence / note |
|---|---|---|
| domain → React | **No** | No `react` import under sim/modes files. |
| domain → Three.js | **Math only**, plus 3 type-only rendering leaks | `simulation.ts`, `ai.ts`, `combat.ts` use `THREE.Vector3/Quaternion/MathUtils` as maths. **Rendering types** leak into the contract: `mode.ts` (`THREE.Camera` in `show()`), `modes.ts` (`THREE.Scene` in `ModeContext`), `ffa/mode.ts` (`THREE.Camera`). **Verdict:** keep Three.js maths (§19); remove the rendering types (Phase 4). |
| domain → DOM | **No** | `window`/`document`/`localStorage` appear only in `settings.ts`, `loadout.ts` (in functions, try/catch), `input.ts`, `audio.ts` (**import-time**), `loading.ts`, `canvasTextures.ts`, `ground.ts` (canvas, shimmed on the server), `runtime.ts`, `turntable.ts`. |
| domain → WebSocket | **No** | Only `net/connection.ts`, `net/matchmaking.ts` and `server/server.ts` touch sockets. |
| domain → browser APIs (wall clock, `Math.random`) | **No** | `Math.random` only in `match.ts` (seed pick), `camera.ts`, `audio.ts`, `effects.ts` (presentation). `performance.now` only in runtime/net/server. |
| server → client-only code | **No production edge** | `server/browser.ts` (a check helper) imports `net/client` + `net/connection`: fine for checks. Import-time hazard: nothing guards against a future import of `audio.ts`. |
| presentation → simulation internals | **By design, read-only** | HUD reads `match.mode.rules` query methods (`standings()`, `deficit()`, `soleLeader()`…). `view.ts` reads the Rapier vehicle controller (REVIEW, §8). |
| simulation → rendering | **No** | `SimEvents` only. |
| gameplay → networking | **No** | Rules/sim never import `net/`. The adapters' `share`/`mirror` are plain data. |

The full edge list is in [Appendix A](#appendix-a-import-graph-current).

---

## 4. Architectural layers

### 4.1 What the constraints actually are

Before proposing layers, here are the **real** constraints, derived from the code rather than from a template:

1. **Server-reachable code must be import-safe and call-safe in Node.** No React, no WebAudio, no `WebGLRenderer` calls, no import-time `window`/`document` use. The arena builders are the documented exception (DOM shim + headless material path).
2. **Authoritative gameplay must not depend on presentation**, and must not read the wall clock or unseeded randomness.
3. **React stays above the runtime.** It reads the `Match` object and never drives steps.
4. **`net/` (client mirror) and `server/` share only `net/protocol.ts` (+ the future `net/events.ts`) and gameplay.** Never each other's transport.

### 4.2 Recommended layers: 7 new folders beside the existing `net/`, `screens/` and `hud/` (not 9 layers + subfolders)

| Layer (folder) | Purpose | Allowed imports | Forbidden imports | Current files |
|---|---|---|---|---|
| **`shared/`** | Leaf utilities used by *both* gameplay and content/view | nothing internal | anything internal | `rng.ts`; shared `Point`/`Life` types; `clock()` formatter |
| **`content/`** | What the world *is*: vehicle/weapon **specs** (data), 3D **models/turrets**, **arenas** (builders + layout), shared parts | `shared/`, `render/` (materials, geometry), `three` | `sim/`, `modes/`, `view/`, `runtime/`, `net/`, React, audio | `arena/*`, `vehicle/{vehicles,vehicle,parts}.ts`, weapon specs from `combat.ts`, `maps.ts` registry, legacy generators |
| **`render/`** | Shared Three.js infrastructure: the one WebGL context, the material library + GPU bake, geometry helpers, sky/sun, post-processing | `shared/`, `three` | `sim/`, `modes/`, `view/`, `runtime/`, `net/`, React | `renderer.ts`, `materials/*`, `geometry.ts`, `environment.ts`, `postprocessing.ts` |
| **`sim/`** | Authoritative, headless gameplay: simulation, combat mechanics, physics, driving, AI, scoring, the mode **contract**, loadout type | `shared/`, `content/` **specs, types and registries only**, `three` (**maths only**), Rapier | `render/`, `view/`, `runtime/`, `net/`, React, DOM globals, `Math.random`, `Date.now`, `performance.now` | `simulation.ts`, `combat.ts` (mechanics), `physics.ts`, `vehicle/drive.ts`, `ai.ts`, `scoring.ts`, `mode.ts`, `loadout.ts` (type part) |
| **`modes/`** | Each game mode as a feature folder (rules, adapter, tactics, config, check) + registry + roster; **client-only** files inside each mode folder, reachable only through two client registries: `modes/scenery.ts` (`MODE_SCENERY`, Three.js, view level) and `modes/views.ts` (`MODE_VIEWS`, React panels, UI level) | `sim/`, `content/`, `shared/`. Scenery files may also import `render/`; panel files may also import React and the `Match` type | domain files: same as `sim/`. `modes/scenery.ts` is imported only by `runtime/`; `modes/views.ts` only by `hud/` and `screens/` | `ffa/*`, `tdm/*`, `modes.ts`, `roster.ts` |
| **`view/`** | The match as the local player sees, hears and controls it: view, effects, camera, audio, sounds, feed, pilot, input, settings, garage turntable | `render/`, `content/`, `sim/` (read), `modes/` (types), `shared/`, DOM, WebAudio | `runtime/`, `net/`, `screens/`, `hud/`, React | `view.ts`, `effects.ts`, `camera.ts`, `audio.ts`, `sounds.ts`, `feed.ts`, `pilot.ts`, `input.ts`, `settings.ts`, `turntable.ts` |
| **`runtime/`** | Browser composition (the "application" layer): `startGame`, `playMatch`, the practice and online sources, loading, local persistence | everything below + `net/` | `screens/`, `hud/` (React) | `runtime.ts`, `match.ts`, `online.ts`, `loading.ts`, loadout storage |
| **`net/`** (unchanged) | Wire protocol, codec, socket, client mirror, prediction, interpolation, matchmaking store, Nakama session, chat | `sim/`, `modes/` (domain), `content/` specs, `shared/` | `view/`, `runtime/`, `render/`, `screens/`, `hud/`. `protocol.ts` + `events.ts` must also stay **server-safe** | as today |
| **`screens/`, `hud/`** (unchanged) | React UI | `runtime/` (Match API), registries, `modes/views.ts`, `net/matchmaking`, `net/chat`, `view/settings`, `view/turntable` | Rapier, `sim/physics`, `sim/simulation` values, `render/` (except via the turntable) | as today |
| **`server/`** (unchanged, flat) | Authoritative server | `sim/`, `modes/` (domain), `content/`, `net/protocol.ts`, `net/events.ts`, `shared/` | `view/`, `runtime/`, `screens/`, `hud/`, `render/` **except** through `content/arenas` builders, the client parts of `net/`, React, Nakama JS | as today |

**Why no separate `application/`, `engine/`, `platform/` or `presentation/`:**

- **`application/`**: `runtime/` *is* the application layer. The name `runtime` matches existing vocabulary (`MODULE_BOUNDARIES.md` calls it "Browser runtime"). `session/` would clash with `net/session.ts`.
- **`engine/`**: physics, driving and the simulation cannot be separated from gameplay without a physics abstraction (rejected, §19). Hitscan, AI perception and item placement *are* Rapier queries.
- **`platform/`**: the platform adapters are `input.ts`, `audio.ts`, `renderer.ts` and two small `localStorage` users. Each has one consumer layer. A separate folder would hold 3–4 files and add a rule without preventing any real mistake.
- **`presentation/`**: `screens/` and `hud/` are already clean folders. Moving them gains nothing.

`render/` exists (instead of folding it into `view/`) for one concrete reason: `materials/library.ts` is **server-reachable** through the arena builders, and it imports `renderer.ts`. If both lived in `view/`, the rule "server must not reach `view/`" would need an exception. [Medium confidence: the alternative is `view/` + one documented exception. Both work; `render/` states the truth about reachability.]

---

## 5. Contracts and interfaces

Each candidate is judged on whether it solves a problem **in this codebase**.

| Contract | Exists today? | Why it exists / would exist | Consumers | Implementers | Verdict |
|---|---|---|---|---|---|
| **`MatchMode`** | Yes (`game/mode.ts`) | The engine never asks which mode runs | `simulation.ts`, `match.ts`, `room.ts`, `net/client.ts`, HUD (via `kind`) | `ffa/mode.ts`, `tdm/mode.ts` | **KEEP**; **remove `show(camera)`** (client scenery, Phase 4); keep `share`/`mirror`. |
| **`ModeRules`** | Yes | What the simulation drives each step | `simulation.ts`, `feed.beep`, HUD | FFA/TDM rules | **KEEP**. |
| Mode registry entry | Yes (`MODES`) | Cards, line-up, factory | MapSelect, roster, room, lobby, client | – | **KEEP**; add **`teamPlay: boolean`** (for `room.ts` team chat; feed colours are already passed as a flag); move `createPickups` out. |
| **`ModeViews`** (new, client-only) | No | Removes the `mode.kind` branches from `Hud.tsx`/`Results.tsx` | `hud/Hud.tsx`, `screens/Results.tsx` | `modes/ffa/{hud,results}`, `modes/tdm/{hud,results}` | **ADD (Phase 4)**. Solves a concrete problem: a third mode would otherwise edit 2 large shared files. |
| **`ModeScenery`** (new, client-only, optional per mode) | No (`MatchMode.show` + a scenery injected through `modes.ts`) | Takes rendering out of the gameplay contract and out of the server bundle | `runtime/runtime.ts` (per frame, restart, dispose) | `modes/ffa/scenery.ts` | **ADD (Phase 4)**; a `Partial<Record<Mode, …>>` registry, because most modes have no scenery. |
| **`VehicleSpec`** (definition) | Yes (`vehicle/vehicles.ts`) | Data for physics, garage, rewind | sim, garage, rewind, view, protocol | `ROSTER` entries | **KEEP**; move the `Handling`/`Chassis` types next to it (content), out of `drive.ts`. |
| **`WeaponSpec`** (definition) | Yes (`combat.ts`) | Data for sim, garage, HUD, bots | many | `ROSTER` entries | **KEEP**, and **add `id`** (identity), **`turret`** (open key replacing the `model` union), **`icon`** (SVG path data), optional **`cue`** (fire sound) and optional **`ai`** style (Phase 2). |
| **Turret builder registry** (new) | No (a ternary) | A new weapon without editing `vehicle.ts` | vehicle model builders, turntable | `content/weapons/<id>/turret.ts` | **ADD (Phase 2)**; a plain `Record<string, () => THREE.Group>`. |
| **`Arena`** (built) + **`MapInfo`** (definition) | Yes | The map contract | sim, modes, physics, runtime, server | builders | **KEEP**. |
| **`SimEvents`** | Yes | The sim reports; presenters present | sim | view, recorder, client playback, checks (no-ops) | **KEEP**. The wire codec (below) mirrors it. |
| **Wire event codec** (new) | No (split) | One definition of each event's fields for encoder *and* decoder | `server/recorder.ts`, `net/client.ts` | `net/events.ts` | **ADD (Phase 3)**. |
| **`Feed`** | Yes | Mode adapters announce to the player | adapters | `feed.ts` | **KEEP**. |
| **`MatchSource`** | Yes | Practice vs online step source | `playMatch` | `createMatch`, `createOnlineMatch` | **KEEP**. This is the "local and online share the core" seam. |
| **`MatchTransport`** | Effectively yes: `Link` (`take`, `send`, `status`, `close`, `onPong`) | Socket abstraction | `net/client.ts`, `online.ts` | `connection.ts link()`; checks use it in Node | **KEEP; no new interface.** |
| **`RandomSource`** | Yes: `() => number` + `createRng` | Seeded streams | sim, ai, rules | mulberry32 | **KEEP.** A function type is the right size; an interface/class adds nothing. |
| **`Clock`** | Partly: sim time is the `dt` argument; `matchmaker` takes a `now()` hook | – | – | – | **REJECT** a general `Clock` interface. Gameplay already has no wall clock; the matchmaker's hook covers its case. |
| **`PhysicsWorld` / `PhysicsBody`** | No | Hypothetical engine swap | – | – | **REJECT** (§19). One engine; determinism depends on the exact Rapier call sequence; prediction replays `drive.ts` on the same Rapier. |
| **`Audio`** | No (module functions) | – | view, feed | `audio.ts` | **REJECT.** `SimEvents` already isolates the sim from audio; headless runs never import audio. |
| **`AssetLoader`** | No (`public/models/` does not exist yet) | glTF loading + caching + disposal | – | – | **POSTPONE** until the first glTF asset. When it lands, it needs its **own disposal rule**: glTF materials are per-asset, unlike `materials/library.ts`, which must never be disposed (§18). |
| **`Loadout`** | Yes (ids) | Garage → match → wire | App, protocol, room, lobby | – | **KEEP**; add a pure `parseLoadout(unknown)` used by both `localStorage` restore and `parseClient` (dedupes validation). |

---

## 6. Plug-and-play content

For each content type: **today** (what a developer must touch now), then **target** (after the phases).

### 6.1 Vehicle

**Today:**
1. `vehicle/vehicles.ts`: `ROSTER` entry (handling, chassis, turret mount, armour, garage copy).
2. A model builder + `MODELS` entry in `vehicle/vehicle.ts`. That file's layout constants read `VEHICLES.razor` at module scope, so a second vehicle needs its own builder file anyway.
3. **Server:** `room.join` ignores the person's vehicle. `server.check.ts:365` fails on purpose once `VEHICLES` has two entries. The `ro` message and the replay `join` line carry no vehicle.
4. **Bots:** always `BOT_VEHICLE` (`roster.ts`).
5. **Garage:** no vehicle pager (`Garage.tsx` shows `loadout.vehicle` only).
6. `server/arena.check.ts` checks spawn clearance against `VEHICLES.razor` shells only.

**Target (after Phases 5 and 7):**

```
Developer creates:
    content/vehicles/<id>/spec.ts      VehicleSpec (name, kind, blurb, armour, handling, chassis, turret mount)
    content/vehicles/<id>/model.ts     builder → THREE.Group with nodes body, turret > gun, wheel_fl/fr/rl/rr
Registers:
    content/vehicles/index.ts          VEHICLES: one line
    content/vehicles/models.ts         MODELS: one line    (Record<VehicleId, …>: forgetting it is a compile error)
Core engine changes:
    none, once Phase 7 (a one-time capability) has landed: seats honour loadout.vehicle, bots may draw vehicles,
    garage pager, arena.check over every vehicle's shells
Checks that pick it up automatically:
    simulation.check "content" (positive numbers, 4 wheels, it drives), arena.check spawn clearance (after Phase 7)
```

A new **mechanic** (tracks, jumping) is not content. It is a capability in `sim/drive.ts`/`sim/simulation.ts`, as `ARCHITECTURE.md` already states. [High confidence]

### 6.2 Weapon

**Today:**
1. `combat.ts`: `ROSTER` entry, **plus** widen the `model: 'minigun' | 'rocketPod'` union.
2. `vehicle/parts.ts`: a turret builder, **plus** edit the ternary in `vehicle/vehicle.ts:195`.
3. `hud/Hud.tsx`: another hard-coded SVG and `data-kind` rule.
4. Sound: `view.ts` picks `'launch'`/`'shot'` by the `rocket` flag. Bot fighting style: `ai.ts TACTICS` picks `gun`/`rocket` by the same flag.
5. **Bug risk:** if the new weapon reuses an existing turret model, `weaponId()` misreports it on the wire, in records and in replays.

**Target (after Phase 2):**

```
Developer creates:
    content/weapons/<id>/spec.ts       WeaponSpec { id, name, blurb, damage, fireRate, magazine, reloadTime, range,
                                                    spread, rocket?, turret: '<turretKey>', icon: '<svg path>',
                                                    cue?: SoundCue, ai?: { engage, circle, keep, fire } }
    content/weapons/<id>/turret.ts     only if it needs a new turret model
Registers:
    content/weapons/index.ts           WEAPONS: one line
    content/weapons/turrets.ts         TURRETS: one line (only for a new turret model)
Core engine changes:
    none for a new hitscan gun or a new rocket weapon (numbers, turret, icon, sound, bot style)
    a NEW FIRING BEHAVIOUR (beam, homing, mine) is a sim capability: a branch in simulation.fire,
    new SimEvents/wire events (codec, Phase 3), playback, view effects. That is accepted core work, not content.
Automatic:
    garage lists it; bots may draw it (armBot uses the roster); HUD shows name/ammo/icon; the wire, records and
    replays carry spec.id; rewind works for hitscan (it reads chassis shells, not weapons)
```

### 6.3 Arena (map)

**Today:** already close to plug-and-play.
1. `arena/<name>.ts`: a builder returning `Arena` (spawns, bases for TDM, zones for FFA, colliders via `solid()`, nav graph, emitters, minimap floor, `update`).
2. `maps.ts`: a `MAPS` entry (name, preview, arrival line, `modes` hosted, `build`).
3. `public/maps/<id>.jpg` preview.
4. **Online:** it must build headless (no import-time `window`) and give the same digest in Node and in the browser. Add its digest to `server/digests.json`; `arena.check` enforces it.

**Target:** the same steps, in a feature folder:

```
Developer creates:
    content/arenas/<id>/<id>.ts        builder → Arena (uses content/arenas/kit/*)
    public/maps/<id>.jpg
Registers:
    content/arenas/index.ts            MAPS: one line (lists the modes it hosts)
    server/digests.json                one line (arena.check prints the digest)
Core engine changes:
    none
New check (Phase 1/4):
    for every (map, hosted mode) pair: MODES[mode].lineUp(arena) and the adapter build headless
    (today tdm/mode.ts throws at runtime for an arena without bases)
```

### 6.4 Game mode

**Today:** a folder (`config`, pure `rules`, `mode.ts` adapter, check) + a `MODES` entry are clean, **but** you must also:
- edit `hud/Hud.tsx` (score panel, clock/overtime, effect chips, scoreboard ordering/headers),
- edit `screens/Results.tsx` (verdict, badge, MVP),
- edit `server/room.ts:219` if the mode has teams (team chat),
- add the mode to each map's `modes` list in `maps.ts` (explicit by design).

**Target (after Phase 4):**

```
Developer creates:
    modes/<id>/config.ts               tuning (durations, respawn, scoring numbers for scoring.ts)
    modes/<id>/rules.ts                pure ModeRules implementation + its own state and events
    modes/<id>/mode.ts                 adapter → MatchMode (lineUp, starts, plan, outcome, report→Feed, share/mirror)
    modes/<id>/<id>.check.ts           plain-node rules check
    modes/<id>/hud.tsx                 ModeViews.hud panel (client only)
    modes/<id>/results.tsx             ModeViews.results panel (client only)
    modes/<id>/scenery.ts              optional (client only)
Registers:
    modes/index.ts                     MODES: one line (label, tags, blurb, teamPlay, lineUp, create)
    modes/views.ts                     MODE_VIEWS: one line (Record<Mode, …>: missing = compile error)
    modes/scenery.ts                   MODE_SCENERY: one line, only if the mode draws scenery
    content/arenas/index.ts            add the id to the `modes` of every map that hosts it
Core engine changes:
    none (simulation, playMatch, room, lobby, net/client, Hud.tsx and Results.tsx untouched)
```

"Custom mode" needs a **product decision** first (see §7.5).

### 6.5 Pickup (FFA item)

**Today** (all inside `ffa/`, except the HUD):
- `items.ts`: `ItemType` union + `ITEMS` entry.
- `config.ts`: weights and durations.
- `rules.ts`: `apply()` branch (or an `Effect` key).
- `pickups.ts`: `tokenGeometry` switch (exhaustive on `ItemType`: a compile error if missed; good).
- `mode.ts`: `share()`/`mirror()` list the effect keys **explicitly**.
- `Hud.tsx`: the `EFFECTS` chip list.

**Target (Phase 4):** all of the above stays FFA-scoped. Two things change:
- `share()`/`mirror()` loop over the `Effect` keys.
- The HUD chips derive from `ITEMS` (with a `timed` flag) inside the FFA HUD panel.

```
Developer creates/edits (modes/ffa/ only):  items.ts entry, config numbers, rules.apply effect, scenery token
Core engine changes: none
```

**REVIEW:** do **not** promote pickups to a cross-mode system until a second mode wants them (YAGNI).

### 6.6 Effect (visual)

**Today:** a method on the pooled particle system (`effects.ts`) + a call from `view.ts` on a `SimEvents` callback. **KEEP.** An effect registry would be over-engineering at this size. If `effects.ts` grows past a handful of new emitters, split it into `view/effects/pool.ts` + emitter files. No new contract.

### 6.7 Audio

**Today:** a synth function in `sounds.ts` + a `CUES`/`LOOPS` entry in `audio.ts` (a registry keyed by the `SoundCue` union) + the call site (view or feed). **KEEP.** The only addition is the optional weapon `cue` (§6.2), so a weapon's fire sound is data.

### 6.8 AI behaviour

- **Per mode:** the `Plan` hooks (`value`, `errand`) supplied by the adapter (`tdm/tactics.ts`, FFA's `targetValue`/`errand`). **KEEP.** This is already the plug-in point.
- **Per difficulty:** the `DIFFICULTIES` registry. **KEEP.**
- **Per weapon:** today `TACTICS` is picked by the `rocket` flag. **REFACTOR (Phase 2):** optional `spec.ai` style with the existing two as defaults.
- **New bot behaviours** (e.g., item hunting in TDM) go through `Plan` first. Change `think()` only if a behaviour cannot be expressed as a target value or an errand.

---

## 7. Game mode architecture

### 7.1 Current (already largely correct)

```
            playMatch (match.ts)              room.ts (server)            net/client.ts (online mirror)
                    │                               │                               │
                    ▼                               ▼                               ▼
            createSimulation(sim) ─────────── MatchMode (contract: game/mode.ts) ◄──┘ (share/mirror, report)
                                                    │
                                   ┌────────────────┴────────────────┐
                                   ▼                                 ▼
                            ffa/mode.ts (adapter)             tdm/mode.ts (adapter)
                                   │                                 │
                            ffa/rules.ts (pure)               tdm/rules.ts (pure) + tactics.ts
                                   └──────────── scoring.ts ─────────┘
```

The engine (simulation, `playMatch`, room, client) reaches a mode **only through `MatchMode`**. That is the brief's "Match Engine → MatchMode → FFA/TDM", and it exists. [High confidence: verified by grep; the only `kind` checks outside the mode folders are UI and `room.ts:219`.]

### 7.2 What leaks, and the fix

| Leak | Where | Fix | Phase |
|---|---|---|---|
| Mode panels in shared UI | `Hud.tsx` (42 lines), `Results.tsx` (26 lines), `minimap.ts` (FFA marks) | Client-only `MODE_VIEWS: Record<Mode, ModeViews>`: `hud` (markup rendered once + `update(match)` writing its own `data-hud` slots), `results` (verdict/badge/extra panels) | 4 |
| Rendering in the gameplay contract | `MatchMode.show(camera)`, `ModeContext.scene`, `modes.ts → createPickups` | Remove `show`. `runtime/` builds `MODE_SCENERY[kind]?.(scene)` (a separate, view-level registry, so `runtime/` never imports React panels) and calls `update(mode, camera)` each frame, `clear()` on restart, `dispose()` with the match. The FFA adapter no longer receives `scenery`. | 4 |
| Team-play knowledge on the server | `room.ts:219` (`kind !== 'tdm'`) | `MODES[kind].teamPlay` | 4 |
| Arena compatibility known only at runtime | `tdm/mode.ts teamBases()` throws | Keep `MAPS[id].modes` explicit. Add a check that every listed (map, mode) pair lines up headless. | 1/4 |
| Duplicated types | `Life` ×3, `Point` ×3, `FfaPhase`/`TdmPhase` vs `ModePhase` | `shared/types.ts` (`Point`, `Life`); phases stay mode-specific but must be assignable to `ModePhase` (already enforced by `satisfies MatchMode`) | 2 |

**`ModeViews` sketch** (client-only; the size is deliberate):

```ts
// modes/views.ts: client-only (UI level); Record<Mode, …> makes a missing mode a compile error.
export interface ModeViews {
  hud: { Panel: () => ReactNode; bind(root: HTMLElement): (match: Match) => void } // markup once; per-frame writer
  results: (props: { match: Match }) => ReactNode                                  // verdict, badge, extra panels
}
export const MODE_VIEWS: Record<Mode, ModeViews> = { tdm: TDM_VIEWS, ffa: FFA_VIEWS }

// modes/scenery.ts: client-only (view level, Three.js); only modes that draw something have an entry.
export interface ModeScenery { update(mode: MatchMode, camera: THREE.Camera): void; clear(): void; dispose(): void }
export const MODE_SCENERY: Partial<Record<Mode, (scene: THREE.Scene) => ModeScenery>> = { ffa: createFfaScenery }
```

Each panel narrows `match.mode` to its own adapter type internally, so no `if (kind === …)` remains in shared files.

**Why not a HUD view model?** The previous refactor rejected one because it had a single consumer. Per-mode *panels* keep the imperative, allocation-free write pattern and put the branching where the knowledge is. [Medium confidence: it is a judgement call; a view-model is viable if the owner prefers data over components.]

### 7.3 Where each concern lives (target)

| Concern | Location | Server-reachable? |
|---|---|---|
| Rules (clock, phases, lives, respawn waits, protection, scoring decisions) | `modes/<m>/rules.ts` (pure) | yes |
| Scoring (statistics, assists, streaks, combat score) | `sim/scoring.ts` (shared), numbers from `modes/<m>/config.ts` | yes |
| Initial seating (`lineUp`) | `modes/<m>/mode.ts` | yes |
| Respawn choice (`pickSpawn`) | `modes/<m>/rules.ts`; `starts` from the adapter; executed by `sim/simulation.ts` | yes |
| Bot behaviour | generic `sim/ai/*` + the mode's `Plan` (`modes/tdm/tactics.ts`, FFA `targetValue`/`errand`) | yes |
| Result (`outcome`) | adapter | yes |
| Online state (`share`/`mirror`) | adapter | yes |
| Feed announcements (text) | adapter's `report(feed)` | yes (no-op on the server) |
| HUD | `modes/<m>/hud.tsx` via `MODE_VIEWS` | **no** |
| Results screen panels | `modes/<m>/results.tsx` via `MODE_VIEWS` | **no** |
| Scenery (pickups, zone) | `modes/<m>/scenery.ts` via `MODE_SCENERY` | **no** |
| Configuration | `modes/<m>/config.ts` | yes |
| Arena-screen card (label, tags, blurb) | `modes/index.ts` (plain strings) | yes (harmless) |

### 7.4 Timing and phases

`ModeTiming` (preMatch, finalMinute, finalPush, finalCountdown) and `ModePhase` are generic enough for the HUD clock and the countdown beeps. **KEEP.**

### 7.5 "Custom" mode: needs a product decision

`MapSelect` shows "Custom — coming soon" (`screens/MapSelect.tsx:25`). There are two readings:

- **(a) A third fixed mode** (e.g., CTF, King of the Hill): covered by §6.4.
- **(b) Parametrised rules** (FFA/TDM with custom duration, weapons, bots): the rules **read config constants directly**. `FFA.` appears 40 times in `ffa/rules.ts` and `TDM.` 14 times in `tdm/rules.ts`. Supporting per-match config means passing `config` into `createFreeForAll`/`createTeamDeathmatch`. This is mechanical, but it touches the most-checked files. It also needs the config on the wire (the welcome) and in the replay header.

**Ask the owner which one** before designing further. [Low confidence on what "Custom" means; High confidence on the cost of (b).]

---

## 8. Client / simulation / rendering separation

### 8.1 Evaluating the brief's intended direction

```
Gameplay state → Simulation → View adapter → Three.js
```

**Correct, and implemented.** Combatants and the rules hold the state. `simulation.step` mutates them and reports through `SimEvents`. `view.ts` reads state each frame (`place`, `animate`) and handles events. Three.js meshes never feed back. [High confidence]

```
React → Application/UI state → Game runtime
```

**Correct, and implemented.** `App.tsx` holds UI choices. `GameCanvas` calls `startGame()` (runtime) and receives the `Match` object. The HUD reads `Match` every frame through an imperative handle. React never steps the game. [High confidence]

### 8.2 Boundary table

| Concern | Owner | Talks to | Rule |
|---|---|---|---|
| Simulation | `sim/simulation.ts` | Rapier, rules, `SimEvents` | No meshes, audio, DOM, wall clock |
| Rapier physics | `sim/physics.ts`, `sim/drive.ts` | the sim; `net/prediction.ts` drives the same `drive.ts`; `view/pilot.ts` ray-casts for aim (non-authoritative) | One world per match; creation order is part of determinism |
| Three.js rendering | `view/view.ts` (+ `render/*`) | reads combatants; implements `SimEvents` | Never writes gameplay state |
| React UI | `screens/`, `hud/` | `Match` object; `MODE_VIEWS` | No per-frame re-render; no physics imports |
| Input | `view/input.ts` → `view/pilot.ts` → `Combatant.control` | DOM | Controls are data; the chat box stops keys (already) |
| Audio | `view/audio.ts`, `view/sounds.ts` | view, feed | Presentation only; unseeded randomness allowed |
| Effects | `view/effects.ts` | view | Pooled; unseeded |

### 8.3 One REVIEW item

`view.ts` reads the Rapier vehicle controller (`wheelIsInContact`, `poseWheels` from suspension length) for dust, tyre squeal and wheel poses. Online remote cars stay kinematic bodies in a local world, with `controller.updateVehicle(dt)` run only so these reads work (`net/client.ts step`). This is presentation reading the physics engine. It is acceptable while every client runs a local world. If a future client ever renders without local physics (spectator, replay viewer), wheel contacts must go into the pose data. **Postpone.** [High confidence on the mechanism]

---

## 9. Networking architecture

### 9.1 Current boundaries (mostly right)

| Concern | Module | Server-safe? | Verdict |
|---|---|---|---|
| Protocol (messages, quantisation, binary frame, validation, `PROTOCOL`, `BUILD`) | `net/protocol.ts` | **yes** (imported by the server) | **KEEP** (fix `weaponId`; rename its `Seat` → `SeatInfo`) |
| Wire events codec | split: `server/recorder.ts` (encode) + `net/client.ts` (decode) | – | **REFACTOR → `net/events.ts`** (server-safe) |
| Transport (socket, hello, `Link`, reasons) | `net/connection.ts` | runs in Node for checks | **KEEP** |
| Replication (mirror of a server match: snapshots, state, roster, events → view/feed) | `net/client.ts` | DOM-free | **KEEP** |
| Prediction/reconciliation | `net/prediction.ts` | DOM-free | **KEEP** |
| Interpolation | `net/snapshots.ts` | DOM-free | **KEEP** |
| Matchmaking (page store) | `net/matchmaking.ts` | browser (sessionStorage) | **KEEP** |
| Identity (Nakama) | `net/session.ts` | browser | **KEEP** |
| Chat | `net/chat.ts` + `net/chatCommand.ts` (pure) | browser / pure | **KEEP** |
| Online `MatchSource` | `game/online.ts` | browser | **KEEP**; move to `runtime/` |

### 9.2 The codec refactor (concrete)

Today, adding one `SimEvents` callback means changing:
- `simulation.ts` (the interface + the call),
- `view.ts` (the handler),
- `server/recorder.ts` (`events.push(['xx', tick, …positional])`),
- `net/client.ts` in two places (`play()` decodes `f[n]`, `mine()` decides "the player's own"),
- and possibly `PROTOCOL`.

The client's `mine()` hard-codes that the victim of `'sh'` sits at `f[11]`.

**Target:** `net/events.ts` exports, per wire code:
- the field layout (names → positions) and quantisation,
- `encode(…)`, used by the recorder,
- `decode(row)`, used by the client and returning a typed object,
- `owners(row)`: the seat ids that make it "mine".

A round-trip check covers every code (`protocol.check`). **Bytes on the wire must not change**, so `PROTOCOL` stays at 5; a golden snapshot-bytes fixture from Phase 0 proves it.

### 9.3 Domain vs transport

The domain never touches the WebSocket (verified). Practice and online share the core through `MatchSource`:
- practice: `sim.step` locally;
- online: `client.receive/step/place`, where the local rules are a **mirror** (never ticked) and the local car is predicted with the same `drive.ts`.

**No new abstraction is needed.** [High confidence]

---

## 10. Server architecture

### 10.1 Current separation (already sound)

| Concern | Module | Verdict |
|---|---|---|
| Transport + door (origins, auth, rate, strikes, backlog, hello deadline) | `server.ts` | **KEEP**; optionally extract the dev latency simulator to `netsim.ts` |
| Authentication | `auth.ts` (HS256 verify/mint) | **KEEP** |
| Matchmaking policy | `matchmaker.ts` (pure, hooks) | **KEEP** |
| Room lifecycle and seating | `lobby.ts` | **KEEP** |
| Match hosting (simulation) | `room.ts` | **REFACTOR, light** (journal format, input policy) |
| Replay (write) | `room.ts` journal → `records.ts` | extract the format to `journal.ts` |
| Replay (read/run) | `replay.ts`, `replay-main.ts` | **KEEP**; import the format from `journal.ts` |
| Persistence | `records.ts` | **KEEP** |
| Fair-play | `fairplay.ts` (pure) + `room.watch()` (geometry) | **KEEP** (optionally move `watch()` beside `fairplay.ts` when `room.ts` is split) |
| Lag compensation | `rewind.ts` | **KEEP** |
| Headless arenas | `arenas.ts`, `headless.ts`, `digests.json` | **KEEP** |

### 10.2 Folder structure

**KEEP `server/` flat.** 13 production files, each cohesive. Subfolders would add path noise, churn `vite.server.config.ts` `ENTRIES`, and prevent no mistake. [High confidence]

### 10.3 Server-specific rules for the boundary check

- Production entries (`main.ts`, `load.ts`, `replay-main.ts`) must not reach `view/`, `runtime/`, `screens/`, `hud/`, React, Nakama JS, or `net/{connection,client,prediction,snapshots,matchmaking,session,chat}`.
- `server/*.check.ts` and `server/browser.ts` are exempt (they emulate pages on purpose).

---

## 11. Definition vs runtime state

| Pair | Today | Mixing? | Recommendation |
|---|---|---|---|
| `VehicleSpec` / `Combatant` + `Car` | Separate; the combatant holds a `vehicle` id and a `car` built from the spec | No | **KEEP** |
| `WeaponSpec` / `WeaponState` | `WeaponState.spec` holds a spec object; **bots hold a scaled copy** (`botGun`), so the definition is mutated per instance and identity is lost | **Yes** | **REFACTOR:** `id` on the spec (copies keep it). Longer term, optionally `WeaponState { id, modifiers }`, but `id` alone fixes the defect. [High confidence] |
| `MapInfo` / `Arena` | Registry entry vs built arena (immutable during play; `update()` is visual) | Layout is **derived from visual meshes** (`solid()` → `collectColliders`) | **REVIEW → postpone** (§18). For **new** maps only, allow an optional `layout()` separate from `dress()` if a map author wants it. Do not rewrite the two existing maps (digest risk). |
| `MatchDefinition` / `MatchRuntime` | Mode config (`FFA`, `TDM` constants) + `Seat`/`Recruit` vs rules state + combatants | No | **KEEP** (see §7.5 if Custom = parametrised) |
| `Loadout` (ids) / armed combatant | ids resolved in `createMatch`/`room.join` | No | **KEEP**; add `parseLoadout` |
| `Skill` (difficulty) / `Brain` | Separate | No | **KEEP** |
| `ITEMS` / live `Item` | Catalogue vs rules' live items | No | **KEEP** |

---

## 12. Registry architecture

| Registry | Location | Keyed by | Verdict |
|---|---|---|---|
| `VEHICLES` | `vehicle/vehicles.ts` | `VehicleId` (`keyof ROSTER`) | **KEEP** plain record; move to `content/vehicles/index.ts` with per-vehicle files |
| `MODELS` | `vehicle/vehicle.ts` | `VehicleId` | **KEEP**; own file `content/vehicles/models.ts` (client + headless-safe) |
| `WEAPONS` | `combat.ts` | `WeaponId` | **KEEP**; move to `content/weapons/index.ts`; add `id` |
| `TURRETS` | *(missing: ternary)* | turret key | **ADD** |
| `MAPS` | `maps.ts` | `MapId` | **KEEP**; move to `content/arenas/index.ts`; `loadArena` cache → `runtime/` |
| `MODES` | `modes.ts` | `Mode` | **KEEP**; drop the scenery wiring; add `teamPlay` |
| `MODE_VIEWS` | *(missing: branches in HUD/Results)* | `Mode` | **ADD** (client-only, UI level) |
| `MODE_SCENERY` | *(missing: `createPickups` wired inside `modes.ts`)* | `Mode` (partial) | **ADD** (client-only, view level) |
| `DIFFICULTIES` | `ai.ts` | `Difficulty` | **KEEP** |
| `ITEMS` | `ffa/items.ts` | `ItemType` | **KEEP** |
| `CUES`/`LOOPS` | `audio.ts` | `SoundCue` | **KEEP** |
| `LIVERIES`, `QUALITY`, `FACADES` | various | – | **KEEP** |

**Why plain records and not a `register()` API:** `Record<Id, …>` makes a missing entry a **compile error**. It needs no import-order or side-effect-import tricks, and it survives both Vite bundles and plain-node checks unchanged. Self-registering modules (`registry.add(...)` at import time) would depend on import order and tree-shaking, and could silently drop content from the server bundle. **"Feature folder + one explicit line in an index file" is the right design.** [High confidence]

Adding content therefore touches **the feature folder + 1–2 registry lines + (maps only) `digests.json`**. It never touches the engine.

---

## 13. Testing architecture

### 13.1 Keep the convention

Plain-node `.check.ts` files (Node's type stripping, `.ts` import extensions) + Vite-bundled server checks. **Do not adopt Vitest/Jest.** The suite is fast, dependency-free and matches `AGENTS.md`. Vitest sits unused in `devDependencies`: **REVIEW**, the owner decides whether to remove it. [High confidence]

### 13.2 Existing checks: keep or move

| Check | Covers | Keep? | Moves to (Phase 5) |
|---|---|---|---|
| `game/ai.check.ts` (20 lines) | router | keep | `sim/ai/ai.check.ts` |
| `game/bots.check.ts` | bot behaviour ranges on a headless yard | keep | `sim/ai/bots.check.ts` |
| `game/ffa/ffa.check.ts` (143) | FFA rules | keep | `modes/ffa/` |
| `game/tdm/tdm.check.ts` (163) | TDM rules + tactics | keep | `modes/tdm/` |
| `game/simulation.check.ts` (28) | both modes through the real sim; same-process determinism; content numbers | keep | `sim/` |
| `game/loading.check.ts` (23) | loading runner | keep | `runtime/` |
| `net/protocol.check.ts` (70) | wire round trips, binary frame, input rules | keep; **extend** with the codec round trip + golden bytes | stays |
| `net/chat.check.ts` (14) | chat command parsing | keep | stays |
| `server/matchmaker.check.ts` (63) | matchmaking on a manual clock | keep | stays |
| `server/fairplay.check.ts` (10) | fair-play signals | keep | stays |
| `server/arena.check.ts` (64) | both arenas headless + digests | keep; **extend** with the (map × hosted mode) matrix and every vehicle's shells | stays |
| `server/server.check.ts` (133) | real server, sockets, authority, replay equality, room = practice | keep | stays |
| `server/client.check.ts` (32) | headless pages through `net/` | keep | stays |
| `server/netplay.check.ts` (33) | prediction, interpolation, lag compensation under latency | keep (timing-dependent: range asserts only) | stays |

### 13.3 New checks

| New check | Why (concrete gap) | Type | Phase |
|---|---|---|---|
| **Golden fingerprints** (`server/golden.check.ts`, bundled because it needs the real arenas) | Nothing pins gameplay **across commits** today. Run bots-only, each mode × each arena, fixed seed, 60 s; hash poses (mm), hulls, stats, the rules' events and recorded wire events; compare with committed constants. Also run one practice-style simulation (no room) to pin the `createMatch` path. | bundled | **0** |
| **Golden wire bytes** (in `protocol.check`) | Moving/refactoring protocol code must not change bytes: fix a combatant state, pack a snapshot, a welcome and an input, compare with committed bytes/strings. | node | **0** |
| **Boundaries check** (`game/boundaries.check.ts`) | Enforce §4 mechanically (§14). | node | **1** |
| **Registry consistency** | `WEAPONS[k].id === k`; every `spec.turret` exists in `TURRETS`; every map's `modes` entries line up headless; `MODE_VIEWS` completeness (compile-time via `Record`) | node + bundled | **2 / 4** |
| **Wire events round trip** | every `SimEvents` code encodes/decodes to the same values; `owners()` agrees with the old `mine()` table | node | **3** |
| **Input policy** (`server/inputs.check.ts`) | queue/drain/stale behaviour in isolation (today only covered end-to-end) | node | **6** |
| **Replay fixture** (optional) | A committed, short room journal (~KBs) with joins/leaves and human inputs, plus its expected records: proves `replay.ts` + `journal.ts` stay compatible across refactors. Must be re-pinned (in a separate commit) whenever gameplay changes on purpose. | bundled | 0 (optional) |

### 13.4 How determinism and replay testing fit

- **Same-process determinism** (existing): catches nondeterminism, such as a stray `Math.random` or Map iteration order.
- **Golden fingerprints** (new): catch *behaviour change*, such as a reordered RNG draw, a changed constant, a reordered body creation or a moved step.
- **Room = practice** (existing, server.check): catches drift between the server path and the practice path.
- **Replay equality** (existing, server.check): the journal replays to identical records within a build.
- **Arena digests** (existing): layout parity across builds and machines.

**Caveat on the golden hashes:** they depend on Rapier's WASM (the same binary everywhere, so IEEE-deterministic) and on V8's `Math.sin/cos/atan2/exp/hypot`. V8's implementations are software ports and consistent across platforms in practice. A **Node/V8 upgrade** or a **Rapier upgrade** may legitimately change the hashes. **Needs verification in Phase 0:** run the golden check on CI (Node 24) and on one developer machine (e.g., Node 22 on macOS ARM), and confirm the hashes match. If they differ by platform, pin per platform or fall back to tolerance-based fingerprints. [Medium confidence]

---

## 14. Dependency enforcement

| Option | Enforces | Cost | Verdict |
|---|---|---|---|
| **TypeScript path aliases** (`@sim/…`) | nothing by itself (cosmetic) | **Breaks the plain-node checks**: Node's type stripping does not resolve `tsconfig` paths without a loader | **Reject** [High confidence] |
| **oxlint `no-restricted-imports` + `overrides`** | per-folder import bans | Config only. The installed oxlint (1.85) has `overrides` in its schema. **Rule support needs verification.** It cannot express "reachable from the server entry" (transitive) or "no `Math.random` in sim". | **Optional complement** (Medium) |
| **dependency-cruiser** | full graph rules | A new dev dependency + config DSL; the repo's rules favour no new dependencies | **Reject for now** (Medium) |
| **Separate packages/workspaces** | hard boundaries | Breaks the one-package Vite setup, the build id, the shared server bundle and the checks | **Reject** [High confidence] |
| **Per-layer `tsconfig` projects** (e.g., `sim` without the DOM lib) | compile-time "no DOM in sim" | `@types/three` references DOM types; `skipLibCheck` may make it workable. **Needs a spike.** | **REVIEW (optional, later)** (Low) |
| **Custom node check** (`boundaries.check.ts`) | folder/file rules, **transitive reachability** from server entries, banned globals per folder, 0 cycles | About 150 lines, no dependency; runs in `npm run check` | **Recommend** [High confidence on need, Medium on exact mechanism] |

### 14.1 Rules for `boundaries.check.ts`

These are written for the **target** folders. Before Phase 5 the same rules are expressed as explicit file lists.

1. **No import cycles** (value or type). Today: 0.
2. **Server reachability:** from `server/{main,load,replay-main}.ts`, the transitive closure excludes `view/**`, `runtime/**`, `screens/**`, `hud/**`, `modes/*/{hud,results,scenery}.*`, `modes/views.ts`, `modes/scenery.ts`, `net/{connection,client,prediction,snapshots,matchmaking,session,chat}.ts`, `react*`, `@heroiclabs/*`.
3. **Gameplay purity:** files under `sim/**`, `shared/**` and `modes/**` (excluding client-only files and `*.check.ts`) must not import `render/**`, `view/**`, `runtime/**`, `net/**`, React, or `three/examples/**`. Their source must not contain `Math.random(`, `Date.now(`, `performance.now(`, `window.`, `document.` or `localStorage`.
4. **Layer direction** (folder pairs from §4.2).
5. **UI physics ban:** `screens/**` and `hud/**` must not import `@dimforge/rapier3d-compat`, `sim/physics`, or `sim/simulation` values (types are allowed).
6. **Import-time safety list:** modules with import-time browser side effects (today `view/audio.ts`) may only be reached from `view/`, `runtime/`, `screens/`, `hud/`.

It should print a readable table of violations. **Phase 1** lands it passing against today's code (§16). [High confidence on today's state: I ran the equivalent graph queries.]

---

## 15. File-by-file migration map

Legend: **keep** = no change · **move** = `git mv` + import paths only · **split** = content divided · **edit** = behaviour-preserving change in place · **new** = created. "Phase" refers to §16. Paths are relative to `game/src/` unless they start with `server/`.

### 15.1 Simulation core → `sim/`, `shared/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/simulation.ts` | `sim/simulation.ts` | move | 5 | Authoritative core; unchanged content |
| `game/combat.ts` | `sim/combat.ts` (WeaponState, armWeapon, pullTrigger, Shot, scatterAim, castRound, sustainedDps) + `content/weapons/{types,index}.ts` + `content/weapons/{minigun,rocketPod}/spec.ts` | edit (2: `id`, `turret`, `icon`) → split (5) | 2, 5 | Definitions vs mechanics; identity fix |
| `game/physics.ts` | `sim/physics.ts` | move | 5 | – |
| `game/vehicle/drive.ts` | `sim/drive.ts`; the `Handling`/`Chassis` types → `content/vehicles/types.ts` | move + type split | 5 | The driving model is sim; specs are content |
| `game/rng.ts` | `shared/rng.ts` | move | 5 | Used by sim, content and view |
| `game/scoring.ts` | `sim/scoring.ts` | move | 5 | – |
| `game/mode.ts` | `sim/mode.ts` | edit (4: drop `show`) → move (5) | 4, 5 | The contract belongs with its main consumer (the simulation) |
| (duplicates of `Life`, `Point`) | `shared/types.ts` | new (type-only) | 2 | Dedupe `mode.ts`/`ffa/rules.ts`/`tdm/types.ts`/`ffa/items.ts` |
| `game/loadout.ts` | `sim/loadout.ts` (type, default, `parseLoadout`) + `runtime/stored.ts` (`savedLoadout`, `saveLoadout`) | split | 2 | Pure validation shared with `parseClient`; storage is browser-only |
| `game/ai.ts` | `sim/ai/{skill,brain,perception,navigation,think}.ts` | move (5) → split (6) | 5, 6 | Five responsibilities; watch RNG order |
| `game/ai.check.ts`, `game/bots.check.ts` | `sim/ai/` | move | 5 | Tests follow their subject |
| `game/simulation.check.ts` | `sim/` | move | 5 | – |

### 15.2 Modes → `modes/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/modes.ts` | `modes/index.ts` (domain registry + `teamPlay`) + `modes/views.ts` (client UI) + `modes/scenery.ts` (client 3D) | split | 4, 5 | The server bundle stops pulling scenery; mode UI plugs in |
| `game/roster.ts` | `modes/roster.ts` | move | 5 | Depends on `MODES` + bot arming |
| `game/ffa/{config,items,rules,mode}.ts`, `ffa.check.ts` | `modes/ffa/…` | move (+ `share` effects loop, 4) | 4, 5 | Feature folder |
| `game/ffa/pickups.ts` | `modes/ffa/scenery.ts` | move + rename; client-only | 4 | Out of the adapter; into `MODE_SCENERY` |
| – | `modes/ffa/hud.tsx`, `modes/ffa/results.tsx` | new (extracted from `Hud.tsx`/`Results.tsx`) | 4 | Per-mode panels |
| `game/tdm/{config,types,rules,tactics,mode}.ts`, `tdm.check.ts` | `modes/tdm/…` | move | 5 | Feature folder |
| – | `modes/tdm/hud.tsx`, `modes/tdm/results.tsx` (MVP) | new (extracted) | 4 | Per-mode panels |

### 15.3 Content → `content/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/vehicle/vehicles.ts` | `content/vehicles/index.ts` (VEHICLES) + `content/vehicles/razor/spec.ts` + `content/vehicles/types.ts` | split | 5 | Vehicle feature folder |
| `game/vehicle/vehicle.ts` | `content/vehicles/razor/model.ts` + `content/vehicles/models.ts` (MODELS) | edit (2: turret via `TURRETS`) → split (5) | 2, 5 | Removes the turret ternary |
| `game/vehicle/parts.ts` | `content/parts.ts` (wheel, tyre, blade, lamp, spike) + `content/weapons/{minigun,rocketPod}/turret.ts` + `content/weapons/turrets.ts` | split | 2, 5 | Turrets belong to weapons; parts are shared by vehicles and arenas |
| `game/arena/arena.ts` | `content/arenas/arena.ts` | move | 5 | Contract + kit helpers |
| `game/arena/digest.ts` | `content/arenas/digest.ts` | move | 5 | Pure |
| `game/arena/{props,ground,buildings,street}.ts` | `content/arenas/kit/…` | move | 5 | Shared by both maps |
| `game/arena/scrapyard.ts` | `content/arenas/scrapyard/scrapyard.ts` | move | 5 | Feature folder |
| `game/arena/city.ts` | `content/arenas/city/city.ts` | move | 5 | Feature folder |
| `game/maps.ts` | `content/arenas/index.ts` (MAPS, MapId, mapsFor); `loadArena` → `runtime/runtime.ts` | split | 5 | The registry is content; the session cache is runtime |
| `game/{VehicleGenerator,ArenaGenerator,proceduralTexture}.ts` | `content/legacy/` or delete | **owner decision** | 8 | Unused; `AGENTS.md` keeps them as reference |

### 15.4 Render infrastructure → `render/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/renderer.ts` | `render/renderer.ts` | move | 5 | The one WebGL context; lazy |
| `game/materials/*` | `render/materials/*` | move | 5 | Shared GPU resources; server-reachable (headless path) |
| `game/geometry.ts` | `render/geometry.ts` | move | 5 | Geometry helpers + `disposeGeometries` |
| `game/environment.ts` | `render/environment.ts` | move | 5 | Sky/sun |
| `game/postprocessing.ts` | `render/postprocessing.ts` (owns the `Quality` type) | move | 5 | Removes the `settings` ↔ `postprocessing` type edge direction issue |

### 15.5 Presentation → `view/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/view.ts` | `view/view.ts` | move (+ `spec.cue`, 2) | 2, 5 | – |
| `game/pilot.ts`, `game/input.ts`, `game/camera.ts` | `view/…` | move | 5 | The local control source + camera |
| `game/feed.ts` | `view/feed.ts` | move | 5 | Implements `Feed` |
| `game/effects.ts`, `game/audio.ts`, `game/sounds.ts` | `view/…` | move | 5 | `audio.ts` has import-time side effects: listed in boundary rule 6 |
| `game/settings.ts` | `view/settings.ts` | move | 5 | Read by camera/audio/HUD/runtime |
| `game/turntable.ts` | `view/turntable.ts` | move | 5 | Garage stage |

### 15.6 Runtime → `runtime/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/runtime.ts` | `runtime/runtime.ts` | move | 5 | Fixes the game↔net inversion |
| `game/match.ts` | `runtime/match.ts` (`playMatch`, `MatchSource`, `Match`) + `runtime/practice.ts` (`createMatch`) | move (5) → split (6) | 5, 6 | Symmetry with `online.ts` |
| `game/online.ts` | `runtime/online.ts` | move | 5 | – |
| `game/loading.ts`, `loading.check.ts` | `runtime/…` | move | 5 | – |

### 15.7 Net, UI, server (mostly unchanged)

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `net/protocol.ts` | same path (**must not move**: `scripts/match-smoke.mjs` reads it) | edit: `weaponId → spec.id`; `Seat` → `SeatInfo`; use `parseLoadout` | 2 | Identity; naming; dedupe |
| – | `net/events.ts` | new | 3 | Shared wire-event codec |
| `net/client.ts` | same | edit: decode via `events.ts` | 3 | Removes positional indices |
| `net/{connection,prediction,snapshots,matchmaking,session,chat,chatCommand}.ts` | same | keep | – | Cohesive |
| `hud/Hud.tsx` | `hud/Hud.tsx` (shared) + mode panels in `modes/*/hud.tsx` (+ optional `hud/{compass,vitals,weapon,feed,markers,scoreboard}.ts`) | split | 4, 6 | Mode branches out; weapon icon from data |
| `hud/minimap.ts` | same; marks + colours passed in by the mode panel | edit | 4 | Drops the `ffa/items` import |
| `hud/Chat.tsx` | same | keep | – | – |
| `screens/Results.tsx` | same (generic frame) + `modes/*/results.tsx` | split | 4 | Mode branches out |
| `screens/GameCanvas.tsx` | same + `screens/game/{PauseMenu,ExitConfirm,LoadingOverlay}.tsx` | REVIEW split | 6 | Readability |
| `screens/Garage.tsx` | same; vehicle pager | edit | 7 | Second-vehicle capability |
| other `screens/*`, `App.tsx`, `main.tsx`, `analytics.ts` | same | keep (import paths only) | 5 | – |
| `server/room.ts` | `room.ts` + `server/journal.ts` + `server/inputs.ts` (+ optional `server/matchRecord.ts`) | split | 6 | A persisted format and a tuned policy get their own modules |
| `server/replay.ts` | same; imports `journal.ts` | edit | 6 | One definition of the format |
| `server/recorder.ts` | same; encodes via `net/events.ts` | edit | 3 | Shared codec |
| `server/server.ts` | same + optional `server/netsim.ts` | REVIEW split | 6 | Dev tool vs production door |
| `server/{lobby,matchmaker,fairplay,records,rewind,auth,arenas,headless,main,load,replay-main,browser}.ts`, `digests.json` | same | keep | – | Cohesive |

### 15.8 Non-source files that reference paths (must be updated with the moves)

| File | Reference | Phase |
|---|---|---|
| `game/package.json` `check` script | `src/game/*.check.ts` paths | 5 |
| `scripts/arena-parity.mjs` | `src/game/maps.ts`, `physics.ts`, `arena/digest.ts` | 5 |
| `.claude/work/ffa/ffa-metrics.js`, `.claude/work/tdm/tdm-metrics.js` | `import('/src/game/ai.ts')`, `combat.ts`, `ffa/config.ts`, `tdm/config.ts` | 5 |
| `scripts/match-smoke.mjs` | `game/src/net/protocol.ts` (do **not** move) | – |
| `game/AGENTS.md` code map, root `AGENTS.md`, `.claude/work/arch/*.md`, `net/NET_ARCHITECTURE.md` | many paths | every phase that moves files |
| `vite.server.config.ts` `ENTRIES` | server paths (unchanged) + new `golden.check` | 0 |

---

## 16. Migration phases

Global invariants for **every** phase (the definition of "behaviour-preserving"):

- `npm run lint` (no new warnings), `npx tsc -b`, `npm run build`, `npm run check` are all green.
- **Golden fingerprints unchanged** (from Phase 0 on). `server/digests.json` unchanged. `PROTOCOL` stays **5** (except Phase 7).
- The build id changes with every PR. That is expected: pages open across a deploy are told to reload.
- One concern per PR. Moves (`git mv`) never share a PR with content edits, so review and `git log --follow` stay useful.
- Every PR updates the docs it invalidates (`game/AGENTS.md` code map at minimum). This is a repo rule.

---

### Phase 0: Baseline and characterization (test-only)

| | |
|---|---|
| **Goal** | Make behaviour change detectable across commits before touching production code. |
| **Files affected** | New: `server/golden.check.ts`. Edited: `net/protocol.check.ts` (golden bytes), `vite.server.config.ts` (`ENTRIES`), `package.json` (`server:check`). Optional: `server/fixtures/room.ndjson.gz` + expected records. |
| **What changes** | (1) Golden fingerprints: bots-only matches for {tdm, ffa} × {scrapyard, city}, fixed seeds, 60 s each, hashed (poses to mm, hull, stats, rules events, recorder wire events) against committed constants. (2) Golden wire bytes for one snapshot frame, one welcome, one input, one state. (3) Record the baseline table (§0.3) in the PR description. (4) Manual browser smoke checklist (§17) run once and recorded. |
| **What must NOT change** | Any production file. |
| **Tests/checks** | The new checks pass on CI (Node 24) **and** on a developer machine (verifies cross-machine stability; §13.4). |
| **Expected risk** | Low (test-only). One risk: platform-dependent hashes. Mitigation: per-platform pins or a tolerance fallback. |
| **Rollback** | Revert the PR. |
| **Definition of done** | Fingerprints committed and green on CI + one other machine; the baseline recorded. |

### Phase 1: Boundary check (codify today's architecture)

| | |
|---|---|
| **Goal** | Mechanically prevent regressions of the boundaries that already hold. |
| **Files affected** | New: `game/boundaries.check.ts` (plain node, no dependencies). Edited: `package.json` `check`. Optional: the (map × mode) line-up matrix in `server/arena.check.ts`. |
| **What changes** | Rules from §14.1, expressed as **file lists for the current layout**: "server-reachable set", "gameplay files", "browser-only files". It lands **enforcing** (today's code passes it). |
| **What must NOT change** | Production code. |
| **Tests/checks** | The check passes. Negative test: temporarily add `import '../game/audio'` to `simulation.ts` → the check fails with a clear message (done in review, not committed). |
| **Expected risk** | Low. Main risk: false positives from dynamic `import()` or `export … from` forms. Handle both. |
| **Rollback** | Revert; or remove it from the `check` script. |
| **Definition of done** | In `npm run check`; documented in `game/AGENTS.md` ("Checks" section); `MODULE_BOUNDARIES.md` points to it as the source of truth. |

### Phase 2: Content identity and registration

| | |
|---|---|
| **Goal** | Make weapons plug-and-play and fix the identity defect; small dedupes. |
| **Files affected** | `game/combat.ts`, `game/ai.ts` (`botGun` keeps `id`; optional `spec.ai`), `net/protocol.ts` (`weaponId`, `Seat` → `SeatInfo`, `parseLoadout`), `game/vehicle/vehicle.ts` + `parts.ts` (`TURRETS`), `hud/Hud.tsx` (icon from `spec.icon`), `game/view.ts` (`spec.cue`), `game/loadout.ts` (+ `runtime`-side storage split, or keep until Phase 5), `game/mode.ts`/`ffa/*`/`tdm/types.ts` (shared `Life`/`Point`), `server/room.ts` + `rewind.ts` + `recorder.ts` (callers of `weaponId`), checks. |
| **What changes** | `WeaponSpec.id` (equal to its registry key, asserted). `WeaponSpec.turret: string` replaces the `model` union (the HUD `data-kind` uses the turret/icon). A `TURRETS` registry. `WeaponSpec.icon` (SVG path data). Optional `cue`/`ai` with today's values as defaults. `parseLoadout`. Shared types. **Split into 3–4 PRs** (see §22). |
| **What must NOT change** | Wire bytes (ids are the same strings), fingerprints, visuals (the turret meshes and HUD icons must be pixel-identical), bot RNG draws. |
| **Tests/checks** | Golden fingerprints; golden wire bytes; new registry consistency checks; `server.check` (records carry weapon ids); manual: garage turntable swaps both turrets, HUD weapon panel shows both icons. |
| **Expected risk** | Low-medium. `weaponId` has several callers (room, recorder `ro`, records, rewind); a missed caller is a type error once `model` is renamed. |
| **Rollback** | Revert per PR (each is self-contained). |
| **Definition of done** | Adding a test-only third weapon (hitscan, reusing the minigun turret) in a scratch branch touches only its spec + `WEAPONS`, and every check passes with the correct id on the wire. |

### Phase 3: Wire event codec

| | |
|---|---|
| **Goal** | One definition of each wire event, used by both the encoder and the decoder. |
| **Files affected** | New: `net/events.ts`. Edited: `server/recorder.ts`, `net/client.ts` (`play`, `mine`), `net/protocol.check.ts`. |
| **What changes** | Per code (`sh`, `ln`, `rk`, `bu`, `hu`, `wr`, `cr`, `rl`, `sp`, `rc`, `ru`, `go`): the field layout, `encode`, `decode`, `owners`. The recorder and client call them. The rocket-flight joining stays in the recorder. |
| **What must NOT change** | Bytes/JSON on the wire (`PROTOCOL` 5); **event playback order** (immediate for the player's own events, deferred to the drawn tick for others, catch-up semantics). |
| **Tests/checks** | Golden wire bytes; codec round trip; `client.check`, `netplay.check`, `server.check` (replay). |
| **Expected risk** | Medium: an index mistake silently mis-renders effects. The round-trip check and the old-vs-new `owners` table catch it. |
| **Rollback** | Revert (single PR). |
| **Definition of done** | No positional `f[n]` access left in `net/client.ts`; adding a hypothetical event is one codec entry + one handler. |

### Phase 4: Mode seam completion

| | |
|---|---|
| **Goal** | A third mode touches no shared file. |
| **Files affected** | `game/mode.ts` (drop `show`), `game/modes.ts` (drop scenery wiring, add `teamPlay`), `game/ffa/mode.ts` (no `scenery` parameter; `share`/`mirror` effects loop), `game/ffa/pickups.ts` → client scenery, new `modes/views.ts` + `modes/scenery.ts` (or `game/modeViews.ts` + `game/modeScenery.ts` before Phase 5), new FFA/TDM hud/results panels, `hud/Hud.tsx`, `hud/minimap.ts`, `screens/Results.tsx`, `game/runtime.ts` / `match.ts` (scenery lifecycle), `server/room.ts:219`, `net/client.ts` (`seatOnline` no longer passes `scene` to the mode). |
| **What changes** | The `MODE_VIEWS` and `MODE_SCENERY` registries; per-mode HUD/results panels; scenery driven by the runtime; `teamPlay`. **Split into 3 PRs:** (a) scenery out of `MatchMode`; (b) `teamPlay` + the map×mode check; (c) HUD/Results panels. |
| **What must NOT change** | HUD/Results visuals and per-frame cost; pickup visuals (blink timing, pooling, compile-time materials, which are pre-parked at y=−100 so the shader compile covers them); `restart`/`dispose` order; online mirroring. |
| **Tests/checks** | All checks; fingerprints unchanged (no sim change); **manual visual parity** (practice TDM + FFA on both arenas: score panels, board, clock through overtime, effect chips, scoreboard with Tab and on death, results for win/draw/loss, the MVP, online people line); a performance spot-check (F3 FPS, no new per-frame allocations: a Chrome allocation timeline for 10 s). |
| **Expected risk** | Medium (UI parity is manual). |
| **Rollback** | Revert per PR. |
| **Definition of done** | `grep "mode.kind ==="` finds nothing in `hud/` or `screens/`; `room.ts` has no mode id literals; the server bundle no longer contains `pickups`/`scenery` code (grep `dist-server`). |

### Phase 5: Folder reorganisation (mechanical)

| | |
|---|---|
| **Goal** | Folders express the boundaries; content becomes feature folders. |
| **Files affected** | Everything under `src/game/` (§15), import paths across `src/` and `server/`, `package.json`, `scripts/arena-parity.mjs`, balance probes, docs. |
| **What changes** | `git mv` in **bottom-up order, one PR per layer:** (a) `shared/` + `render/`; (b) `content/` (with the vehicle/weapon/arena feature folders, incl. the `combat.ts` and `vehicles.ts` splits); (c) `sim/` (incl. `ai.ts` moved whole); (d) `modes/`; (e) `view/`; (f) `runtime/` (removes `src/game/`). Then switch the boundaries check from file lists to folder globs. |
| **What must NOT change** | File contents beyond import specifiers (and the pre-agreed type/registry splits in (b)); the `.ts` import extension convention for node-run modules; `net/protocol.ts`'s path; digests; fingerprints; bytes. |
| **Tests/checks** | Everything; plus `node scripts/arena-parity.mjs` locally (needs global Playwright, per the root `AGENTS.md`) and one balance-probe run in the dev build to confirm the probe imports resolve. |
| **Expected risk** | Low per PR but high churn: merge conflicts with in-flight work. Mitigation: announce a short freeze per layer PR; land each PR quickly. |
| **Rollback** | Revert the layer PR (pure moves revert cleanly). |
| **Definition of done** | `src/game/` no longer exists; the boundaries check uses folder rules; `AGENTS.md` code map and `MODULE_BOUNDARIES.md` rewritten to the new tree. |

> **Optional-phase note:** Phase 5 is the **least valuable per unit of risk**. If no third mode or second vehicle is planned in the next few months, stop after Phase 4 and 6. Phases 1–4 deliver the plug-and-play fixes and the enforcement without moving any file. [Medium confidence]

### Phase 6: Split oversized modules

| | |
|---|---|
| **Goal** | Smaller, single-responsibility modules where it pays. |
| **Files affected** | `sim/ai.ts` → `sim/ai/*`; `server/room.ts` → `journal.ts`, `inputs.ts` (+ optional `matchRecord.ts`); `server/replay.ts`; `runtime/match.ts` → `practice.ts`; `hud/Hud.tsx` shared parts (optional); `screens/GameCanvas.tsx` (optional); `server/server.ts` → `netsim.ts` (optional). |
| **What changes** | Code moves between files; **no statement reordering** inside `think()`, `room.step()` or `playMatch.frame()`. |
| **What must NOT change** | RNG draw order; room step order; journal line format; replay compatibility of journals written by the same build. |
| **Tests/checks** | Fingerprints; `bots.check`; `server.check` replay; new `inputs.check.ts`; manual: bots in practice look the same over a 2-minute match. |
| **Expected risk** | Medium (the AI split is the riskiest file-level change in this plan). |
| **Rollback** | Revert per module PR. |
| **Definition of done** | No production file over ~450 lines except content builders (`city.ts`, `scrapyard.ts`, `recipes.ts`) and the cohesive rules files. |

### Phase 7: Second-vehicle capability (a feature, not a refactor; only when scheduled)

| | |
|---|---|
| **Goal** | Seats honour `loadout.vehicle`; bots may draw vehicles; the garage pages vehicles. |
| **Files affected** | `server/room.ts` (`join`/`leave`/`takeWheel`: replace the car body in place: free the old body, `createCar`, `placeCar` at the current pose), `net/protocol.ts` (`ro` + vehicle, **`PROTOCOL` 6**), `server/journal.ts` (`join` + vehicle), `net/client.ts` (`roster` refit + body swap), `view/view.ts` (`refit` by vehicle), `modes/roster.ts` (bot vehicle draw from its own stream), `screens/Garage.tsx` (pager), `server/arena.check.ts` (every vehicle's shells), `server/server.check.ts` (replace the one-vehicle guard with a two-vehicle test using a check-only spec). |
| **What changes** | Behaviour changes on purpose, so **re-pin the golden fingerprints in a separate commit**, with a justification. |
| **What must NOT change** | Practice/online parity; replay equality within a build; the rewind (reads `chassis.shells` per machine). |
| **Tests/checks** | All; a new mid-match takeover with a different vehicle in `server.check`; netplay with the second vehicle. |
| **Expected risk** | **High**: Rapier body replacement mid-match (handles, colliders, the vehicle controller, the prediction state machine). |
| **Rollback** | Revert; `PROTOCOL` goes back to 5 with the revert. |
| **Definition of done** | A person can pick vehicle B in the garage and play it online; bots use both; replays reproduce it. |

### Phase 8: Documentation and cleanup

| | |
|---|---|
| **Goal** | Docs match the code; decide the loose ends. |
| **Files affected** | `game/AGENTS.md`, root `AGENTS.md` (repo layout line), `.claude/work/arch/*` (`ARCHITECTURE.md`, `MODULE_BOUNDARIES.md`, `STATE_OWNERSHIP.md`, `GAME_LOOP.md`), `.claude/work/net/NET_ARCHITECTURE.md` ("Adding content, online"). Owner decisions: legacy generators, the Vitest dev dependency. |
| **What changes** | Docs; possibly remove unused files/dependencies (owner's call). |
| **What must NOT change** | Code behaviour. |
| **Tests/checks** | All green; the `boundaries.check` rules quoted verbatim in `MODULE_BOUNDARIES.md`. |
| **Expected risk** | Low. |
| **Rollback** | Revert. |
| **Definition of done** | The "How to add …" section lists exactly the steps in §6 of this plan. |

---

## 17. Behaviour that must not change

| Behaviour | Where it lives | How it is verified |
|---|---|---|
| Practice match (player + 7 bots, difficulty) | `createMatch`, roster, sim | golden fingerprints; `simulation.check`; manual smoke |
| FFA (rules, items, hot zones, overtime, standings, nemesis) | `ffa/*` | `ffa.check` (143); fingerprints; manual HUD/Results |
| TDM (team score, protection ends on firing, MVP, overtime) | `tdm/*` | `tdm.check` (163); fingerprints; manual |
| Bots (targeting, routing, cover, ambushes, per-difficulty skill, weapon draw) | `ai.ts`, `roster.ts`, tactics | `bots.check` (ranges); fingerprints |
| Vehicle physics (ray-cast car, upend recovery, stuck recovery) | `drive.ts`, `simulation.ts` | fingerprints; `simulation.check` "it drives"; `netplay.check` (prediction error) |
| Weapons (fire rate, magazine, reload, spread, rockets, blast shove/falloff) | `combat.ts`, `simulation.ts` | fingerprints; `simulation.check`; `server.check` fire-rate authority |
| Combat, damage, protection, wrecks | sim + rules | same |
| Scoring and statistics | `scoring.ts` + rules | rules checks; `server.check` records |
| Respawn (wait, spawn choice with sight lines) | rules `pickSpawn`, sim `respawn` | rules checks; fingerprints |
| Map loading (session cache, headless parity) | `maps.ts`, `arenas.ts` | `arena.check` digests; `arena-parity.mjs` (manual) |
| Loadout (ids, persistence, server fallback for unknown ids) | `loadout.ts`, `parseClient` | `protocol.check`; manual reload |
| Matchmaking (queue, groups, ready check, grace, backfill, requeue) | `matchmaker.ts`, `lobby.ts`, `net/matchmaking.ts` | `matchmaker.check` (63); `server.check` |
| Ready check | same | same |
| Online match (authority, forged/stale input, hold for page loads, next match) | `room.ts`, `server.ts` | `server.check` (133) |
| Prediction/reconciliation | `prediction.ts`, `client.ts` | `netplay.check`, `client.check` |
| Interpolation (DELAY 4 ticks, events at drawn tick) | `snapshots.ts`, `client.ts` | `netplay.check` |
| "Reconnect". In this codebase this means a **search** survives a dropped socket or reload for 15 s (grace + `sessionStorage`). A dropped *match* socket hands the seat to a bot; there is no mid-match rejoin of the same seat. | `matchmaker.ts`, `net/matchmaking.ts` | `server.check` "a drop and its grace"; manual reload during search |
| Replay (room journal → identical records, fair-play counts included) | `room.ts`, `records.ts`, `replay.ts` | `server.check` replay; `replay.js` tool |
| Chat (channels from the welcome, whispers by uid, typing never drives) | `net/chat.ts`, `hud/Chat.tsx`, `room.ts` channels | `chat.check`; `scripts/chat-smoke.mjs` (needs Nakama); manual |
| Fair-play tracking | `fairplay.ts`, `room.watch` | `fairplay.check`; `server.check` records |
| Build/protocol compatibility | `PROTOCOL`, `BUILD`, `digests.json` | `server.check` door; deploy smoke (`match-smoke.mjs`) |
| Loading (progress = finished/total, retry, cancel, failure paths) | `loading.ts`, `runtime.ts` | `loading.check` |
| Dev tooling (`window.match`, `window.camera`, `window.tick`, F3 overlay lines) | `runtime.ts`, `match.ts` | manual; balance probes rely on the `window.match` shape |

**Manual browser smoke checklist** (run in Phase 0 and after each UI-touching PR; the repo has no browser automation in CI):
1. Startup → menu (no console errors).
2. Garage: swap weapons; the turret updates.
3. Practice TDM on the Scrapyard: countdown, fight, get wrecked (death board), respawn, Tab scoreboard, pause/resume, settings change, forced end → results → Play again.
4. Practice FFA on The City: pickups, hot zone, effect chips, standings, results (placing, crown).
5. Online: a local server + two tabs (Find Match → ready check → match), chat, leave → a bot takes over.
6. Exit to garage: `window.match` cleared; the renderer holds the same geometries/textures count as before (the `LOADING_ARCHITECTURE.md` method).

---

## 18. High-risk areas

| Area | Why it is dangerous | Guard |
|---|---|---|
| **Deterministic simulation / RNG streams** | Three seeded mulberry32 streams (`seed ^ 0x9e3779b9` sim, `seed ^ 0x2545f491` bot guns, `createRng(seed)` FFA rules). **Every `random()` call's position in the sequence matters.** Reordering a `think()` branch, iterating combatants in another order, or adding one draw shifts every later draw, so all later matches diverge. Online, prediction does not use RNG, but replays and practice↔room parity do. | fingerprints; `simulation.check` replay; `server.check` room=practice |
| **Rapier state and creation order** | Collider creation order (ground slab, arena colliders in array order, then one `world.step()` to build query structures), body creation order (`enlist` in seat order), shell order, wheel order (fl, fr, rl, rr), `setCanSleep(false)`, CCD, body-type switches (remote cars kinematic; the prediction toggles Dynamic/Kinematic). Handles and solver order change results. A known Rapier 0.20 quirk (NET_PLAN F5): moved kinematic bodies are invisible to ray casts until `world.step()`. | fingerprints; `netplay.check` |
| **Fixed timestep** | `PHYSICS_STEP = 1/60` (`physics.ts`) and `RATE.step = 60` (`protocol.ts`) are **two constants that must agree**; `world.timestep` is set explicitly; frame dt is clamped at 0.1 s; the server `CATCH_UP = 5`. Moving either constant without the other breaks the client `held()` GO-step maths. | a check asserting `RATE.step * PHYSICS_STEP === 1` (add in Phase 1) |
| **Prediction/reconciliation** | The client drives with controls **rounded to hundredths exactly as `parseClient` reads them**; `held(seq)` re-derives the server's pre-match hold step by summing steps the way the rules do; reconciliation replays inputs with `world.step()` on the whole local world; `firm` handling until the first ack. Small refactors (rounding, order of `drive` vs body placement) cause constant corrections. | `netplay.check` (corrections/errors printed and bounded) |
| **Snapshot interpolation** | `DELAY = 4` ticks; the server-tick reckoning from the least-delayed arrivals; `drawn` never goes backwards; events wait for the drawn tick except the player's own (`mine()`); catch-up after 1 s drops effects but keeps rules events. The codec refactor (Phase 3) touches exactly this. | `netplay.check`, `client.check`; old-vs-new `owners()` table |
| **Replay determinism** | Journal semantics: join/leave apply *after* step k, `in` lines apply *at* step k (`ahead()` in `replay.ts`); a given is written only when it changes, and the view is stored as lag. `room.step()` order: release hold → drive humans → journal → `sim.step` → `rewind.record` → rules events → report → outcome/restart → broadcast. Any reorder breaks replay equality. Journals are for fair-play review within `REPLAY_DAYS`; a deploy in that window already means replaying old-build journals on new code. That is a pre-existing limitation; do not make it worse by changing the line format without a version field. | `server.check` replay; the optional replay fixture |
| **Server/client protocol** | Binary layout (`HEAD_BYTES 14`, `CAR_BYTES 44`, `ME_BYTES 40`), clamps, JSON event arrays, `PROTOCOL` bump discipline, `parseClient` clamping. A refactor that changes bytes without a bump lets old pages mis-parse. | golden wire bytes; `protocol.check` |
| **Build id** | `build-id.ts` hashes `src/`, `server/` and the lockfile **including relative paths**. Every move changes the id, so every deploy makes open pages reload. That is expected but should be announced. `scripts/match-smoke.mjs` regex-reads `PROTOCOL` from `game/src/net/protocol.ts`: **moving that file breaks the deploy smoke test.** | keep `net/protocol.ts` in place |
| **Arena digests** | Colliders come from `solid()` on props, via `matrixWorld`, and are rounded to mm. Changing builder call order, a seeded draw, a prop dimension, or when `solid()` is called relative to placement changes the digest. The server and deployed pages then disagree ("Arena mismatch — reload"). | `arena.check` vs `digests.json`; `arena-parity.mjs` |
| **Headless arena builds** | The server runs visual builders with a fake `document` and no `window`; `materials/library.ts` skips the GPU bake when `window` is undefined. Adding `window` access at import time to any content module, or a new material path that ignores `headless`, crashes or diverges the server. | `arena.check`; boundaries rule 2/6 |
| **Asset lifecycle and Three.js disposal** | Library materials and baked textures are session-wide and **never disposed** by scene code. Geometries are disposed per match (`disposeGeometries`). The cached arena is borrowed by a scene and handed back by `scene.clear()`. `view.refit` rebuilds a model in place (the HUD keeps the `CarView` object). The server `strip()` disposes headless geometries. A "tidy" refactor that adds `material.dispose()` breaks the next match. **The first glTF assets will bring per-asset materials/textures that *do* need disposal:** the rule must be refined then. | `STATE_OWNERSHIP.md`; the renderer counts in the manual smoke |
| **Shared materials** | `memo` keys dedupe materials across the whole app; wreck charring swaps materials per mesh and restores from `paint`. Mutating a shared material (colour, uniforms) changes every mesh using it. | code review; manual smoke |
| **WebSocket lifecycle** | The matchmaking socket *becomes* the match link (`takeSeat`); `link.close()` semantics; StrictMode double-mount in dev ("a link whose match never ran is the caller's"); `App` closes the seat on exit; ping interval cleanup. | `client.check`, `server.check`; manual dev-mode run |
| **Matchmaking state** | Ticket grace (15 s), `sessionStorage` mark, retry backoff `[0, 1000, 2000, 4000, 7000]`, one live connection per user (a second tab takes the ticket), searching while seated is refused. | `matchmaker.check`, `server.check` |
| **`.ts` import convention** | Modules loaded by plain-node checks import local files **with `.ts`**. A refactor that "cleans up" extensions, or adds path aliases, breaks `npm run check` outside Vite. | CI |
| **HUD per-frame writes** | `update()` runs every frame and writes only changed values; no allocations; elements are collected once by `data-hud`. Mode panels must scope their queries to their own root, or the TDM panel can grab the FFA panel's slots. | manual perf/allocation spot-check |
| **Loading runner** | Reports must be painted before each task (`flushSync`); tasks must be idempotent for Retry; dispose is safe at any point. Moving loading code must keep these contracts. | `loading.check` |
| **Dev probes** | The balance probes import `/src/game/*.ts` paths and rely on the `window.match` shape (`match.mode.rules`, `match.mode.tactics`, `match.chase`, `match.others`). | Phase 5 updates the probes; one probe run |

---

## 19. Avoiding over-engineering

**Rejected, with the concrete reason for this codebase:**

| Idea | Why not here |
|---|---|
| **ECS** | About 8 combatants and a few rockets per match. Combatants are plain objects iterated in arrays; the step order *is* the determinism contract. An ECS would re-express everything, risk the RNG and Rapier order, and speed nothing up (a room step averages ~0.24 ms, measured by `server.check`). |
| **Event bus** | `SimEvents` (direct calls, one interface) and rules event queues (drained per step) already give ordering guarantees a bus would lose. Mid-step ordering of effects and sounds matters (`GAME_LOOP.md`). |
| **DI framework / service locator** | Composition happens in 3 places (`createMatch`, `createOnlineMatch`, `createRoom`) with explicit arguments. It is readable and checkable. |
| **Redux/Zustand** | React state holds 6 UI values in `App.tsx`; gameplay state must *not* be in React. The two external stores (`settings`, matchmaking) are 60–190-line modules with `onChange`/`useSyncExternalStore`. |
| **Physics abstraction** | One engine; prediction must run the *same* `drive.ts` on the *same* Rapier; hitscan, AI perception, item placement and aim are Rapier queries. An abstraction would be leaky and would put bit-exactness at risk. |
| **Replacing Three.js maths in the sim** | `THREE.Vector3/Quaternion` are used as maths throughout the hot paths; replacing them changes float operation sequences, so every fingerprint, replay and practice↔room parity result changes. Nothing is gained: the server bundles Three.js anyway (for the arena builders). |
| **Separate packages per feature / workspaces** | One Vite package produces two bundles from one build id; the checks run the source directly. Packages would break the build id, the shared server bundle and the check convention. |
| **Microservices** | One Node process runs rooms + matchmaking; Nakama is already the separate control plane. |
| **Abstract base classes / class hierarchies** | The codebase is factory functions returning objects (`createX`), typed by `ReturnType`. Stay consistent. |
| **A HUD view-model layer** | Per-mode panels (§7.2) solve the concrete problem (mode branches) without a per-frame object. |
| **Self-registering plugins** (`registry.add()` at import) | Loses compile-time completeness; depends on import order and bundling (§12). |
| **Generic `Clock`, `Audio`, `AssetLoader` interfaces** | No second implementation exists or is planned (§5). Add the asset loader with the first glTF asset. |
| **TS path aliases** | Break the plain-node checks (§14). |
| **`platform/` and `application/` layers as separate folders** | Too few files; `runtime/` covers the application role (§4.2). |

**Test for any new abstraction (use it in review):** *Which file would a developer otherwise have to edit to add content, or which bug does this prevent? Name it.* If you cannot, do not add it.

---

## 20. Target directory tree

```
game/
├── build-id.ts · vite.config.ts · vite.server.config.ts · tsconfig*.json · package.json   (unchanged; ENTRIES += golden.check)
├── boundaries.check.ts                   NEW (Phase 1): layer rules, server reachability, banned globals, cycles
├── public/                               (unchanged; public/models/ when the first glTF lands)
├── server/                               FLAT, as today
│   ├── main.ts · server.ts · auth.ts · lobby.ts · matchmaker.ts · room.ts
│   ├── journal.ts                        NEW (Phase 6): ReplayLine, Given, row encode/decode (room + replay)
│   ├── inputs.ts                         NEW (Phase 6): per-person input queue / drain / stale policy
│   ├── recorder.ts · rewind.ts · fairplay.ts · records.ts · replay.ts · replay-main.ts
│   ├── arenas.ts · headless.ts · digests.json · load.ts · browser.ts
│   └── *.check.ts (+ golden.check.ts NEW, Phase 0)
└── src/
    ├── main.tsx · App.tsx · analytics.ts · index.css           app shell (unchanged)
    ├── screens/                                                 React screens (unchanged; GameCanvas optionally split)
    ├── hud/                                                     Hud.tsx (shared panels only) · minimap.ts · Chat.tsx
    ├── runtime/                                                 browser composition (the "application" layer)
    │   ├── runtime.ts            startGame: loading steps, scene, composer, loop, arena session cache, dev globals
    │   ├── match.ts              playMatch, MatchSource, Match
    │   ├── practice.ts           createMatch (practice MatchSource)
    │   ├── online.ts             createOnlineMatch (online MatchSource)
    │   ├── loading.ts            runTasks, STARTUP (+ loading.check.ts)
    │   └── stored.ts             loadout persistence (localStorage)
    ├── view/                                                    what the local player sees/hears/controls
    │   ├── view.ts · feed.ts · pilot.ts · input.ts · camera.ts
    │   ├── effects.ts · audio.ts · sounds.ts · settings.ts · turntable.ts
    ├── render/                                                  shared Three.js infrastructure (server-reachable via arenas)
    │   ├── renderer.ts · environment.ts · postprocessing.ts · geometry.ts
    │   └── materials/ (library.ts · bake.ts · recipes.ts · facade.ts · canvasTextures.ts · groundGrime.ts · noise.ts)
    ├── sim/                                                     authoritative, headless, server-shared
    │   ├── simulation.ts · combat.ts · physics.ts · drive.ts · scoring.ts
    │   ├── mode.ts               MatchMode / ModeRules / Feed / ModeTiming contract (no rendering types)
    │   ├── loadout.ts            Loadout type, DEFAULT_LOADOUT, parseLoadout
    │   ├── ai/ (skill.ts · brain.ts · perception.ts · navigation.ts · think.ts · ai.check.ts · bots.check.ts)
    │   └── simulation.check.ts
    ├── modes/
    │   ├── index.ts              MODES (label, tags, blurb, teamPlay, lineUp, create); Mode
    │   ├── views.ts              MODE_VIEWS (client only, UI level: hud and results panels)
    │   ├── scenery.ts            MODE_SCENERY (client only, view level: pickups, zone)
    │   ├── roster.ts             recruits, bot names/vehicle/guns
    │   ├── ffa/  config.ts · items.ts · rules.ts · mode.ts · ffa.check.ts │ hud.tsx · results.tsx · scenery.ts (client)
    │   └── tdm/  config.ts · types.ts · rules.ts · tactics.ts · mode.ts · tdm.check.ts │ hud.tsx · results.tsx (client)
    ├── content/
    │   ├── parts.ts              wheels, tyres, blades, lamps, spikes (vehicles + arena props)
    │   ├── vehicles/ index.ts (VEHICLES) · models.ts (MODELS) · types.ts · razor/{spec.ts, model.ts}
    │   ├── weapons/  index.ts (WEAPONS) · turrets.ts (TURRETS) · types.ts · minigun/{spec.ts, turret.ts} · rocketPod/{spec.ts, turret.ts}
    │   ├── arenas/   index.ts (MAPS) · arena.ts (contract + kit helpers) · digest.ts
    │   │             kit/{props,ground,buildings,street}.ts · scrapyard/scrapyard.ts · city/city.ts
    │   └── legacy/   (owner decision: VehicleGenerator, ArenaGenerator, proceduralTexture)
    ├── net/                                                     unchanged location
    │   ├── protocol.ts           (must stay here: deploy smoke test reads it)
    │   ├── events.ts             NEW (Phase 3): wire-event codec shared with server/recorder.ts
    │   ├── connection.ts · client.ts · prediction.ts · snapshots.ts
    │   ├── matchmaking.ts · session.ts · chat.ts · chatCommand.ts
    │   └── *.check.ts
    └── shared/
        ├── rng.ts                mulberry32 (sim, content, view)
        └── types.ts              Point, Life, clock() formatter
```

**Dependency direction (enforced by `boundaries.check.ts`).** A module may import from its own level or any lower level, never from a higher one:

```
Level 6   screens/, hud/, modes/views.ts, modes/*/{hud,results}.tsx      React UI
Level 5   runtime/                                                        browser composition
Level 4   view/, net/, modes/scenery.ts, modes/*/scenery.ts               presentation, client networking, mode 3D
Level 3   modes/ (domain: index.ts, roster.ts, */config|rules|mode|…)     rules + adapters
Level 2   sim/                                                            authoritative gameplay
Level 1   content/                                                        specs, models, arenas
Level 0   render/, shared/                                                Three.js infrastructure; leaf utilities

server/   imports Levels 0–3 and net/{protocol,events}.ts only (Level 0 render/ only through content/arenas, headless)
net/protocol.ts and net/events.ts must additionally stay server-safe (no browser APIs)
```

**One extra rule on top of the levels** (§14.1, rule 3): `sim/` and the domain files of `modes/` must not import `render/` or `three/examples/**`, and use `three` only for maths. `render/` sits at Level 0 because `content/` needs it (arena and vehicle builders use the material library); the level order alone would otherwise let `sim/` reach it.

---

## 21. Architecture decision summary

| Decision | Recommendation | Why | Confidence |
|---|---|---|---|
| Big-bang restructure into the 9-layer tree | **No** | Most boundaries already exist; the cost is churn, broken probes/scripts/docs, conflicts | High |
| Golden behaviour fingerprints | **Yes, first** | Nothing pins gameplay across commits today | High (need) / Medium (cross-machine stability) |
| Mechanical boundary enforcement | **Yes**: custom `boundaries.check.ts` | Server safety is convention + a DOM shim; no dependency needed | High (need) / Medium (mechanism) |
| TS path aliases | **No** | Break the plain-node checks | High |
| dependency-cruiser | **No** (for now) | New dependency; a custom check covers the rules | Medium |
| oxlint `no-restricted-imports` | Optional complement | Needs verification of rule support | Low |
| Feature-based content folders | **Yes** (vehicles, weapons, arenas, modes), in Phase 5 | Content becomes "a folder + a registry line" | Medium |
| Domain layer | **Yes, as the `sim/` folder** = today's headless code; no new abstractions | The boundary exists; name and enforce it | High |
| Separate `engine/`, `platform/`, `application/` folders | **No** | Too few files; `runtime/` is the application layer | Medium |
| `render/` folder separate from `view/` | **Yes** | Materials are server-reachable; keeps the rules exception-free | Medium |
| Physics abstraction | **No** | One engine; determinism; prediction parity | High |
| Network abstraction | **No new one**; `Link` + `MatchSource` already are | Verified: practice and online share the core via `MatchSource` | High |
| Wire-event codec shared by server and client | **Yes** | Removes positional indices and a 3-file edit per event | High |
| Weapon `id` on the spec | **Yes** | Fixes identity-by-turret-model | High |
| Turret registry + icon/cue/ai as weapon data | **Yes** | Weapons stop editing `vehicle.ts`/`Hud.tsx`/`view.ts`/`ai.ts` | High |
| Remove rendering from `MatchMode` (`show`) | **Yes** | The server bundle pulls pickups; the contract leaks Three.js types | High |
| Per-mode UI panels (`MODE_VIEWS`) | **Yes** | 68 mode-referencing lines across `Hud.tsx`/`Results.tsx` | Medium-High |
| `teamPlay` flag on the mode registry | **Yes** | Removes `room.ts:219` | High |
| Event bus | **No** | `SimEvents` + rules queues keep ordering | High |
| ECS | **No** | Tiny entity counts; order-dependent determinism | High |
| DI framework / Redux / Zustand | **No** | Explicit composition in 3 places; React holds only UI state | High |
| Split `ai.ts` | **Yes, carefully** (Phase 6) | Five responsibilities; RNG order risk | Medium |
| Split `room.ts` (journal, inputs) | **Yes, partially** | A persisted format and a tuned policy deserve modules | Medium |
| Keep `server/` flat | **Yes** | 13 cohesive production files | High |
| Physics-free arena layout (separate from visuals) | **Postpone** | Digest risk on existing maps; offer it as an option for new maps | Medium |
| Mid-match vehicle swap / second-vehicle capability | **Only when scheduled** (Phase 7, `PROTOCOL` 6) | It is a feature | High |
| "Custom" mode | **Product decision first** | Parametrised rules = config injection into the most-checked files | Low |
| Adopt Vitest/Jest | **No**; the owner may remove unused Vitest | The `.check.ts` convention works | High |
| Replace Three.js maths in the sim | **No** | Changes float sequences → fingerprints/replays | High |

---

## 22. Final recommendation

### Recommended refactor strategy

**1. Definitely change**
- Add golden fingerprints and golden wire bytes (Phase 0) **before anything else**.
- Add `boundaries.check.ts` (Phase 1).
- Weapon identity (`spec.id`) + `TURRETS` + weapon icon/cue/ai as data (Phase 2).
- A shared wire-event codec (Phase 3).
- Remove rendering from `MatchMode`; add `teamPlay`; add per-mode HUD/Results panels (Phase 4).
- Dedupe `Life`/`Point`/`Seat`/loadout validation (Phase 2).

**2. Probably change**
- The folder reorganisation into `runtime/ view/ render/ sim/ modes/ content/ shared/` (Phase 5). Do it only if more content or contributors are coming soon.
- Split `ai.ts`; extract `journal.ts` + `inputs.ts` from `room.ts`; move `createMatch` to `practice.ts` (Phase 6).

**3. Do not change**
- The simulation/view/pilot/feed split.
- `MatchSource`.
- `MatchMode` + pure rules + `share`/`mirror`.
- `SimEvents` as a direct interface.
- The fixed step, the accumulator and the clamps.
- The seeded streams and their constants.
- Rapier creation order.
- The binary protocol layout, `PROTOCOL`/`BUILD`, and `net/protocol.ts`'s path.
- Arena builders and digests.
- The headless shim approach.
- The session caches and the "never dispose materials" rule.
- The imperative HUD writes.
- The `.check.ts` convention and `.ts` import extensions.
- The flat `server/` folder and the pure matchmaker.
- Nakama as the control plane only.

**4. Postpone**
- Second-vehicle seats (until a second vehicle is scheduled).
- A firing-behaviour table (until a third behaviour exists).
- A physics-free arena layout.
- A glTF asset loader and its disposal rules (until the first asset).
- Parametrised "Custom" rules (until the product decision).
- Per-layer tsconfig spikes.
- The `GameCanvas` split.
- The `netsim.ts` extraction.

### The first 5 PRs

| # | PR | Scope | Why it is safe | Verification | Revert |
|---|---|---|---|---|---|
| **1** | **Golden fingerprints + golden wire bytes** | New `server/golden.check.ts` (bots-only, {tdm, ffa} × {scrapyard, city}, fixed seeds, 60 s, hashed state + rules events + recorder events); a golden snapshot/welcome/input/state fixture in `net/protocol.check.ts`; `vite.server.config.ts` entry; `server:check` script | Test-only | Green on CI (Node 24) and on one other machine; mutation test: change `BLAST_SHOVE` by 0.1 locally → the golden check fails | trivial |
| **2** | **`boundaries.check.ts`** codifying today's rules | New plain-node check: 0 cycles; the server-reachable closure excludes browser-only files; gameplay files free of `Math.random`/`Date.now`/`performance.now`/`window`/`document`/`localStorage`; UI never imports Rapier; `audio.ts` reachable only from the browser side; `RATE.step * PHYSICS_STEP === 1` | Test-only; passes on today's code | Negative test in review (a temporary bad import fails with a clear message) | trivial |
| **3** | **Weapon identity** | `WeaponSpec.id` (== key, asserted); `botGun` preserves it; `weaponId(spec) → spec.id`; callers in `room.ts`, `recorder.ts`, records, `client.ts` updated | Same id strings on the wire; no RNG change | Fingerprints + wire bytes unchanged; `server.check` records; new check: a scaled bot copy keeps its id; scratch-branch experiment: a third gun reusing the minigun turret reports its own id | single PR |
| **4** | **Turret registry + weapon icon/cue as data** | `TURRETS` (turret key → builder); `WeaponSpec.turret` replaces the `model` union; the `vehicle.ts` ternary removed; `spec.icon` (SVG path) replaces the two hard-coded HUD SVGs; `spec.cue` with today's defaults | Presentation-only; sim untouched | Fingerprints unchanged; manual: the garage turntable shows both turrets identically; the HUD weapon panel identical for both weapons (screenshot compare); reload dimming still works | single PR |
| **5** | **Wire-event codec** | New `net/events.ts` (layout, encode, decode, owners per code); `server/recorder.ts` and `net/client.ts` use it; round-trip check | Bytes unchanged (`PROTOCOL` stays 5) | Golden wire bytes; codec round trip; `client.check`, `netplay.check`, `server.check` replay; an old-vs-new `owners()` table asserted for every code | single PR |

After these five: Phase 4 as three PRs (scenery out of `MatchMode` → `teamPlay` + map×mode check → HUD/Results panels). Then decide, with the owner, whether Phase 5 is worth it now.

### What would change this recommendation

- **A third mode or "Custom" is scheduled** → move Phase 4 up to PR 3, and get the §7.5 decision first.
- **A second vehicle is scheduled** → Phase 7 becomes a planned feature right after Phase 2 (it needs `PROTOCOL` 6).
- **The golden hashes are not stable across machines** → use tolerance-based fingerprints, and treat the same-process determinism check + room=practice parity as the primary guard. The plan stays the same; the safety margin for Phase 6 (the AI split) shrinks, so postpone that split.
- **Several people/agents start working in parallel on content** → do Phase 5 sooner; feature folders reduce merge conflicts more than anything else here.

---

## Appendix A: Import graph (current)

Internal edges of production files (`type:` = type-only import), from `game/`. Checks are omitted (they may import anything). Generated by the script in Appendix B.

```
App.tsx -> game/loadout, game/maps, game/modes, type:net/connection, net/matchmaking, screens/{GameCanvas,Garage,Loading,MainMenu,MapSelect,Matchmaking}
game/ai.ts -> type:arena/arena, combat, type:vehicle/drive
game/arena/arena.ts -> geometry, materials/library, arena/props
game/arena/buildings.ts -> geometry, materials/{facade,library,recipes}, rng, arena/props
game/arena/city.ts -> geometry, materials/{facade,library}, rng, arena/{arena,buildings,ground,props,street}
game/arena/digest.ts -> type:arena/arena
game/arena/ground.ts -> geometry, materials/library
game/arena/props.ts -> geometry, materials/library, vehicle/parts
game/arena/scrapyard.ts -> geometry, materials/library, rng, arena/{arena,ground,props,street}
game/arena/street.ts -> geometry, materials/library, rng, vehicle/parts, arena/{buildings,props}
game/audio.ts -> settings, sounds
game/camera.ts -> settings
game/environment.ts -> materials/noise
game/feed.ts -> audio, type:mode, scoring, type:view
game/ffa/items.ts -> ffa/config
game/ffa/mode.ts -> type:ai, type:arena/arena, mode, ffa/{config,items,rules}
game/ffa/pickups.ts -> materials/library, ffa/{config,items}
game/ffa/rules.ts -> rng, scoring, ffa/{config,items}
game/loading.ts -> physics, renderer
game/loadout.ts -> combat, vehicle/vehicles
game/maps.ts -> type:arena/arena, arena/city, arena/scrapyard, type:modes
game/match.ts -> ai, type:arena/arena, arena/digest, audio, combat, feed, type:loadout, type:maps, modes, physics, pilot, roster, simulation, vehicle/drive, view
game/materials/bake.ts -> materials/noise, type:materials/recipes
game/materials/canvasTextures.ts -> rng
game/materials/facade.ts -> materials/recipes
game/materials/library.ts -> renderer, materials/{bake,canvasTextures,facade,groundGrime,recipes}
game/mode.ts -> type:ai, type:arena/arena, type:scoring
game/modes.ts -> type:arena/arena, ffa/{config,mode,pickups}, type:simulation, tdm/{config,mode}
game/online.ts -> net/client, type:net/connection, type:arena/arena, feed, type:maps, match, simulation, view
game/physics.ts -> type:arena/arena
game/pilot.ts -> type:camera, input, type:simulation
game/postprocessing.ts -> type:settings
game/roster.ts -> ai, type:arena/arena, type:combat, modes, rng, type:simulation, type:vehicle/vehicles
game/runtime.ts -> net/connection, type:ai, arena/digest, audio, environment, loading, type:loadout, maps, match, type:modes, online, physics, postprocessing, renderer, settings
game/simulation.ts -> ai, type:arena/arena, combat, type:mode, rng, scoring, vehicle/drive, vehicle/vehicles
game/sounds.ts -> rng
game/tdm/mode.ts -> type:ai, type:arena/arena, mode, tdm/{config,rules,tactics}, type:tdm/types
game/tdm/rules.ts -> scoring, tdm/config, type:tdm/types
game/tdm/tactics.ts -> tdm/config, type:tdm/types
game/turntable.ts -> type:combat, environment, geometry, materials/library, renderer, vehicle/vehicle, type:vehicle/vehicles
game/vehicle/drive.ts -> physics
game/vehicle/parts.ts -> geometry, materials/library
game/vehicle/vehicle.ts -> type:combat, geometry, materials/library, rng, vehicle/{parts,vehicles}
game/vehicle/vehicles.ts -> type:geometry, type:vehicle/drive
game/view.ts -> type:arena/arena, audio, camera, effects, geometry, materials/library, type:simulation, vehicle/{drive,vehicle,vehicles}
hud/Hud.tsx -> ffa/{config,items}, type:ffa/rules, type:match, mode, modes, settings, type:simulation, tdm/config, hud/minimap
hud/minimap.ts -> type:arena/arena, ffa/items
hud/Chat.tsx -> type:net/chat, net/chatCommand, screens/search
net/chat.ts -> net/chatCommand, type:net/protocol, net/session
net/client.ts -> type:arena/arena, combat, type:mode, modes, physics, simulation, vehicle/{drive,vehicles}, type:net/connection, net/{prediction,protocol,snapshots}
net/connection.ts -> type:loadout, net/protocol
net/matchmaking.ts -> type:loadout, net/{connection,protocol,session}
net/prediction.ts -> physics, vehicle/drive, type:net/protocol
net/protocol.ts -> combat, type:loadout, scoring, type:simulation, vehicle/vehicles
net/snapshots.ts -> net/protocol
screens/GameCanvas.tsx -> type:ai, audio, type:loading, type:loadout, maps, type:match, type:modes, runtime, net/chat, net/connection, settings, hud/{Chat,Hud}, screens/{Drawer,Menu,Results,SettingsPanel}
screens/Garage.tsx -> combat, loading, type:loadout, turntable, vehicle/drive, vehicle/vehicles, screens/Menu
screens/MapSelect.tsx -> ai, loading, maps, modes, net/matchmaking, screens/{Menu,search}
screens/Matchmaking.tsx -> audio, maps, modes, net/matchmaking, screens/{Menu,search}
screens/Results.tsx -> audio, maps, type:match, modes, type:simulation, tdm/config, screens/Menu
server/arenas.ts -> server/headless, type:arena/arena, geometry, maps
server/lobby.ts -> type:arena/arena, type:loadout, maps, modes, net/protocol, server/{matchmaker,room}, type:server/records
server/main.ts -> maps, physics, server/{arenas,server}
server/recorder.ts -> type:simulation, net/protocol
server/replay.ts -> type:arena/arena, type:maps, physics, server/room
server/rewind.ts -> combat, type:simulation, vehicle/vehicles
server/room.ts -> ai, combat, type:loadout, type:arena/arena, type:maps, modes, physics, roster, simulation, arena/digest, net/protocol, server/{arenas,fairplay,recorder,rewind}
server/server.ts -> type:arena/arena, type:maps, physics, rng, net/protocol, server/{auth,lobby,room,records}, type:server/matchmaker
```

Cycles: **none** (value or type).

## Appendix B: Reproducing the analysis

```bash
git clone --depth 1 https://github.com/aasumitro/bbmvc && cd bbmvc/game
npm ci
npm run lint && npx tsc -b && npm run check        # baseline (§0.3)
grep -rn "'tdm'\|'ffa'\|\.kind ===\|kind !==" src server --include=*.ts --include=*.tsx | grep -v check.ts   # mode leaks (§7)
grep -rn "Math.random\|performance.now\|localStorage\|window\.\|document\." src/game --include=*.ts | grep -v check.ts   # browser APIs (§3.3)
```

The import graph came from a ~60-line Node script. It parses `import … from`, `export … from` and side-effect imports, resolves relative specifiers to files, marks `import type`, and runs Tarjan's SCC for cycles. Dynamic `import()` (only `main.tsx → App`) was not counted. The same logic is the natural core of `boundaries.check.ts` (Phase 1).
