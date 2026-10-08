# Scrapyard `game/` Architecture Refactor Plan — Revision 2

> **PLAN ONLY.** Nothing in `aasumitro/bbmvc` was modified. This document is the blueprint for a later, incremental implementation.
>
> - **Revision 2 (2026-10-08)** analyses `main` at **`3b0943d`** (2026-10-07, "Custom lobbies: a list, invites, a waiting room, matches on the owner's settings (#4)": 100 files, +6,516 / −820 lines). **Revision 1** analysed `f28082e` (2026-09-30).
> - **Scope:** `game/src/` (the client) and `game/server/` (the authoritative game server), one npm package. The product in the code is **Scrapyard**; the repo's briefs call it BBMV.
> - **Method (revision 2):** read the whole custom-lobby commit, the repo's own plan and log for it (`.claude/work/custom/PLAN.md`, `LOG.md`), the updated `AGENTS.md` files and `.claude/work/arch/` notes; rebuilt the import graph over all 132 `.ts`/`.tsx` files; re-ran the full baseline (`lint`, `tsc -b`, `npm run check` including `server:check`) on a local clone, Linux x64, Node 22.22.2.

---

## Revision 2: what changed

### R2.1 What the custom-lobby commit added (facts)

| Area | What is new | Where |
|---|---|---|
| Lobby service | Lobbies, members, slots and sides, owners, bans, invite codes (40 bits, Crockford), passwords (scrypt + lockout), the tally, a 20 s grace, idle close, the public list. **Pure**: no sockets, rooms or timers; hooks for the rest (the pattern of `matchmaker.ts`) | `server/custom.ts` (499 lines), `server/custom.check.ts` (71 checks) |
| Wiring | `lobby.ts` runs custom lobbies beside Classic's matcher: a session uses one or the other; a lobby's match gets a room of its own on the members' lobby sockets; `MAX_LOBBIES` (24); `/health` counts custom rooms and lobbies | `server/lobby.ts`, `server.ts`, `main.ts` |
| Custom rooms | Room options `lobby`, `plan` (seat plan), `chat`, `over`, `idle`: bots only where the owner put them, empty seats, the end handed back to the lobby, abandoned after 10 s with nobody seated, a minute idle → back to the waiting room | `server/room.ts` |
| Match settings | One typed object for how a match is played: size, duration, respawn speed, friendly fire, pickup groups, weapons, kill limit; `classic(mode)` for Classic/practice; `CUSTOM` choices; `checkSettings` = the one validator for form and server | `game/matchSettings.ts` (102) |
| Pickups for any mode | The FFA item system became a module: catalogue, groups, a per-match `supply` (waves, expiry, effects), the token view; FFA keeps its hot zone (`ffa/zone.ts` view); TDM plugs the supply in when a lobby turns pickups on; `MatchMode.supply?` | `game/items/*`, `game/ffa/zone.ts` |
| Empty seats | `Combatant.present`; `Life` gains `absent` (and `LIVES` is now defined once in `mode.ts`); `rules.leave/enter`; `sim.vacate/occupy`; the car row's `absent` flag (Classic rows byte-identical); `Seat.present` | `simulation.ts`, `mode.ts`, rules, `protocol.ts`, `net/client.ts`, view, HUD |
| Friendly fire | The rules alone decide who is hurt (the simulation hands every hit over); a team kill costs the team a point; `Stats.teamKills`; bots hold fire near teammates (`Plan.careful`) | `simulation.ts`, `tdm/rules.ts`, `scoring.ts`, `ai.ts` |
| 12 machines | Six starts per team base on both arenas (digests now `227c4ce7` / `8913ad26`); 12 bot names; HUD pools of 12 | arenas, `roster.ts`, `Hud.tsx` |
| Wire | `PROTOCOL` 6; `lb` (18 actions) / `lbs` messages; lobby parsing in `parseClient`; `welcome.settings`, `welcome.lobby` | `net/protocol.ts` (477 → 604 lines) |
| Page | The custom-lobby store (list, waiting room, the match on the same socket, coming back within the grace, `?join=` links); `link(socket, welcome, aside)` + `release()`; screens `Custom`, `Lobbies`, `Lobby`, `LobbyForm`, `Avatar`, `Confirm` | `net/custom.ts` (274), `net/connection.ts`, `App.tsx`, `screens/*` |
| Checks | **Golden whole-match pins** (2 on the test yard, 4 on the real arenas), the custom lobby service, the page's store in `client.check`, many new cases elsewhere; `scripts/browser-match.mjs custom` | `simulation.check.ts`, `server.check.ts`, `custom.check.ts`, `client.check.ts` |

### R2.2 What changed in this plan

| # | Revision 1 said | Revision 2 says | Why |
|---|---|---|---|
| 1 | Phase 0: add golden fingerprints first | **Largely done by the repo.** Six whole-match pins exist; I re-ran the four real-arena pins on Linux x64 / Node 22.22.2 and got the hashes the repo's log recorded on macOS arm64 / Node 26.7 and 24.21. Phase 0 shrinks to the gaps: wire-byte fixtures and a pin for non-Classic settings | `server.check` output; `custom/LOG.md` phase 0 |
| 2 | Pickups across modes: postpone until a second mode wants them | **Done** (the owner wanted TDM pickups in custom lobbies). §6.5 rewritten | `game/items/` |
| 3 | "Custom" needs a product decision | **Decided and built:** a lobby flow plus parametrised matches (`MatchSettings`), not a third mode. §7.5 rewritten | `custom/PLAN.md` §1–2 |
| 4 | `Life` declared three times | **Fixed** (`LIVES` in `mode.ts`). `Point {x,z}` is still declared three times | `mode.ts`, `tdm/types.ts`, `items/items.ts` |
| 5 | One lint warning | **Fixed:** oxlint now reports 0 warnings | baseline |
| 6 | Mode knowledge in shared code: 7 lines in 3 files | **NEW, worse:** 34 lines in 7 shared files, two of them on the server (`server/custom.ts`, `game/matchSettings.ts`) | §7.2 |
| 7 | — | **NEW:** four rules are written twice or more across server and page: which side a slot is on (×3), when a lobby can start (×2), the tally key (×2), the biggest match's 12 seats (×5) | §1.5 |
| 8 | — | **NEW:** `room.ts` branches on the kind of room (`lobby`) at about nine points | §2, §10 |
| 9 | 0 import cycles, type imports included | **NEW:** value imports still have none, but one type-level cycle now spans 14 files. Two root edges | §3.4 |
| 10 | — | **NEW:** the two page stores (`net/matchmaking.ts`, `net/custom.ts`) duplicate the socket, hello and reconnect machinery. The repo's own review found most of the custom feature's bugs in this layer | §9 |
| 11 | `protocol.ts` cohesive, optional split | `protocol.ts` is now 604 lines, about a fifth of them (≈120) the lobby wire. **Split the lobby part out** (keep the file's path) | §9 |
| 12 | Unchanged | Weapon identity by turret model (now also behind the lobby's one-gun setting); the turret ternary; hard-coded HUD weapon icons; the wire-event codec split across server and page; the server ignoring the chosen vehicle (and the seat plan has no vehicle); `MatchMode.show(camera)` with scenery wired in `modes.ts` (now pickups and the hot zone); the game↔net folder inversion; no boundary enforcement | §0.2 |
| 13 | Phases 0–8, first five PRs | **Phases re-ordered and partly re-scoped; a new "mode traits" phase; first five PRs replaced** | §16, §22 |

This revision stands alone: everything from revision 1 that still holds is restated here with today's numbers, so there is no need to read the two side by side. Revision 1 stays in this file's git history (commit `d9d7020`).

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

### 0.1 The main finding still holds, and the custom-lobby commit is evidence both ways

The brief assumed the codebase needs restructuring into a layered architecture. Revision 1 found that most of the requested boundaries already exist in code (a refactor on 27 Sept 2026 split the old `match.ts`, introduced `MatchMode`, id-keyed registries, `SimEvents`, seeded randomness and `MatchSource`). The custom-lobby commit tests that finding with a large real feature:

- **Where the seams held.** Custom lobbies reuse the room, the simulation, the records, the replays, the fair-play watch and the page's whole online match (`online.ts`, `net/client.ts`) without forking any of them. The repo's plan states the rule ("one implementation of the game") and kept it. Classic was held unchanged through the refactor by whole-match hashes. This is the strongest evidence so far that the existing seams work, and that a big-bang move into a 9-layer tree would mostly relabel boundaries that hold. [High confidence: I read the diff and re-ran the pins.]
- **Where the seams leaked.** Where the existing contracts had no answer — *does this mode have teams? which sizes does it take? what are its Classic numbers? which side is slot 7 on? what kind of room is this?* — the new code asked directly: `mode === 'tdm'` (now in two server modules and three screens) and `lobby ? … : …` (about nine points in `room.ts`). Four rules now live in two or more places, on both sides of the wire.

So the recommendation sharpens rather than changes: **add the few small contracts the code keeps improvising (mode traits, shared lobby rules), enforce the boundaries mechanically, and do not big-bang the folders.**

### 0.2 What blocks plug-and-play now (priority order)

| # | Problem (evidence) | Status vs rev 1 | Verdict |
|---|---|---|---|
| 1 | **Mode knowledge outside the mode folders, and lobby rules written twice.** 34 lines naming `'tdm'`/`'ffa'` in 7 shared production files (rev 1: 7 lines in 3), including `server/custom.ts` (6: sides, slot claims, regrouping, start rules, tally) and `game/matchSettings.ts` (6: Classic numbers, sizes, friendly-fire validity). Which side a slot is on is computed in `tdm/mode.ts`, `server/custom.ts` and `screens/Lobby.tsx`; the start conditions in `server/custom.ts` and `Lobby.tsx` ("as the server holds it"); the tally key in `server/custom.ts` and `Results.tsx`. A third mode now edits about seven shared files, two of them on the server. | **NEW / worse** | **REFACTOR (Phase 3: mode traits + shared lobby rules)** |
| 2 | **Boundaries are enforced by convention only.** Unchanged, with more surface: `server/custom.ts` is "pure like `matchmaker.ts`" by comment; `audio.ts` still runs `window.addEventListener` at import. The repo's own review (`custom/LOG.md`, 2026-10-06) found that agent-written log entries claimed checks that did not exist ("client.check had no custom-lobby case at all … its '50 checks' never existed"). | Unchanged | **REFACTOR (Phase 1)** |
| 3 | **Weapon identity is recovered from the turret model** (`net/protocol.ts:254`). Bots carry a scaled copy of the spec, and now the lobby's one-gun setting re-fits people too. Two weapons sharing a turret model would be misreported on the wire, in records and in replays. | Unchanged | **REFACTOR (Phase 2)** |
| 4 | **Adding a weapon edits core files:** the turret ternary (`vehicle/vehicle.ts:195`), two hard-coded HUD icons (`hud/Hud.tsx:237–243`), the closed `model` union. (Good news: `CUSTOM.weapons` derives from `WEAPONS`, so the lobby form offers a new gun automatically.) | Unchanged | **REFACTOR (Phase 2)** |
| 5 | **A type-level import cycle across 14 files**, from two root edges: the `Mode` id type is derived from the `MODES` registry object (so `matchSettings.ts` imports the registry's module for a type), and `mode.ts` (the contract) imports `Supply` from `items/supply.ts`, which imports `mode.ts` back. Erased at runtime, but the ids sit at the top of the graph instead of the bottom. | **NEW** | **REFACTOR (Phase 2, type-only)** |
| 6 | **The wire-event codec is split** (`server/recorder.ts` encodes, `net/client.ts` decodes with positional `f[n]`). | Unchanged | **REFACTOR (Phase 4)** |
| 7 | **Mode UI and scenery:** `Hud.tsx`/`Results.tsx` still narrow on `mode.kind`; `MatchMode.show(camera)` remains; `modes.ts` wires `createPickups` and `createHotZone`, so the server bundle carries both views. The HUD's effect chips and minimap items now read `mode.supply` generically, which is better. | Partly better | **REFACTOR (Phase 5)** |
| 8 | **`room.ts` branches on the room's kind** (`lobby`): seat choice, the gun, leaving, results → end or restart, abandonment, idle, `open()`, record and journal fields. | **NEW** | **REVIEW** now; **REFACTOR** into a small room-kind object only when a third kind (ranked, tournament, spectating) is planned |
| 9 | **Two page stores duplicate the session-socket machinery** (dial + hello, retry schedule, `comeBack` loop, `sessionStorage` mark, attempt counters, welcome → link). | **NEW** | **REFACTOR, small (Phase 7):** shared helpers, two stores |
| 10 | **Characterization gaps:** whole-match pins exist; wire bytes are not pinned; no pin covers non-Classic settings (12 seats, friendly fire, TDM pickups, kill limit, an empty seat). | Mostly done | **Phase 0 (test-only)** |
| 11 | **A second vehicle is a feature:** the server ignores the chosen vehicle (`server/server.check.ts:368` fails on purpose if `VEHICLES` gets a second entry), bots use `BOT_VEHICLE`, the `SeatPlan` has no vehicle, `ro` and the journal's `join` carry no vehicle. | Unchanged (+ seat plan) | **Postpone (Phase 8)** |
| 12 | **`src/game/` mixes five concerns** (now 71 files); `game/runtime.ts`/`online.ts` import `net/` while `net/` imports `game/`. | Unchanged | **REFACTOR, mechanical (Phase 6)** |

### 0.3 Baseline (re-measured at `3b0943d`, Linux x64, Node 22.22.2)

| Command | Result |
|---|---|
| `npm run lint` | pass, **0 warnings** (oxlint 1.85) |
| `npx tsc -b` | clean |
| `npm run check` | **all pass:** router ok · bots 21 (kills a minute: easy 7.8, normal 13.8, hard 17.8) · ffa 152 · tdm 182 · simulation 58 · loading 23 · protocol 90 · chat 14 · matchmaker 63 · custom 71 · fairplay 10 · arena 72 (digests `227c4ce7` / `8913ad26`) · server 166 · client 47 · netplay 33 |
| Golden pins (real arenas) | `tdm scrapyard d9ee9ad8fa7017af`, `tdm city 467400f8225aeea3`, `ffa scrapyard 3c2735848552d2b4`, `ffa city 6c4c3343a80b5f99`: **identical** to the pins recorded on macOS arm64. Cross-machine stability, which revision 1 marked "needs verification", is now shown on two platforms and three Node majors. |
| Server bundle | main shared chunk 4,435 kB (unchanged: Three.js and every visual arena builder ride along) |
| Import cycles | value imports: **0**; type-level: **1** strongly connected set of 14 files (§3.4) |
| Code size | about 22.0k lines of production TS/TSX (rev 1: 19.2k) and about 4.9k lines of checks (rev 1: 3.9k); 132 files (rev 1: 119) |
| CI on GitHub | not verified by me: this session cannot read the repo's check runs. My local run is the evidence. |

The repo's own log records one known flaky check: `netplay.check`'s rough-link case (a wall-clock timing case) failed on a loaded M1 Pro in several runs and passed in others (`custom/LOG.md`, phases 0, 2 and 4). It is not caused by the custom work; see §13.4.

### 0.4 Labels used

- **KEEP**: sound as it is; no refactor needed.
- **REVIEW**: questionable but not clearly harmful; investigate before changing.
- **REFACTOR**: change it, for the concrete reason given.
- **[High / Medium / Low confidence]**: based on what I read or measured. "Needs verification" means not proven.

---

## 1. Current architecture

### 1.1 Folder overview

```
game/
├── src/
│   ├── main.tsx, App.tsx, analytics.ts, index.css   app shell: screens, loadout, pick, online seat; now also the custom store's
│   │                                                 seat, the ?join= link and resuming a lobby after a reload
│   ├── screens/   (19 files)  React screens; new: Custom (the arena screen's Custom entry), Lobbies (the list), Lobby (the
│   │                          waiting room, 415 lines), LobbyForm (create/edit drawer), Avatar, Confirm (moved out of GameCanvas)
│   ├── hud/       (3 files)   Hud.tsx (per-frame imperative DOM writes), minimap.ts, Chat.tsx (now also docked in the waiting room)
│   ├── game/      (71 files)  everything else about the game; new: matchSettings.ts, items/ (config, items, supply, pickups), ffa/zone.ts
│   └── net/       (10 files + 2 checks)  protocol, socket, client mirror, prediction, interpolation, two page stores
│                              (matchmaking.ts for Classic, custom.ts for lobbies), Nakama session, chat
└── server/        (~17 production files + checks)  door + loop, lobby (rooms, sessions, both flows), matchmaker (pure),
                               custom (pure), room, recorder, rewind, fair-play, records, replay, headless arenas, auth
```

### 1.2 What each area owns today

| Area | Owns | Assessment |
|---|---|---|
| `src/` root | `App.tsx` (130 lines): screen state, `Loadout`, the pick, the online `Link` from either store; `?join=` handling; resuming a search or a lobby after a reload | **KEEP**; it is the one place that knows both stores, which is right |
| `src/game/` sim part | `simulation.ts` (431: + empty seats, every hit to the rules), `combat.ts`, `physics.ts`, `vehicle/drive.ts`, `ai.ts` (581: + careful bots, a weapon roster for `armBot`), `scoring.ts` (+ teammate path, `teamKills`), `rng.ts`, `mode.ts` (contract + `LIVES` + `ms`/`clock`), `roster.ts` (+ seat plans, sizes, settings), `loadout.ts`, **`matchSettings.ts`** | **KEEP** the design; `matchSettings.ts` mixes per-mode knowledge into a shared module (§7) |
| `src/game/` modes | `ffa/`, `tdm/` (rules take `settings`; TDM gains friendly fire, team kills, a seeded stream, an optional supply), `modes.ts` (registry: `lineUp(arena, size)`, `create({…, settings})`), **`items/`** (the supply any mode plugs in) | **KEEP** the design; leaks listed in §7.2 |
| `src/game/` content | arenas (bases now 6 a side), vehicles, materials, `maps.ts` | **KEEP**; weapon registration leaks unchanged (§6.2) |
| `src/game/` presentation | `view.ts` (+ hides absent machines), effects, audio, camera, `feed.ts` (+ team kills), pilot, input, settings, turntable, renderer, environment, post-processing; **`items/pickups.ts`, `ffa/zone.ts`** (scenery) | **KEEP** |
| `src/game/` runtime | `runtime.ts`, `match.ts` (practice plays `classic(mode)`), `online.ts`, `loading.ts` | **KEEP** the design; **REFACTOR** the location |
| `src/net/` | `protocol.ts` (604: + lobby messages, `PROTOCOL` 6), `connection.ts` (+ `aside`, `release`), `client.ts` (+ empty seats), `prediction.ts`, `snapshots.ts`, `matchmaking.ts`, **`custom.ts`**, `session.ts`, `chat.ts` (+ notes) | **KEEP** the boundaries; split the lobby protocol out (§9); share the store helpers (§9) |
| `src/screens/` | + the custom-lobby screens; `Results.tsx` (346: + the lobby's tally, Back to lobby); `GameCanvas.tsx` (345: Confirm moved out, custom labels); `MapSelect.tsx` (Custom enabled) | **KEEP** the structure; **REVIEW** `Lobby.tsx` size and its copy of server rules |
| `src/hud/` | `Hud.tsx` (728): pools of 12, a TK column, effect chips and minimap items from `mode.supply` | **REFACTOR** into per-mode panels (§7) |
| `server/` | `custom.ts` (499, pure) beside `matchmaker.ts`; `lobby.ts` (316) wires both flows; `room.ts` (613) hosts Classic and custom rooms; `server.ts` (321) keeps a session through a custom match; `rewind.ts` skips absent machines; `replay.ts` seats a custom room's people where the journal says | **KEEP** flat; light extractions (§10) |

### 1.3 The custom-lobby flow (new)

```
page                                                       game server (one process)
 MapSelect ─ Custom.tsx ─┬─ Lobbies.tsx (list, join by code)
                         └─ Lobby.tsx (waiting room, chat docked)
            ▲ reads                                        server.ts  (door: hello with no map = a session)
 net/custom.ts store ── socket: hello, {t:'lb', do} ─────►   lobby.ts ──► custom.ts (pure: lobbies, slots, owners,
            │           ◄── {t:'lbs'} list, {t:'lb'} lobby       │          codes, passwords, tally, grace, idle)
            │                                                    │  hooks.start(lobby, seat plan)
            │                                                    ▼
            │           ◄── welcome (settings, lobby) ────── room.ts (lobby, plan, chat, over, idle; empty seats)
 connection.link(socket, welcome, aside) ── match: in / s / st / ro ── same simulation, recorder, rewind, fair play
            │   lobby words during the match go to `aside`        │
 App ─► GameCanvas ─► runtime ─► online.ts                       │ results over → over(winner) → lobby.ts finish()
            │                                                    │   → release() the members' sockets → custom.over()
 store: link.release() ◄── {t:'lb'} lobby (playing: false) ◄─────┘   → tally, everyone unready, waiting again
 App ─► MapSelect (Custom entry: the waiting room)
```

A drop or a reload comes back within the 20 s grace (`back`); a minute without input in a custom match takes the person back to the waiting room on the same socket (Classic would hand the seat to a bot).

### 1.4 Good decisions (KEEP; do not undo)

Carried over from revision 1 and re-checked at `3b0943d`:

1. **React is out of the loop.** `renderer.setAnimationLoop(frame)` drives `match.frame`, and the HUD writes the DOM through an imperative handle. React re-renders only on loading steps and player-view phase changes (`GAME_LOOP.md`; verified in `runtime.ts` and `GameCanvas.tsx`).
2. **One simulation for practice, server and replay.** `createSimulation` is called by practice (`match.ts`), by the room (`server/room.ts`) and by the replay (through `createRoom`). `server.check` holds that "a room is the practice simulation, nothing more", and now also that a custom room replays to the bit.
3. **`SimEvents` as a direct-call interface, not a bus.** Headless runs pass no-ops, the server passes the recorder, the browser passes the view.
4. **Controls are plain data written by interchangeable control sources:** the pilot, a bot brain, a server input queue, a replay's `forced` input.
5. **Pure, event-queue mode rules** (152 and 182 checks) behind an adapter (`MatchMode`). `share()`/`mirror()` carry rules state online without the browser ever ticking rules.
6. **Seeded randomness with separate streams:** simulation `seed ^ 0x9e3779b9`, bot guns `seed ^ 0x2545f491` (`roster.ts`), FFA rules `createRng(seed)`, and (new) TDM rules `createRng(seed)`. The TDM stream is drawn only by the supply when a lobby turns pickups on, so Classic TDM's draws did not move. Presentation (particles, pitch, shake) is deliberately unseeded.
7. **`MatchSource`:** the player's side (`playMatch`) is identical for practice and online; only the step source differs. Custom matches needed no third source.
8. **Protocol discipline:** quantised integers, binary snapshot frames, `PROTOCOL` bumps (now 6), a content-hash `BUILD` id, server-side `parseClient` clamping. The client sends intent only.
9. **Arena digests** held across browser, server, CI and the deploy smoke test.
10. **The `.check.ts` convention:** plain-node self-checks with no test framework, plus Vite-bundled server checks over real sockets.
11. **Session caches with explicit ownership** (`STATE_OWNERSHIP.md`): materials, baked textures, arenas and sound buffers live for the session; geometries are disposed per match.

Added by the custom work:

12. **`server/custom.ts` is pure with hooks**, exactly like `matchmaker.ts`: the clock injected (`LobbyHooks.now`), no sockets, rooms or timers; 71 plain-node checks drive it on a hand-moved clock.
13. **One validator for match settings** (`checkSettings`): the drawer shows its errors, the server runs it again and trusts nothing else. It returns a fresh object, so no unnamed field gets through.
14. **Settings are data that travel with the match:** the welcome, the replay header (with the lobby and the seat plan) and the match record carry them, so a custom room replays to the bit (`server.check`).
15. **Empty seats keep every per-seat array's length** (snapshots, rewind, fair play, statistics). The machine is out of play (`present` false, body disabled), so no index anywhere shifts.
16. **Classic's bytes did not move:** the `absent` flag is set only for an empty seat, so a Classic car row is the same bytes as before (`protocol.check` holds it); `classic(mode)` reproduces Classic's numbers (the pins hold it).
17. **Pins were re-pinned on purpose, with reasons** (the new `teamKills` field; the new bases), and only after showing that the old hashes still matched with the new field left out. This is the right discipline; keep it.
18. **Secrets stay secret:** codes and passwords are never logged (`server.check` asserts it); the list carries no code, password or user id; passwords are scrypt with a salt, compared in constant time, behind a lockout.
19. **Custom rooms never leak into Classic:** `room.open()` is false for them, so neither the matcher nor a direct seat lands there; rooms are counted by kind on `/health`.

### 1.5 Problem decisions (REFACTOR or REVIEW)

| Problem | Evidence | Verdict |
|---|---|---|
| Mode knowledge in shared modules | 34 `'tdm'`/`'ffa'` lines in 7 shared files: `matchSettings.ts` 6, `server/custom.ts` 6, `screens/LobbyForm.tsx` 8, `screens/Lobby.tsx` 7, `hud/Hud.tsx` 4, `screens/Results.tsx` 2, `server/room.ts` 1 (+ data lists in `maps.ts` and `App.tsx`'s default pick, which are fine) | **REFACTOR** (Phase 3) |
| Rules written twice or more across server and page | side of a slot: `tdm/mode.ts lineUp`, `server/custom.ts side()`, `screens/Lobby.tsx side()`; start conditions: `custom.ts unstartable()`, `Lobby.tsx startHint()`; tally key: `custom.ts over()`, `Results.tsx winnerKey()`; the biggest match's 12 seats: `Hud.tsx MARKERS`, `SCORE_ROWS`, `protocol.ts SLOTS`, `roster.ts BOT_NAMES` length, `CUSTOM.sizes` | **REFACTOR** (Phase 3). The server stays authoritative, so a drift only misleads the page (a wrong hint, a wrong tally highlight), but it will drift. |
| Room-kind branching | `server/room.ts`: `lobby ? at : freeSeat()`, the gun, `leave` (empty seat vs bot), results → `end` vs `restart`, abandonment, idle, `open()`, record, journal header; also `lobby.ts` (rooms closing rule, `unseat`) and `server.ts` (keeping the session) | **REVIEW** (refactor with a third kind) |
| Type-level cycle | `matchSettings.ts → type modes.ts`; `mode.ts ↔ items/supply.ts` | **REFACTOR** (Phase 2) |
| Wire module depends on the AI module | `protocol.ts → matchSettings.ts → ai.ts` (for `DIFFICULTIES` keys): the wire format's import closure now holds the bot AI and Rapier, also in the plain-node `custom.check` | **REFACTOR** (Phase 2: difficulties as a leaf data module) |
| Two page stores, same machinery | `net/matchmaking.ts` and `net/custom.ts`: dial, `RETRY`, `comeBack`, `sessionStorage` mark, `attempt`, welcome → link | **REFACTOR, small** (Phase 7) |
| `protocol.ts` growth | 604 lines; lobby types, `LOBBY_ACTIONS`, `INVITE`, `readCode`, `tidy`, `LOBBY_FORM`, `lobbying()` | **REFACTOR** (Phase 4): `net/lobbyProtocol.ts`; `protocol.ts` keeps `PROTOCOL` and its path |
| `screens/Lobby.tsx` size | 415 lines: slots grid, owner menus, the bar and its hint, settings card, invite, tally, chat, keys | **REVIEW**: split when next touched (Phase 7) |
| Weapon identity by turret model (unchanged) | `net/protocol.ts:254` (`weaponId`); `ai.ts:73` (`botGun` copies the spec); a custom seat is armed with `WEAPONS[gun]` and the lobby's one-gun setting goes through the same lookup | **REFACTOR** (Phase 2) |
| Closed-set turret and icon selection (unchanged) | `vehicle/vehicle.ts:195` (`weapon === 'rocketPod' ? … : …`); `hud/Hud.tsx:237–243` (two inline SVGs); the closed `model` union in `combat.ts` | **REFACTOR** (Phase 2) |
| Rendering in the gameplay contract (unchanged, wider) | `game/mode.ts:89` `show(camera)`; `modes.ts:37, 44` inject `createPickups(scene)` and `createHotZone(scene)`, so the **server bundle includes both views** | **REFACTOR** (Phase 5) |
| Wire-event codec split; positional indices (unchanged) | `server/recorder.ts` encodes ↔ `net/client.ts` `play()`/`mine()` decode with `f[n]` | **REFACTOR** (Phase 4) |
| Duplicate domain types (partly fixed) | `Life` is now defined once (`LIVES` in `mode.ts`); `Point {x,z}` ×3 (`mode.ts:28`, `tdm/types.ts:8`, `items/items.ts:35`); two unrelated `Seat` types (`mode.ts:60`, `net/protocol.ts:110`) | **REFACTOR** (type-only, Phase 2) |
| Loadout validation duplicated (unchanged) | `loadout.ts savedLoadout()` vs `protocol.ts pick()` (line 578) | **REFACTOR, small** (Phase 2) |
| game↔net folder inversion (unchanged) | `game/runtime.ts → net/connection.ts`, `game/online.ts → net/client.ts`, while `net/client.ts → game/*` | **REFACTOR** (move, Phase 6) |
| Arena gameplay layout derived from the visual scene graph (unchanged) | `props.ts solid()` stores colliders in `userData`; `arena.ts collectColliders()` reads `matrixWorld`; the server builds the visual arena under a DOM shim, then `strip()`s it | **REVIEW → postpone** (digests guard it; §18) |
| View reads Rapier internals (unchanged) | `view.ts` `wheelIsInContact` (lines 221, 237), `poseWheels` (suspension from the controller) | **REVIEW** (documented debt; fine while clients run a local world) |
| `Garage.tsx → vehicle/drive.ts` for `drivePerformance` (unchanged) | `screens/Garage.tsx:6`: pure maths inside a Rapier module | **REVIEW** (harmless; move if convenient) |
| Unused Vitest dev dependency (unchanged) | `package.json`; `AGENTS.md` says "installed, unused" | **REVIEW** (owner decision) |
| Legacy generators kept as reference (unchanged) | `VehicleGenerator.ts`, `ArenaGenerator.ts`, `proceduralTexture.ts` (no importers outside the three) | **REVIEW** (owner decision; `AGENTS.md` says keep) |

---

## 2. God objects

Size alone is not a defect: `city.ts` (836 lines), `recipes.ts` (690) and `scrapyard.ts` (640) are big, single-purpose content files. The files below were judged on **responsibilities**. The summary comes first; each file then gets the brief's format (current responsibilities, what should remain, what should move, destination, reason, risk). Phases refer to §16.

### 2.1 Summary

| File | Lines (rev 1 → rev 2) | Verdict | In one line |
|---|---|---|---|
| `game/match.ts` | 318 → 320 | **KEEP** (one small split) | `createMatch` → `runtime/practice.ts` (Phase 7) |
| `game/simulation.ts` | 395 → 431 | **KEEP** | Empty seats landed cleanly; highest determinism risk if touched |
| `game/runtime.ts` | 160 → 160 | **KEEP** (move folder) | Unchanged by the custom work |
| `game/ai.ts` | 556 → 581 | **REFACTOR** (Phases 2, 7) | `DIFFICULTIES` to a leaf (the wire imports it); split by responsibility later |
| `game/view.ts` | 291 → 293 | **KEEP** | Weapon cue becomes data (Phase 2) |
| `game/matchSettings.ts` | new, 102 | **KEEP** (Phase 3 edits) | Per-mode facts move to traits |
| `ffa/rules.ts`, `tdm/rules.ts` | 536 → 536, 335 → 380 | **KEEP** | One `Point` import |
| `server/server.ts` | 310 → 321 | **KEEP** | Optional `netsim.ts` |
| `server/room.ts` | 567 → 613 | **REFACTOR, light** (Phases 3, 7) | `journal.ts`, `inputs.ts`; group the custom options; no `RoomKind` yet |
| `server/lobby.ts` | 224 → 316 | **REVIEW** | Two flows, one job; extract only with a third flow |
| `server/matchmaker.ts` | 278 → 278 | **KEEP** | |
| `server/custom.ts` | new, 499 | **KEEP** (Phase 3 edits) | Its mode branches and two rules move to traits and `lobbyRules.ts` |
| `hud/Hud.tsx` | 723 → 728 | **REFACTOR** (Phases 2, 3, 5) | Per-mode panels; pools from `MAX_SEATS`; icon from data |
| `screens/GameCanvas.tsx` | 391 → 345 | **REVIEW** | Shrank; optional component split |
| `screens/Results.tsx` | 319 → 346 | **REFACTOR** (Phases 3, 5) | Per-mode panels; `winnerKey` → shared tally key |
| `screens/Lobby.tsx` | new, 415 | **REVIEW** (Phases 3, 7) | Two server rules re-implemented; split when next changed |
| `net/client.ts` | 420 → 432 | **REFACTOR, light** (Phase 4) | Decode through the codec |
| `net/protocol.ts` | 477 → 604 | **REFACTOR** (Phases 2, 3, 4) | Lobby wire out; `weaponId`; keep the path |
| `net/custom.ts` | new, 274 | **REFACTOR, small** (Phase 7) | Share dial/retry/mark helpers with `matchmaking.ts` |

### `game/match.ts` (320 lines): **KEEP** (one small split)

| | |
|---|---|
| **Current responsibilities** | `playMatch`: the player's side of any match (pilot wiring, the fixed-step accumulator, interpolation, the player-view phase machine, pause/restart/lose, the HUD-facing state object, debug lines). `createMatch`: the practice `MatchSource` (seeds, roster, world, view, mode, simulation, feed); practice now plays `classic(mode)`. Types: `MatchSource`, `MatchParts`, `Match`. |
| **Should remain** | `playMatch`, `MatchSource`, `Match`. |
| **Should move** | `createMatch`. |
| **Suggested destination** | `runtime/match.ts` + `runtime/practice.ts`, symmetric with `runtime/online.ts` |
| **Reason** | Clarity: the two match sources would sit side by side. No behavioural reason. |
| **Risk** | Low. A pure move; the seed pick (`freshSeed`, `Math.random`, line 34) moves with it. |

### `game/simulation.ts` (431 lines): **KEEP**

| | |
|---|---|
| **Current responsibilities** | Combatants (`enlist`, `reset`); one fixed step (think → drive → fire → `world.step` → read poses → crash → stuck → rockets → rules tick → respawn); hitscan (with the server's optional `castRound` hook); rockets and blasts; damage (every hit is handed to the rules, which decide who is hurt: friendly fire); wrecks; respawn via the rules; stuck recovery; `standDown`; `restart`. **New:** empty seats (`vacate`/`occupy`, `present` checks in every loop, the body taken out of the world). |
| **Should remain** | All of it. It is the authoritative core, and its step order *is* the game's determinism. |
| **Should move** | Nothing now. **Only if** a third firing behaviour (beam, homing, mine) is scheduled: turn `if (c.weapon.spec.rocket) return launch(c)` (line 281) into a small table keyed by `spec.kind`. |
| **Suggested destination** | `sim/simulation.ts` (folder move only, Phase 6) |
| **Reason** | Cohesive and well checked (58 checks plus the pins). The empty-seat change landed as a few small functions (`vacate`, `occupy` and their helpers) and `present` checks, not a fork. |
| **Risk** | High if touched: step order, RNG draw order, Rapier call order (§18). |

### `game/runtime.ts` (160 lines): **KEEP** (move folder)

| | |
|---|---|
| **Current responsibilities** | Match loading tasks, renderer settings, scene/sky/sun, arena borrow, practice-or-online match creation, the composer and live settings, the frame, resize, release, dev globals, the online arena/digest validation. |
| **Should remain** | All of it: this is the application-level composition. |
| **Should move** | The folder only. The arena session cache `loadArena` could move in from `maps.ts`. |
| **Suggested destination** | `runtime/runtime.ts` (Phase 6), which removes the `game → net` inversion |
| **Reason** | Folder honesty: it composes `net/`, so it cannot sit below it. |
| **Risk** | Low. |

### `game/ai.ts` (581 lines): **REFACTOR** (data out in Phase 2; split in Phase 7)

| | |
|---|---|
| **Current responsibilities** | (1) Tuning (`AI`, `TACTICS`). (2) Difficulty (`Skill`, `DIFFICULTIES`, `Difficulty`, `WEAPON_IDS`, `armBot` with a weapon roster for the lobby's one-gun setting, `botGun`). (3) The agent model (`Agent`, `Brain`, `Plan` with the new `careful` flag, `createBrain`, `provoke`). (4) Perception (`feel`, `canSee`, `clearOfMates` (new), `open`, `hiddenFrom`, `pickHideout`, `nodesInView`). (5) Navigation (`routesTo`, `travel`). (6) Targeting (`pickTarget`, `leadTarget`). (7) The decision loop (`think`). |
| **Should remain together** | `think` + targeting (they share per-call scratch state). |
| **Should move** | Phase 2: `Skill`, `DIFFICULTIES`, `Difficulty` → a leaf data module (no imports). Phase 7: the rest by responsibility. |
| **Suggested destination** | `sim/difficulty.ts` (Phase 2); `sim/ai/skill.ts` (`armBot`, `botGun`, `WEAPON_IDS`), `sim/ai/brain.ts` (`Agent`, `Brain`, `Plan`, `createBrain`, `provoke`), `sim/ai/perception.ts`, `sim/ai/navigation.ts`, `sim/ai/think.ts` (+ targeting) (Phase 7) |
| **Reason** | Seven responsibilities in one file; per-weapon tuning and new behaviours land here. **New:** the wire format depends on this module only for the difficulty names (`protocol.ts → matchSettings.ts → ai.ts`), so the wire's import closure holds the bot AI and Rapier, also in the plain-node `custom.check`. |
| **Risk** | Low for the data move. **Medium-high for the split:** (a) the order of `random()` draws must not change; (b) the module-level scratch vectors (`goal`, `circling`, `weaving`, `lead`, `aimAt`, `from`, `along`, `spot`, `errandAt`, `toAim`, `toLead`) must not become *shared* between functions that are live at the same time. `bots.check` (behaviour ranges) and the golden pins catch both. |

### `game/view.ts` (293 lines): **KEEP**

Cohesive. It builds models, implements `SimEvents` (effects, sound, shake, feedback pulses), interpolates, lays turrets, runs ambient effects and engine audio, `refit`s, restarts and disposes; **new:** it hides an absent machine (no model, smoke, fire or burning loop). Only change: the fire cue is chosen by `c.weapon.spec.rocket ? 'launch' : 'shot'` (line 114); that becomes `spec.cue ?? …` when a weapon needs its own sound (Phase 2, §6.2).

### `game/matchSettings.ts` (102 lines, new): **KEEP** (Phase 2 and Phase 3 edits)

| | |
|---|---|
| **Current responsibilities** | `MatchSettings` (size, duration, respawn, friendly fire, item groups, weapons, kill limit); `classic(mode)`; `RESPAWN` shares and `respawnWait`; `CUSTOM` (the owner's choices: modes, sizes per mode, durations, …); labels (`RESPAWN_LABELS`, `ITEM_GROUPS`, `weaponsLabel`, `killLimitLabel`, `minutes`); `checkSettings`, the one validator. |
| **Should remain** | All of it, as the settings module (labels next to their data, as `VehicleSpec.blurb` does). |
| **Should move** | The per-mode facts (Classic's size and duration, pickups in Classic, which sizes a mode takes, whether friendly fire applies) → traits (Phase 3); the `DIFFICULTIES` import → the leaf (Phase 2); `type Mode` from the leaf ids instead of the registry (Phase 2). |
| **Suggested destination** | `modes/settings.ts` (Phase 6) |
| **Reason** | Six `'tdm'`/`'ffa'` lines; it is one root of the type cycle and the reason the wire imports the AI. |
| **Risk** | Low: `protocol.check` holds the validator and the pins hold `classic()`. |

### `ffa/rules.ts` (536) and `tdm/rules.ts` (380): **KEEP**

Pure, event-sourced, heavily checked (152 and 182 checks). The custom work (settings, friendly fire, team kills, the kill limit, the supply) landed inside them without leaking out. Only change: import `Point` from one place (Phase 2).

### `server/server.ts` (321 lines): **KEEP** (optional extraction)

| | |
|---|---|
| **Current responsibilities** | HTTP `/health` (rooms counted by kind, lobbies); the upgrade door (path, origin, per-address cap); per-socket protocol (hello deadline, version/build check, auth, rate tokens, strikes, backlog cut-off); routing a hello to the lobby (a session: Classic search or custom lobby) or to a room; **new:** keeping a session through a custom match (`release` hands the socket back); the **dev latency simulator** (`held()`, `stallEnd`: lag, jitter, stalls, order-preserving); the fixed-step loop with its catch-up cap (`CATCH_UP` 5); shutdown. |
| **Should remain** | Everything except the latency simulator. |
| **Should move (optional)** | `held()` + `stallEnd`. |
| **Suggested destination** | `server/netsim.ts` (Phase 7) |
| **Reason** | Separates a development tool from the production door. Low value; do it only when touching the file anyway. |
| **Risk** | Low; `netplay.check` exercises it. |

### `server/room.ts` (613 lines): **REFACTOR, light** (Phases 3 and 7)

| | |
|---|---|
| **Current responsibilities** | Seats and people (`join`, `leave`, `freeSeat`, `takeWheel`); the **input queue policy** (queue, drain window, stale/idle, repeats/drops/depth statistics); applying `Given` inputs; the **replay journal format** (`ReplayLine`, `Given`, `DRIVING/STANDING/COASTING`, `note()` row encoding) that `replay.ts` mirrors; the **fair-play watch** geometry; the first-match hold; the step orchestration; the **match record** (`MatchRecord`, `SeatRecord`, `matchRecord()`, now with `custom`); snapshot/state broadcast; chat channel names (team channels branch on `kind !== 'tdm'`, line 236); restart. **New, custom rooms:** a seat plan (bots only where the owner put them, empty seats vacated at once), the lobby's chat channel, the person's own gun (no bot copy), leaving empties the seat instead of handing it to a bot, results → `end` (back to the lobby) instead of `restart`, abandonment after `ABANDON` (10 s) with nobody seated, the `idle` hook (a minute without input → back to the waiting room), `open()` false, `custom` in the record and `lobby`/`plan` in the journal header. |
| **Should remain** | `step()` orchestration **in its exact order**; seats; broadcast; lifecycle; the custom-room branches, grouped. |
| **Should move** | The journal format; the input queue policy; optionally the record builder. Group the five custom options (`lobby`, `plan`, `chat`, `over`, `idle`) into one `custom?: {…}` option; team chat from the `sides` trait (Phase 3). |
| **Suggested destination** | `server/journal.ts` (shared by `room.ts` and `replay.ts`), `server/inputs.ts` (pure, with its own check), optional `server/matchRecord.ts` (Phase 7) |
| **Reason** | The journal format is a persisted, versioned artefact inside a 613-line file, and the input policy is the most tuned code on the server (`NET_LOG.md`). The custom options are one concept spread over five optional parameters. **Do not** introduce a room-kind object for two kinds (§10). |
| **Risk** | **Medium-high:** replay determinism and the journal's line semantics (`ahead()` in `replay.ts`). Guarded by `server.check`'s replay cases (Classic and custom) and the pins. |

### `server/lobby.ts` (316 lines): **REVIEW**

| | |
|---|---|
| **Current responsibilities** | Room lifecycle and seating for Classic (the matcher's hooks: rooms, ready checks, backfill, the drop grace); **new:** custom lobbies beside it (the `createLobbies` hooks: a room per lobby match on the members' lobby sockets, `place`, `finish`, `unseat`, `release` at the end), the one-flow-per-session rule, `MAX_LOBBIES`. |
| **Should remain** | Both flows' wiring. It is still one job: who goes where. |
| **Should move (only with a third flow)** | The `createLobbies` hooks (≈35 lines) and their `place`/`finish` helpers. |
| **Suggested destination** | `server/customRooms.ts` |
| **Reason** | Readability only. |
| **Risk** | Low–medium: socket ownership at the lobby ↔ match boundary (§18). |

### `server/matchmaker.ts` (278 lines): **KEEP**

Pure, clock-injected, 63 checks. Cohesive. Unchanged by the custom work.

### `server/custom.ts` (499 lines, new): **KEEP** (Phase 3 edits)

| | |
|---|---|
| **Current responsibilities** | The custom-lobby domain: lobbies, members, slots and sides, owners and the hand-off, kicks and bans, invite codes (40 bits, Crockford), passwords (scrypt + salt, constant-time compare, lockout), the tally, the 20 s grace, idle close, the public list (no secrets), caps. Pure: the clock and every effect go through `LobbyHooks`. |
| **Should remain** | All of it. |
| **Should move** | Its six mode branches → traits (`sides`, `sideOf`); `unstartable` → `lobbyRules.startable`; the tally key in `over()` → `lobbyRules.tallyKey`. |
| **Suggested destination** | `modes/traits.ts`, `net/lobbyRules.ts` (Phase 3); after Phase 4 it imports `net/lobbyProtocol.ts` |
| **Reason** | The page re-implements three of these rules (§1.5); one home removes the drift. |
| **Risk** | Low–medium: regrouping on a mode change and the tally keys are subtle; `custom.check` (71) drives them. |

### `hud/Hud.tsx` (728 lines): **REFACTOR** (Phases 2, 3, 5; optional 7)

| | |
|---|---|
| **Current responsibilities** | All HUD markup; element collection by `data-hud`; the per-frame update for compass, minimap, **FFA score/board/lead line**, **TDM team score/momentum**, clock/overtime (branching per mode), banner/countdown, feed, speed/health, **effect chips** (`EFFECTS`, a hard-coded list, read from `mode.supply`), weapon panel (**hard-coded icons**, lines 237–243), crosshair, markers (`MARKERS = 12`), hints, scoreboard (**mode-specific ordering, headers, labels**, a TK column, `SCORE_ROWS = 12`), debug; leaves absent machines out. |
| **Should remain** | The shared HUD (compass, clock, banner, feed, vitals, weapon, crosshair, markers, debug) and the imperative-write pattern. |
| **Should move** | Mode panels; the weapon icon (to weapon data, Phase 2); the chip list (from `ITEMS`, Phase 5); the pool sizes (from `MAX_SEATS`, Phase 3); optionally the shared parts into helpers (Phase 7). |
| **Suggested destination** | `modes/<mode>/hud.tsx` via the client-only `MODE_VIEWS` registry (Phase 5); optional `hud/{compass,vitals,weapon,feed,markers,scoreboard}.ts` |
| **Reason** | This is the file a third mode or a new weapon must edit today. |
| **Risk** | Medium: per-frame allocations, the `data-hud` collection scope (a mode panel must query only its own root, or the TDM panel grabs the FFA panel's slots), `memo`. Visual parity is checked by hand; no screenshot tests exist. |

### `screens/GameCanvas.tsx` (345 lines): **REVIEW**

Loading overlay, failure/retry, pause menu, exit confirm (now the shared `Confirm`), results, keyboard handling, chat wiring, custom labels from `link.welcome.lobby`. Splitting out `PauseMenu`, `ExitConfirm` and `LoadingOverlay` would help readability (Phase 7, optional). **Do not** touch the `startGame`/dispose effect or the link-closing semantics (now also `release` after a custom match) without the StrictMode double-mount cases in mind (§18).

### `screens/Results.tsx` (346 lines): **REFACTOR** (Phases 3 and 5)

| | |
|---|---|
| **Current responsibilities** | The results frame and the people line; per-mode panels by `mode.kind` branches (FFA's full record; TDM's team score and the MVP); Play again / Exit; **new:** for a lobby's match, the tally (`Tally` from `Lobby.tsx`), the winner highlight (`winnerKey`, line 82, mirroring the server's tally key) and Back to lobby. |
| **Should remain** | The generic frame, the buttons, the lobby tally display. |
| **Should move** | Per-mode panels (Phase 5); `winnerKey` → `lobbyRules.tallyKey` (Phase 3). |
| **Suggested destination** | `modes/<mode>/results.tsx` via `MODE_VIEWS`; `net/lobbyRules.ts` |
| **Reason** | Mode branching in a shared screen; a duplicated server rule. |
| **Risk** | Medium: manual visual parity (win/draw/loss, the MVP, the tally highlight). |

### `screens/Lobby.tsx` (415 lines, new): **REVIEW** (Phase 3 edits; split in Phase 7)

| | |
|---|---|
| **Current responsibilities** | The waiting room: the slot grid (sides, claims, bots and their difficulty, the owner's menus: kick, hand over, add or remove a bot), the bar (ready, start, and a hint from `startHint()` that mirrors the server's start rule), the settings card, the invite card (code, link), the tally (`Tally`), the docked chat, keys; `side()` mirrors the server's side rule. |
| **Should remain** | The screen and its flow. |
| **Should move** | `side()` and `startHint()` → `sideOf`/`startable` (Phase 3); the cards → components when next changed (Phase 7). |
| **Suggested destination** | `net/lobbyRules.ts`, `modes/traits.ts`; `screens/custom/{SlotGrid,SettingsCard,InviteCard}.tsx` |
| **Reason** | Two server rules re-implemented ("as the server holds it"); four concerns in one 415-line component. |
| **Risk** | Low–medium: the hint must keep the server's order of reasons (`alone`, `waiting`, `sides`). |

### `net/client.ts` (432 lines): **REFACTOR, light** (Phase 4)

Cohesive: the online mirror (snapshots, prediction, reconciliation, the roster from `ro`, empty seats taken out of the local world). Only the wire-event **decode** (`play`, `mine`, positional `f[n]`) moves into the shared codec (`net/events.ts`). Reconcile, place and step stay.

### `net/protocol.ts` (604 lines): **REFACTOR** (Phases 2, 3, 4)

| | |
|---|---|
| **Current responsibilities** | `PROTOCOL` (6) and `BUILD`; message types; quantisation; the binary snapshot frame (`packCars`, `packSnapshot`, `unpackSnapshot`; `HEAD_BYTES` 14, `CAR_BYTES` 44, `ME_BYTES` 40; the `absent` flag (value 8) shares the existing flags byte); `parseClient` validation and clamping; `weaponId`; the loadout `pick()`; `SLOTS = 12`; **the lobby wire** (about a fifth of the file, ≈120 lines: `LOBBY_ACTIONS` (18), `INVITE`, `readCode`, `LOBBY_FORM`, `tidy`, `LobbyForm`, `Lobbying`, `LobbyRow`, `LobbySlot`, `LobbyView`, `LobbyNote`, the `lb`/`lbs` messages, `lobbying()`). |
| **Should remain** | The match wire, in this file, at this path. |
| **Should move** | The lobby wire (Phase 4; `parseClient` delegates `lb`); `weaponId` → `spec.id` and `pick()` → a shared `parseLoadout` (Phase 2); `SLOTS` → `MAX_SEATS` (Phase 3); optionally the binary frame. |
| **Suggested destination** | `net/lobbyProtocol.ts`; `sim/loadout.ts` (`parseLoadout`); `modes/traits.ts` (`MAX_SEATS`); optional `net/snapshotFrame.ts` |
| **Reason** | The lobby wire grows with every lobby feature; the identity defect (§6.2). |
| **Risk** | Medium: bytes and JSON shapes; the Phase 0 fixtures guard them. **Do not move the file itself:** `scripts/match-smoke.mjs` reads `PROTOCOL` from `game/src/net/protocol.ts` with a regex. |

### `net/custom.ts` (274 lines, new): **REFACTOR, small** (Phase 7)

| | |
|---|---|
| **Current responsibilities** | The custom-lobby store: the list (watch), create, join (from the list, by code, by link), `ask` (an action as a promise), the waiting-room state, the match on the same socket (`link(socket, welcome, aside)`, then `release()`), leaving a match (Back to lobby), coming back within the grace (`comeBack`, `RETRY`, the `sessionStorage` mark), `resumeLobby` after a reload, `NOTES` (the refusals' texts). |
| **Should remain** | The store and its state machine. |
| **Should move** | The pieces identical to `net/matchmaking.ts`: `dial()`, the retry schedule and loop, the `sessionStorage` mark helpers. |
| **Suggested destination** | `net/sessionSocket.ts`, shared by both stores |
| **Reason** | The same machinery written twice. Two stores stay (§9.2). |
| **Risk** | Medium: the repo's review fixed four bugs in this layer; `client.check` (47) and `scripts/browser-match.mjs custom` guard it. |

---

## 3. Dependency graph

### 3.1 Folder-level graph (current)

```
                 main.tsx ──(dynamic import)──► App.tsx
                                                  │
              ┌───────────────────────────────────┼─────────────────────────────┐
              ▼                                   ▼                             ▼
          screens/  ──────────────────────►  game/ (runtime.ts, match.ts, …)  ◄──── hud/
              │                                   │  ▲                          │
              │                                   ▼  │ (net/client → game/*)    │
              └───────────► net/ ◄────────────────┘  │                          │
               (custom.ts,  │  (runtime.ts → net/connection; online.ts → net/client)
                matchmaking)└───────────────────────┘
server/ ──► game/ (sim, modes, items, matchSettings, maps→arena builders→materials→renderer), net/protocol.ts
```

### 3.2 Edges that matter (new or changed)

| Edge | Verdict |
|---|---|
| `net/protocol.ts → game/matchSettings.ts` (value: `checkSettings`, `CUSTOM`) `→ game/ai.ts` (value: `DIFFICULTIES`) | **REFACTOR**: the wire format now depends on the bot AI module (and through it Rapier) just to know the difficulty names. A leaf `difficulty.ts` (pure data) fixes it. |
| `server/custom.ts → net/protocol.ts` (value: `LOBBY_FORM`, `tidy`) and types from `game/` | **KEEP** (correct direction). After Phase 4 it imports `net/lobbyProtocol.ts`. |
| `game/matchSettings.ts → type game/modes.ts` (`Mode`) while `modes.ts → matchSettings.ts` | **REFACTOR**: root of the type cycle (§3.4) |
| `game/mode.ts → type game/items/supply.ts` while `supply.ts → mode.ts` (`ms`, `Feed`) | **REFACTOR**: second root (§3.4) |
| `game/modes.ts → items/pickups.ts, ffa/zone.ts` (Three.js scenery) | **REFACTOR** (Phase 5): the server bundle carries both views |
| `screens/Lobby.tsx → game/tdm/config.ts` (`TEAMS`), `game/roster.ts` (`botName`), `game/ai.ts` (`DIFFICULTIES`) | Fine as reads; the team-side and tally logic it re-implements moves to shared rules |
| `screens/Results.tsx → screens/Lobby.tsx` (`Tally`), `screens/Lobby.tsx → screens/Lobbies.tsx` (`Lock`) | **KEEP** (screen-to-screen sharing; move the small shared pieces to a `screens/custom/` folder later) |
| `hud/Hud.tsx → game/items/{config,items,supply}` | **KEEP** (generic supply readers) |
| `game/runtime.ts → net/connection.ts`, `game/online.ts → net/client.ts` | unchanged folder inversion (Phase 6) |

### 3.3 Unwanted-dependency audit

| Unwanted dependency | Present? | Note |
|---|---|---|
| domain → React | **No** | |
| domain → Three.js | **Maths only**, plus rendering types in the contract | `mode.ts` still has `show(camera: THREE.Camera)`; `modes.ts` still takes `THREE.Scene` |
| domain → DOM / wall clock / `Math.random` | **No** | `Math.random` only in `match.ts` (seed pick) and presentation; `custom.ts` uses `node:crypto` for codes and salts (server-only, not gameplay) |
| domain → WebSocket | **No** | |
| server → client-only code | **Yes, two files** | `items/pickups.ts` and `ffa/zone.ts` ride in through `modes.ts` (bundled, never instantiated on the server); Phase 5 removes them, and §14.1 rule 2 allows them until then |
| wire → AI | **Yes (new)** | see §3.2 |
| page re-implements server rules | **Yes (new)** | §1.5 |

### 3.4 The type-level cycle

Value imports have no cycle. Type imports now form one strongly connected set of 14 files (`modes`, `matchSettings`, `mode`, `simulation`, `ffa/{mode,rules,zone}`, `tdm/{mode,rules,types,tactics}`, `items/{items,supply,pickups}`). Every shortest cycle runs through one of two edges:

```
matchSettings.ts ──type Mode──► modes.ts ──► ffa/mode.ts ──► … ──► matchSettings.ts
mode.ts ──type Supply──► items/supply.ts ──value ms, type Feed──► mode.ts
```

Why it matters even though types are erased: (1) the mode ids live at the top of the graph (in the registry that imports every adapter and view), so any module that only needs the id type, including the plain-node `server/custom.ts`, nominally depends on everything; (2) the repo's `MODULE_BOUNDARIES.md` promises no cycles, type imports included; (3) a boundary checker cannot layer what loops. The fix is type-only (Phase 2): `Mode` ids in a leaf module, `MODES` declared with `satisfies Record<Mode, …>`; `ms`/`clock` in a leaf; the contract describes what it reads of a supply with its own small `SupplyView` type.

Full edge list: [Appendix A](#appendix-a-import-graph-current).

---

## 4. Architectural layers

### 4.1 The real constraints

Revision 1's four stand:

1. Server-reachable code must be import-safe and call-safe in Node (the arena builders are the documented exception: DOM shim + headless materials).
2. Authoritative gameplay must not depend on presentation, the wall clock or unseeded randomness.
3. React stays above the runtime.
4. Page and server share only the wire modules and gameplay.

The custom work adds one more:

5. **Lobby rules have one home that both the server and the page import.** The server decides; the page may predict (hints, highlights) only with the same code.

### 4.2 Recommended layers: 7 new folders beside `net/`, `screens/`, `hud/`

| Layer (folder) | Purpose | Allowed imports | Forbidden imports | Files (today → target) |
|---|---|---|---|---|
| **`shared/`** | Leaf utilities | nothing internal | anything internal | `rng.ts`; `Point`; `ms`/`clock` formatters |
| **`content/`** | Specs (data), 3D models/turrets, arenas, shared parts | `shared/`, `render/`, `three` | `sim/`, `modes/`, `view/`, `runtime/`, `net/`, React, audio | arenas, vehicles, weapon specs, `maps.ts` registry |
| **`render/`** | Shared Three.js infrastructure (WebGL context, material library + bake, geometry, sky, post) | `shared/`, `three` | everything above it | `renderer.ts`, `materials/*`, `geometry.ts`, `environment.ts`, `postprocessing.ts` |
| **`sim/`** | Authoritative headless gameplay: simulation, combat, physics, driving, AI, scoring, the mode contract, loadout type, difficulties | `shared/`, `content/` specs/types/registries, `three` (maths only), Rapier | `render/`, `view/`, `runtime/`, `net/`, React, DOM, `Math.random`, `Date.now`, `performance.now` | `simulation.ts`, `combat.ts`, `physics.ts`, `drive.ts`, `ai.ts`, `scoring.ts`, `mode.ts`, `loadout.ts`, new `difficulty.ts` |
| **`modes/`** | **Traits and ids (leaf)**, match settings, the shared **items** module, each mode as a feature folder, the registry, the roster; client-only files (scenery, panels) reachable only through two client registries | `sim/`, `content/`, `shared/`; scenery files may import `render/`; panel files React | domain files: as `sim/` | `traits.ts` (new), `settings.ts` (today `matchSettings.ts`), `items/`, `ffa/`, `tdm/`, `index.ts` (today `modes.ts`), `roster.ts`, `scenery.ts`, `views.ts` |
| **`view/`** | What the local player sees, hears and controls | `render/`, `content/`, `sim/` (read), `modes/` (types, traits), `shared/`, DOM, WebAudio | `runtime/`, `net/`, `screens/`, `hud/`, React | `view.ts`, `effects.ts`, `camera.ts`, `audio.ts`, `sounds.ts`, `feed.ts`, `pilot.ts`, `input.ts`, `settings.ts`, `turntable.ts` |
| **`runtime/`** | Browser composition: `startGame`, `playMatch`, practice and online sources, loading, local persistence | everything below + `net/` | `screens/`, `hud/` | `runtime.ts`, `match.ts`, `practice.ts`, `online.ts`, `loading.ts`, `stored.ts` |
| **`net/`** (unchanged) | Wire (`protocol.ts`, `lobbyProtocol.ts`, `events.ts`, **`lobbyRules.ts`**: server-safe), socket, client mirror, prediction, interpolation, the two stores (+ shared session helpers), Nakama session, chat | `sim/`, `modes/` (domain), `content/` specs, `shared/` | `view/`, `runtime/`, `render/`, `screens/`, `hud/` | as today + the new wire modules |
| **`screens/`, `hud/`** (unchanged) | React UI; optional `screens/custom/` for the lobby screens | `runtime/` (Match API), registries, traits, `modes/views.ts`, `net/` stores and `lobbyRules.ts`, `view/settings`, `view/turntable` | Rapier, `sim/physics`, `sim/simulation` values, `render/` (except via the turntable) | as today |
| **`server/`** (flat) | Authoritative server | `sim/`, `modes/` (domain), `content/`, `net/{protocol,lobbyProtocol,lobbyRules,events}.ts`, `shared/` | `view/`, `runtime/`, `screens/`, `hud/`, `render/` (except through arena builders), the stores, React, Nakama JS | as today |

**Not added**, for the same reasons as revision 1: `application/` (`runtime/` is it), `engine/` (no physics abstraction), `platform/` (3–4 files), `presentation/` (`screens/` and `hud/` are fine). One thing **is** added that revision 1 did not have: a leaf **`modes/traits.ts`**, because the custom work proved that code outside a mode folder needs a few facts about every mode (§5, §7).

---

## 5. Contracts and interfaces

Each candidate is judged on whether it solves a problem **in this codebase**.

| Contract | Exists? | Consumers → implementers | Verdict |
|---|---|---|---|
| **`MatchMode`** | Yes (`game/mode.ts`) | sim, `playMatch`, room, client, HUD → `ffa/mode.ts`, `tdm/mode.ts` | **KEEP**; remove `show(camera)` (Phase 5); keep `share`/`mirror`; replace `supply?: Supply` with `supply?: SupplyView` (what the HUD, minimap and view read), which breaks the type cycle (Phase 2) |
| **`ModeRules`** (+ `leave`/`enter`) | Yes | sim, `feed.beep`, HUD → FFA/TDM rules | **KEEP** |
| Mode registry entry (`MODES`) | Yes (`modes.ts`) | MapSelect, roster, room, lobby, client, lobby screens | **KEEP**; declare it over the leaf ids (`satisfies Record<Mode, …>`, Phase 2); move the scenery wiring out (Phase 5). Revision 1's proposed `teamPlay` flag becomes the `sides` trait. |
| **`MatchSettings`** | Yes (new) | rules, roster, room, protocol, form, waiting room, records, replays | **KEEP**; move the per-mode parts (`classic(mode)`'s numbers, which sizes, whether friendly fire applies) to traits. Keep the labels beside it (the codebase keeps copy next to its data, as `VehicleSpec.blurb` does). |
| **`ModeTraits`** (new) | No | `matchSettings` (`classic`, `checkSettings`), `server/custom.ts` (sides, free slot, regroup, slot claims, start rule, tally), `server/room.ts` (team chat), `tdm/mode.ts` (`lineUp`), screens (labels, hints), HUD/protocol/roster (`MAX_SEATS`) | **ADD (Phase 3).** Removes about 30 mode-literal lines and three duplicated rules across both sides of the wire. Not over-engineering: one small data table with consumers on the server, in the rules and in the UI. |
| **Shared lobby rules** (new) | No (duplicated) | `server/custom.ts` (authority), `Lobby.tsx` (hints), `Results.tsx` (tally highlight) | **ADD (Phase 3)**: `sideOf`, `startable`, `tallyKey` as pure functions over the slot list both sides already have |
| **`Supply`** | Yes (new) | FFA and TDM rules; HUD/minimap/view via `MatchMode.supply` | **KEEP**; its hooks (`wave()` shaping drops) are the right size |
| **`SeatPlan`** | Yes (new) | roster, room, replay header | **KEEP**; add a vehicle only with Phase 8 |
| **`LobbyHooks`** | Yes (new) | `server/custom.ts` → `server/lobby.ts`, `custom.check` | **KEEP** (the matcher's pattern) |
| Room options for custom rooms (`lobby`, `plan`, `chat`, `over`, `idle`) | Yes (new) | `lobby.ts`, `replay.ts` → `room.ts` | **REVIEW**: group into one `custom?: { lobby, plan, chat, over, idle }` option now (no behaviour change); a `RoomKind` object only with a third kind (§10) |
| **`VehicleSpec`** (definition) | Yes (`vehicle/vehicles.ts`) | physics, garage, rewind, view, protocol → `ROSTER` entries | **KEEP**; move the `Handling`/`Chassis` types next to it (content), out of `drive.ts` (Phase 6) |
| **`WeaponSpec`** (definition) | Yes (`combat.ts`) | sim, garage, HUD, bots, lobby form → `ROSTER` entries | **KEEP**, and **add `id`** (identity), **`turret`** (an open key replacing the `model` union), **`icon`** (SVG path data), optional **`cue`** (fire sound) and optional **`ai`** style (Phase 2) |
| Turret builder registry (new) | No (a ternary) | vehicle model builders, turntable → `content/weapons/<id>/turret.ts` | **ADD (Phase 2)**: a plain `Record<string, () => THREE.Group>` |
| **`Arena`** (built) + **`MapInfo`** (definition) | Yes | sim, modes, physics, runtime, server → arena builders | **KEEP** |
| **`SimEvents`** | Yes | sim → view, recorder, client playback, checks (no-ops) | **KEEP**; the wire codec mirrors it |
| Wire event codec (new) | No (split) | `server/recorder.ts`, `net/client.ts` → `net/events.ts` | **ADD (Phase 4)**: one definition of each event's fields for encoder *and* decoder |
| **`Feed`** | Yes | mode adapters → `feed.ts` | **KEEP** |
| **`MatchSource`** | Yes | `playMatch` → `createMatch`, `createOnlineMatch` | **KEEP**: the "practice and online share the core" seam |
| **`Link`** (+ `aside`, `release`) | Yes (`net/connection.ts`) | client, both stores, checks in Node | **KEEP**: it is the match transport and the socket-handoff seam; no separate `MatchTransport` interface |
| `RandomSource` | Yes: `() => number` + `createRng` | sim, AI, rules, supply | **KEEP**: a function type is the right size |
| **`Loadout`** | Yes (ids) | App, protocol, room, lobby, stores | **KEEP**; add a pure `parseLoadout(unknown)` used by both the `localStorage` restore and `parseClient` (Phase 2) |
| `ModeViews`, `ModeScenery` (client) | No | HUD, Results, runtime → per-mode panels; FFA's hot zone | **ADD (Phase 5)**. `ModeScenery` shrinks since revision 1: the pickups view can be driven **generically** from `mode.supply`; only FFA's hot zone needs a per-mode entry (a `Partial<Record<Mode, …>>`). |
| `Clock` | Partly: sim time is the `dt` argument; the matcher and the lobby service take `now()` | – | **REJECT** a general interface: gameplay has no wall clock, and the two hooks cover the services |
| `PhysicsWorld` / `PhysicsBody` | No | – | **REJECT** (§19): one engine; determinism depends on the exact Rapier call sequence; prediction replays `drive.ts` on the same Rapier |
| `Audio` | No (module functions) | view, feed | **REJECT**: `SimEvents` already isolates the sim from audio; headless runs never import it |
| `AssetLoader` | No (`public/models/` does not exist yet) | – | **POSTPONE** to the first glTF asset. It needs its **own disposal rule**: glTF materials are per-asset, unlike `materials/library.ts`, which must never be disposed (§18). |

**`ModeTraits` sketch** (pure, plain-node safe; its only imports are the mode configs):

```ts
// modes/traits.ts — what code outside a mode's folder may know about any mode.
// The ids live here (near the bottom of the graph), so nothing needs the registry for the type.
export const MODE_IDS = ['tdm', 'ffa'] as const
export type Mode = (typeof MODE_IDS)[number]

export interface ModeTraits {
  sides: 0 | 2              // 0: every machine its own side; 2: two sides, the first half of the seats blue
  sizes: readonly number[]  // line-up sizes a custom lobby may choose (today CUSTOM.sizes[mode])
  friendlyFire: boolean     // the option means something in this mode
  classic: { size: number; duration: number; pickups: boolean } // what Classic and practice play (today classic(mode))
}

// Each entry is a plain object declared in its own mode folder (tdm/config.ts, ffa/config.ts);
// this file only lists and checks them. The configs import nothing from here, or the type cycle returns.
export const MODE_TRAITS = { tdm: TDM_TRAITS, ffa: FFA_TRAITS } satisfies Record<Mode, ModeTraits>
export const MAX_SEATS = Math.max(...MODE_IDS.flatMap((mode) => MODE_TRAITS[mode].sizes)) // 12: HUD pools, protocol SLOTS, bot names
export const sideOf = (mode: Mode, size: number, seat: number) => (MODE_TRAITS[mode].sides ? (seat < size / 2 ? 0 : 1) : seat)
```

```ts
// net/lobbyRules.ts — pure, server-safe; the server decides with it, the page predicts with it.
export function startable(lobby: { mode: Mode; size: number; slots: readonly LobbySlot[] }): '' | 'alone' | 'waiting' | 'sides'
export function tallyKey(mode: Mode, winner: number, slots: readonly LobbySlot[]): string | undefined // side, uid, or 'bot:' + slot
```

I checked the cut on today's graph: with `matchSettings.ts` taking `Mode` from such a leaf (the leaf importing only the two configs) and `mode.ts` no longer importing `items/supply.ts`, the type-level strongly connected set disappears (0 cycles). [High confidence: computed with the Appendix B script on `3b0943d`.]

---

## 6. Plug-and-play content

For each content type: **today** (what a developer touches now), then **target**.

### 6.1 Vehicle

**Today:** a `ROSTER` entry; a model builder + `MODELS` entry; and the server side is still missing: `room.join` keeps the roster's vehicle (Classic and custom alike), `server.check.ts:368` fails on purpose with two vehicles, bots use `BOT_VEHICLE`, the `SeatPlan` has no vehicle, `ro` and the journal's `join` carry none, the garage has no vehicle pager, `arena.check` checks spawn clearance with the Razor's shells only.

**Target** (after Phases 6 and 8):

```
Developer creates:  content/vehicles/<id>/{spec.ts, model.ts}
Registers:          content/vehicles/index.ts (VEHICLES), content/vehicles/models.ts (MODELS)
Core changes:       none, once Phase 8 (a one-time capability, PROTOCOL bump) has landed
Custom lobbies:     optionally a `vehicles` match setting (one vehicle for everyone), as `weapons` today
```

### 6.2 Weapon

**Today:** `combat.ts` entry + the closed `model` union; a turret builder + the ternary in `vehicle/vehicle.ts:195`; another hard-coded HUD SVG; sound and bot style chosen by the `rocket` flag; **identity by turret model** (`protocol.ts:254`). Automatic today: the garage, bots' roster, and (new) the lobby form's weapon restriction and its label (`weaponsLabel` uses `spec.name`).

**Target** (after Phase 2): unchanged from revision 1 —

```
Developer creates:  content/weapons/<id>/spec.ts (id, numbers, turret key, icon path, cue?, ai?), turret.ts if new
Registers:          content/weapons/index.ts (WEAPONS), content/weapons/turrets.ts (TURRETS, only for a new turret)
Core changes:       none for a hitscan gun or a rocket weapon; a new FIRING BEHAVIOUR stays sim work
Automatic:          garage, bots, lobby restriction, HUD name/ammo/icon, wire/records/replays by spec.id
```

### 6.3 Arena

**Today:** builder + `MAPS` entry + preview + `digests.json`; must build headless. **New requirements from custom lobbies:** each team base needs **6** starts (TDM up to 6 v 6; `arena.check` asserts it); `arena.spawns` serves FFA lines up to 12 (`spawns[i % length]`: 16 exist on both arenas); TDM pickups need nav nodes clear of every start (`itemSpots`).

**Target:** feature folder + one `MAPS` line + one digest line. Add to `arena.check` (Phase 0/1): for every map and every mode it hosts, line up the **largest** size the mode's traits allow and build the adapter headless.

### 6.4 Game mode

**Today** a mode needs its folder (config, rules with `leave`/`enter` and `settings`, adapter, check) and a `MODES` entry, **and** edits in: `matchSettings.ts` (`classic`, `CUSTOM.modes`, `CUSTOM.sizes`, `checkSettings`), `server/custom.ts` (sides, free slot, regroup, slot claims, start rule, tally key), `server/room.ts` (team chat), `screens/Lobby.tsx` and `LobbyForm.tsx` (labels, hints, defaults), `screens/Results.tsx` and `hud/Hud.tsx` (panels), and each map's `modes` list.

**Target** (after Phases 3 and 5):

```
Developer creates:  modes/<id>/{config.ts (+ its ModeTraits), rules.ts, mode.ts, <id>.check.ts}
                    modes/<id>/{hud.tsx, results.tsx} (client), modes/<id>/scenery.ts (optional, client)
Registers:          modes/traits.ts (MODE_TRAITS: one line + the id in MODE_IDS)
                    modes/index.ts (MODES: one line), modes/views.ts (MODE_VIEWS: one line)
                    modes/scenery.ts (only if it draws something of its own)
                    content/arenas/index.ts (add the id to each map that hosts it)
Core changes:       none: simulation, playMatch, room, lobby, custom.ts, client, Hud.tsx, Results.tsx,
                    Lobby.tsx and LobbyForm.tsx untouched
Plugs in for free:  pickups (the supply), sizes, durations, respawn speed, kill limit, empty seats
```

### 6.5 Pickup (rewritten: pickups are now a shared module)

**Today:** `items/items.ts` (`ItemType`, `ITEMS`, and a `GROUPS` entry: which lobby toggle turns it on), `items/config.ts` (numbers), `items/supply.ts` `apply()` (or an `Effect` key), `items/pickups.ts` token geometry (exhaustive `switch` on `ItemType`: a compile error if missed), **plus** `hud/Hud.tsx` `EFFECTS` (a hard-coded chip list) and `supply.share()` (lists the four effect keys by name).

**Target** (Phase 5, small): derive the HUD chips from `ITEMS` (a `timed` flag) and loop the `Effect` keys in `share`/`mirror`.

```
Developer creates/edits (items/ only):  ITEMS + GROUPS entry, config numbers, supply.apply, token geometry
Core changes: none; every mode that plugs the supply in gets it
```

A **new pickup group** (a fourth lobby toggle) also edits `MatchSettings.items`, `checkSettings`, `ITEM_GROUPS` (labels) and the form; that is a match-setting change (§6.9).

### 6.6 Effect (visual)

**Today:** a method on the pooled particle system (`effects.ts`) + a call from `view.ts` on a `SimEvents` callback. **KEEP.** An effect registry would be over-engineering at this size. If `effects.ts` grows past a handful of new emitters, split it into `view/effects/pool.ts` + emitter files. New code must skip absent machines, as the view now does (§8.4).

```
Developer creates/edits:  an emitter method in view/effects.ts + its call in view/view.ts (a SimEvents callback)
Core changes:             none
```

### 6.7 Audio

**Today:** a synth function in `sounds.ts` + a `CUES`/`LOOPS` entry in `audio.ts` (a registry keyed by the `SoundCue` union) + the call site (view or feed). **KEEP.** The only addition is the optional weapon `cue` (§6.2), so a weapon's fire sound becomes data instead of `spec.rocket ? 'launch' : 'shot'` (`view.ts:114`).

```
Developer creates/edits:  a synth in view/sounds.ts, a CUES/LOOPS entry in view/audio.ts, the call site
Core changes:             none
```

### 6.8 AI behaviour

- **Per mode:** the `Plan` hooks supplied by the adapter: `value` (how much a rival is worth as a target), `errand` (where to go with nobody to fight) and, new, `careful` (friendly fire is on: hold fire rather than hit a teammate, via `clearOfMates`). **KEEP.** This is already the plug-in point.
- **Per difficulty:** the `DIFFICULTIES` registry. **KEEP**, as a leaf data module (Phase 2): the wire format and the lobby screens import it.
- **Per weapon:** today `TACTICS` is picked by the `rocket` flag. **REFACTOR (Phase 2):** an optional `spec.ai` style, with the existing two as defaults.
- **New bot behaviours** (for example, item hunting in TDM now that TDM can have pickups) go through `Plan` first. Change `think()` only if a behaviour cannot be expressed as a target value, an errand or a flag.

```
Developer creates/edits:  the mode's Plan (modes/<id>/tactics.ts or its adapter); a DIFFICULTIES entry; a weapon's spec.ai
Core changes:             none, unless the behaviour cannot be a Plan hook (then sim/ai/think.ts, with bots.check ranges)
```

### 6.9 Match setting (a custom-lobby option) — new

**Today**, one option (e.g., "no respawn", "low gravity") touches: `MatchSettings` (type), `classic()`, `CUSTOM` (choices), `checkSettings`, the rules or the simulation that enforce it, `LobbyForm.tsx` (field), `Lobby.tsx` (settings line), a label helper, `protocol.check`/rules checks, and **`PROTOCOL`** (the settings travel in the welcome and the lobby view). Records and replays carry it for free (the whole object travels).

**Verdict: KEEP the hand-written approach for now.** Seven options do not justify a declarative descriptor that drives validation, form and labels. Revisit if options pass about ten, or several arrive at once. [Medium confidence] The one improvement worth making now is that per-mode validity (friendly fire only with sides) comes from traits, not `mode === 'ffa'`.

### 6.10 Lobby feature (spectators, map vote, …) — new

A lobby feature is inherently cross-layer: a `custom.ts` action and its refusals, a `LOBBY_ACTIONS` entry and parser rule, `LobbyView` fields, a store `ask`, UI, `custom.check` cases, a `PROTOCOL` bump. That is honest work, not a leak. Two things keep it cheap: the lobby protocol in its own module (Phase 4) and the rules both sides need in `lobbyRules.ts` (Phase 3).

---

## 7. Game mode architecture

### 7.1 Current

```
 practice: classic(mode) ─┐                          custom lobby: owner's settings ─┐
                          ▼                                                          ▼
          playMatch / room.ts / net/client ── MODES[kind].create({ …, settings }) ──► MatchMode (mode.ts)
                                                                                     │      ▲ supply?
                                                     ┌───────────────────────────────┴──────┤
                                                     ▼                                      ▼
                                            ffa/mode.ts (adapter)                   tdm/mode.ts (adapter)
                                                     │                                      │
                                            ffa/rules.ts ── items/supply.ts ── tdm/rules.ts + tactics.ts
                                                     └─────────── scoring.ts ───────────────┘
```

The engine still reaches a mode only through `MatchMode`, and pickups plug in through `supply` rather than through mode branches. That part is in good shape. [High confidence]

### 7.2 What leaks, and the fix

| Leak | Where | Fix | Phase |
|---|---|---|---|
| **Per-mode facts asked directly** | `matchSettings.ts` (Classic numbers, sizes, friendly-fire validity), `server/custom.ts` (sides ×4, tally), `server/room.ts:236` (team chat), `LobbyForm.tsx`, `Lobby.tsx` | `MODE_TRAITS` (§5): `sides`, `sizes`, `friendlyFire`, `classic` | **3** |
| **Rules written twice** | side of a slot ×3, start conditions ×2, tally key ×2, 12 seats ×5 | `sideOf`, `startable`, `tallyKey`, `MAX_SEATS` in one place each | **3** |
| Mode panels in shared UI | `Hud.tsx`, `Results.tsx` (incl. the MVP panel), the lobby tally display | `MODE_VIEWS` (HUD panel, results panel); the tally display reads `sides` | 5 |
| Rendering in the contract | `MatchMode.show(camera)`, `ModeContext.scene`, `modes.ts → createPickups, createHotZone` | runtime draws pickups from `mode.supply` (generic) and FFA's zone through `MODE_SCENERY` | 5 |
| Type cycle | `Mode` from the registry; `mode.ts ↔ supply.ts` | ids in `modes/traits.ts`; `SupplyView` | 2 |

### 7.3 Where each concern lives (target)

| Concern | Location | Server-reachable? |
|---|---|---|
| Rules, respawn choice, outcome, online state | `modes/<m>/rules.ts`, `mode.ts` | yes |
| Match settings (type, validator, choices, labels) | `modes/settings.ts` | yes |
| What other code may know of a mode | `modes/<m>/config.ts` traits, listed in `modes/traits.ts` | yes |
| Pickups (catalogue, supply) | `modes/items/` | yes |
| Pickups (tokens) | `modes/items/pickups.ts`, drawn generically by the runtime | **no** |
| Scoring | `sim/scoring.ts`, numbers from each config | yes |
| Bot behaviour | `sim/ai/*` + the mode's `Plan` | yes |
| HUD, results panels | `modes/<m>/{hud,results}.tsx` via `MODE_VIEWS` | **no** |
| Mode scenery (FFA hot zone) | `modes/ffa/zone.ts` via `MODE_SCENERY` | **no** |
| Lobby rules (sides, start, tally) | `net/lobbyRules.ts` (reads traits) | yes |

### 7.4 Timing and phases

Unchanged: `ModeTiming` and `ModePhase` are generic enough. **KEEP.** Respawn waits as shares of the clock (30/50/80 %) reproduce Classic's 180/300/480 s exactly; the pins guard that.

### 7.5 "Custom": decided and built

Revision 1 asked whether "Custom" meant (a) a third fixed mode or (b) parametrised rules. The owner chose **(b), plus a social layer**: Custom is a lobby flow (list, invites, waiting room) whose matches run FFA or TDM on the owner's `MatchSettings`. The rules now take settings (`settings.duration`, `respawnWait(…, settings, …)`, kill limit, friendly fire, item groups), which is exactly the config injection revision 1 estimated. A third fixed mode is still possible and is what §6.4's target describes.

---

## 8. Client / simulation / rendering separation

### 8.1 Evaluating the brief's intended direction

```
Gameplay state → Simulation → View adapter → Three.js
```

**Correct, and implemented.** Combatants and the rules hold the state. `simulation.step` mutates them and reports through `SimEvents`. `view.ts` reads state each frame (`place`, `animate`) and handles events. Three.js meshes never feed back. The custom work kept it: empty seats are a simulation fact (`present`) that the view reads. [High confidence]

```
React → Application/UI state → Game runtime
```

**Correct, and implemented.** `App.tsx` holds UI choices (five `useState` values: screen, loadout, pick, seat, run). `GameCanvas` calls `startGame()` (runtime) and receives the `Match` object. The HUD reads `Match` every frame through an imperative handle. React never steps the game. The custom-lobby screens read a store (`net/custom.ts`) through `useSyncExternalStore` (`useCustom` in `screens/search.ts`), never the match. [High confidence]

### 8.2 Boundary table

| Concern | Owner | Talks to | Rule |
|---|---|---|---|
| Simulation | `sim/simulation.ts` | Rapier, rules, `SimEvents` | No meshes, audio, DOM or wall clock |
| Rapier physics | `sim/physics.ts`, `sim/drive.ts` | the sim; `net/prediction.ts` drives the same `drive.ts`; `view/pilot.ts` ray-casts for aim (non-authoritative) | One world per match; creation order is part of determinism |
| Three.js rendering | `view/view.ts` (+ `render/*`) | reads combatants; implements `SimEvents` | Never writes gameplay state |
| React UI | `screens/`, `hud/` | the `Match` object; `MODE_VIEWS`; the two stores | No per-frame re-render; no physics imports |
| Input | `view/input.ts` → `view/pilot.ts` → `Combatant.control` | DOM | Controls are data; the chat box stops keys (already) |
| Audio | `view/audio.ts`, `view/sounds.ts` | view, feed | Presentation only; unseeded randomness allowed |
| Effects | `view/effects.ts` | view | Pooled; unseeded |
| **Empty seats** (new) | `sim/simulation.ts` (`present`, `vacate`, `occupy`), the rules (`leave`, `enter`) | rewind, recorder, client, view, HUD, minimap, scoreboard, results, bots, spawn choice | Every per-seat reader decides what an absent machine means (§8.4) |
| **Lobby UI state** (new) | `net/custom.ts` (store) | `screens/Custom`, `Lobbies`, `Lobby`, `LobbyForm`; `App.tsx` | Rules the server decides are only *predicted* on the page, with the shared `lobbyRules.ts` (Phase 3) |

### 8.3 One REVIEW item

`view.ts` reads the Rapier vehicle controller (`wheelIsInContact`, `poseWheels` from suspension length) for dust, tyre squeal and wheel poses. Online remote cars stay kinematic bodies in a local world, with `controller.updateVehicle(dt)` run only so these reads work (`net/client.ts` `step`). This is presentation reading the physics engine. It is acceptable while every client runs a local world. If a future client ever renders without local physics (spectator, replay viewer), wheel contacts must go into the pose data. **Postpone.** [High confidence on the mechanism]

### 8.4 A rule for every future feature: empty seats

A machine can be out of play (`present` false). It is skipped by the simulation's loops, its body is disabled (rays, blasts and cars pass through), the rewind ignores it, the view hides it (no smoke, fire or burning loop), the page takes it out of its local world, and the HUD markers, minimap, scoreboard and results leave it out. **Any new code that iterates combatants must decide what an absent machine means** (the repo's own plan lists this as its first risk). The checks walk an empty seat through the simulation, the rules and the page; nothing enforces it for new code, so it goes into the review checklist and §18.

---

## 9. Networking architecture

### 9.1 Boundaries

| Concern | Module | Server-safe? | Verdict |
|---|---|---|---|
| Protocol (messages, quantisation, binary frame, validation, `PROTOCOL`, `BUILD`) | `net/protocol.ts` | yes | **KEEP**, minus the lobby part; fix `weaponId`; keep the path |
| Lobby protocol (types, actions, invite codes, `tidy`, `readCode`, the `lb` parser) | in `protocol.ts` today | yes | **REFACTOR → `net/lobbyProtocol.ts`** (Phase 4); `parseClient` delegates `lb` to it |
| Lobby rules both sides use | duplicated today | yes | **ADD `net/lobbyRules.ts`** (Phase 3) |
| Wire events codec | split | – | **REFACTOR → `net/events.ts`** (Phase 4) |
| Transport (`Link`, `aside`, `release`) | `net/connection.ts` | runs in Node for checks | **KEEP** |
| Replication, prediction, interpolation | `client.ts`, `prediction.ts`, `snapshots.ts` | DOM-free | **KEEP** (empty seats handled) |
| Classic store | `net/matchmaking.ts` | browser | **KEEP** the store; share helpers |
| Custom store | `net/custom.ts` | browser | **KEEP** the store; share helpers |
| Identity, chat | `session.ts`, `chat.ts` (+ notes) | browser | **KEEP** |

### 9.2 The two stores

Both stores open a session socket (`freshSession` → `openSocket` → `sayHello` with no map), keep a module-level state with listeners, remember a mark in `sessionStorage`, guard races with an `attempt` counter, retry on the same schedule (`[0, 1000, 2000, 4000, 7000]` ms) in a `comeBack` loop, and turn a welcome into the match's `Link`. They differ in what they mean: Classic hands the socket to the match for good; custom keeps it through the match (`aside`) and takes it back (`release`). The server enforces one session per user and "Classic or custom, not both"; the page mirrors that in `MapSelect` and `Custom.tsx`.

**Recommendation (Phase 7):** extract `net/sessionSocket.ts` with the shared, already-identical pieces (`dial()`, the retry schedule and loop, the `sessionStorage` mark helpers). **Keep two stores**: their state machines are different, and the repo's review (`custom/LOG.md`, 2026-10-06) fixed four bugs in exactly this layer (StrictMode's double mount dropping an invite, a kicked player landing in a practice match, a reload landing on the main menu, a drop while seated leaving the store stuck). A merged "session manager" would put both flows at risk for little gain. [Medium confidence]

### 9.3 Domain vs transport

Unchanged: the domain never touches the WebSocket; practice and online share the core through `MatchSource`; custom matches add no new source (they are online matches with settings).

### 9.4 The codec refactor (concrete)

Today, adding one `SimEvents` callback means changing:

- `simulation.ts` (the interface + the call),
- `view.ts` (the handler),
- `server/recorder.ts` (`events.push(['xx', tick, …positional])`),
- `net/client.ts` in two places (`play()` decodes `f[n]`; `mine()` decides "the player's own"),
- and possibly `PROTOCOL`.

`mine()` hard-codes that the victim of `'sh'` sits at `f[11]` (`net/client.ts:143`). The twelve codes today: `sh`, `ln`, `rk`, `bu`, `hu`, `wr`, `cr`, `rl`, `sp`, `rc`, `ru`, `go` (unchanged by the custom work).

**Target:** `net/events.ts` exports, per wire code:

- the field layout (names → positions) and quantisation,
- `encode(…)`, used by the recorder,
- `decode(row)`, used by the client and returning a typed object,
- `owners(row)` (a new function): the seat ids that make an event "mine", replacing the hand-written `mine()` switch.

A round-trip check covers every code (`protocol.check`), and an old-vs-new table asserts that `owners()` agrees with today's `mine()` for every code. **Bytes on the wire must not change**, so `PROTOCOL` stays 6; the Phase 0 fixtures prove it.

---

## 10. Server architecture

| Concern | Module | Verdict |
|---|---|---|
| Transport + door (origins, auth, rate, strikes, backlog, hello deadline) and the loop | `server.ts` | **KEEP**; optionally extract the dev latency simulator to `netsim.ts` |
| Authentication | `auth.ts` (HS256 verify/mint) | **KEEP** |
| Classic matchmaking policy | `matchmaker.ts` (pure, hooks) | **KEEP** |
| **Custom lobbies** | **`custom.ts` (pure, hooks)** | **KEEP**; read traits and `lobbyRules.ts` (Phase 3) |
| Rooms and who goes where | `lobby.ts` (both flows) | **REVIEW**; extract the custom hooks only if a third flow appears |
| Match hosting (simulation) | `room.ts` | **REFACTOR, light** (journal format, input policy; group the custom options) |
| Replay (write) | `room.ts` journal → `records.ts` | extract the format to `journal.ts` (Phase 7) |
| Replay (read/run) | `replay.ts`, `replay-main.ts` | **KEEP**; import the format from `journal.ts`; seats a custom room's people where the journal says |
| Persistence | `records.ts` (match records, replays, `REPLAY_DAYS` pruning) | **KEEP** |
| Fair play | `fairplay.ts` (pure) + `room.watch()` (geometry) | **KEEP** (optionally move `watch()` beside `fairplay.ts` when `room.ts` is split) |
| Lag compensation | `rewind.ts` (skips absent machines) | **KEEP** |
| Headless arenas | `arenas.ts`, `headless.ts`, `digests.json` | **KEEP** |

**Folder structure: KEEP `server/` flat.** 17 production files (16 for the server plus `browser.ts`, which emulates pages for the checks), each cohesive. Subfolders would add path noise, churn `vite.server.config.ts` `ENTRIES`, and prevent no mistake. [High confidence]

**Room kinds.** `room.ts` now serves two kinds of room. The difference is a handful of policies: which seat a person takes, which gun they get, what happens when they leave, what follows the results, whether an empty match is abandoned, what idleness does, whether matchmaking may use the room, and what the record and journal note. Today these are `lobby ? … : …` branches. A `RoomKind` object (`{ seatFor, gunFor, onLeave, afterResults, onIdle, open }`) would make a third kind additive, but with two kinds it is an indirection the code has to be read through. **Recommendation:** group the custom options into one `custom?: { lobby, plan, chat, over, idle }` option and keep the branches next to each other; extract `RoomKind` when a third kind is on the roadmap (the repo lists stats and leaderboards for the next phase, which might bring ranked play). [Medium confidence]

**Capacity and operations (facts, not refactors):** custom matches and Classic share `MAX_ROOMS` (12) and are counted by kind; `MAX_LOBBIES` (24); `deploy/compose.yml` deliberately passes neither (podman-compose 1.3.0 would hand `${VAR:-default}` on as text), so the defaults apply. Lobbies live in memory: a deploy ends them all. Password checks (scrypt, N=1024, about 2 ms) run on the loop's one thread; the lockout (5 wrong a minute per person, then 60 s shut) and the door's rate limits bound them.

**Server-specific boundary rules** are §14.1 rules 2 and 4 (reachability from `main.ts`, `load.ts`, `replay-main.ts`; the two pure services).

---

## 11. Definition vs runtime state

| Pair | Today | Mixing? | Recommendation |
|---|---|---|---|
| `VehicleSpec` / `Combatant` + `Car` | Separate; the combatant holds a `vehicle` id and a `car` built from the spec | No | **KEEP** |
| `WeaponSpec` / `WeaponState` | `WeaponState.spec` holds a spec object; **bots hold a scaled copy** (`botGun`), so the definition is copied per instance and its identity is recovered from the turret model. Custom seats get `WEAPONS[gun]` itself, and the lobby's one-gun setting goes through the same `weaponId` lookup | **Yes** | **REFACTOR:** `id` on the spec (copies keep it). Longer term, optionally `WeaponState { id, modifiers }`, but `id` alone fixes the defect. [High confidence] |
| `MapInfo` / `Arena` | Registry entry vs built arena (immutable during play; `update()` is visual) | Layout is **derived from visual meshes** (`solid()` → `collectColliders`) | **REVIEW → postpone** (§18). For **new** maps only, allow an optional `layout()` separate from `dress()` if a map author wants it. Do not rewrite the two existing maps (digest risk). |
| Mode config (`FFA`, `TDM` constants) + **`MatchSettings`** / rules state + combatants | Config and settings are read; the rules own their state | No | **KEEP**. Revision 1's question (config injection for parametrised rules) is answered: the rules now take `settings`. |
| **`MatchSettings`** (how a match is played) / rules state (new) | Settings are read, never written, by the rules; they travel in the welcome, the replay header and the record | No | **KEEP** |
| `Loadout` (ids) / armed combatant | ids resolved in `createMatch` / `room.join` | No | **KEEP**; add `parseLoadout` (Phase 2) |
| `Skill` (difficulty) / `Brain` | Separate | No | **KEEP**; `Skill` and `DIFFICULTIES` to a leaf (Phase 2) |
| `ITEMS` + `GROUPS` (catalogue) / `supply` items and effects | Catalogue vs the supply's live items and timed effects | No | **KEEP** |
| `Lobby` (server state, with secrets) / `LobbyRow`, `LobbyView` (wire projections) (new) | The list's rows carry no code, password or user id; the members' view carries the code, never the password | No | **KEEP** |
| `SeatPlan` (who sits where) / `Combatant.present` (in play now) (new) | The plan is fixed at the start; `present` changes as people leave and come back | No | **KEEP** |

---

## 12. Registry architecture

| Registry | Location | Keyed by | Verdict |
|---|---|---|---|
| `VEHICLES` | `vehicle/vehicles.ts` | `VehicleId` (`keyof ROSTER`) | **KEEP** plain record; move to `content/vehicles/index.ts` with per-vehicle files (Phase 6) |
| `MODELS` | `vehicle/vehicle.ts` | `VehicleId` | **KEEP**; own file `content/vehicles/models.ts` (client + headless-safe) |
| `WEAPONS` | `combat.ts` | `WeaponId` | **KEEP**; add `id` (Phase 2); move to `content/weapons/index.ts` (Phase 6) |
| `TURRETS` | *(missing: a ternary)* | turret key | **ADD** (Phase 2) |
| `MAPS` | `maps.ts` | `MapId` | **KEEP**; move to `content/arenas/index.ts`; the `loadArena` cache → `runtime/` (Phase 6) |
| `MODES` | `modes.ts` | `Mode` | **KEEP**; declare with `satisfies Record<Mode, …>` over the leaf ids (Phase 2); drop the scenery wiring (Phase 5) |
| **`MODE_TRAITS`** | *(missing: `'tdm'`/`'ffa'` comparisons)* | `Mode` | **ADD** (Phase 3) |
| `MODE_VIEWS` | *(missing: branches in HUD/Results)* | `Mode` | **ADD** (client-only, UI level, Phase 5) |
| `MODE_SCENERY` | *(missing: `createHotZone` wired inside `modes.ts`)* | `Mode` (partial) | **ADD** (client-only, view level, Phase 5); only FFA's hot zone needs it now, since pickups are drawn generically from `mode.supply` |
| `DIFFICULTIES` | `ai.ts` | `Difficulty` | **KEEP**; move to a leaf data module (Phase 2) |
| `ITEMS`, `GROUPS`, `SUPPLY` | `items/` | `ItemType`, group | **KEEP** |
| `CUSTOM` (the owner's choices) | `matchSettings.ts` | – | **KEEP**; `CUSTOM.sizes` and the friendly-fire rule come from traits (Phase 3) |
| `LOBBIES` (lobby service numbers) | `server/custom.ts` | – | **KEEP** (the matcher's `MATCHMAKING` pattern) |
| `CUES`/`LOOPS` | `audio.ts` | `SoundCue` | **KEEP** |
| `LIVERIES`, `QUALITY`, `FACADES` | various | – | **KEEP** |

**Why plain records and not a `register()` API:** `Record<Id, …>` makes a missing entry a **compile error**. It needs no import-order or side-effect-import tricks, and it survives both Vite bundles and the plain-node checks unchanged. Self-registering modules (`registry.add(...)` at import time) would depend on import order and tree-shaking, and could silently drop content from the server bundle. **"Feature folder + one explicit line in an index file" is the right design.** [High confidence]

Adding content therefore touches **the feature folder + 1–2 registry lines (+ the traits line for a mode; + `digests.json` for a map)**. It never touches the engine.

---

## 13. Testing architecture

### 13.1 The convention: keep it

Plain-node `.check.ts` + Vite-bundled server checks. No Vitest/Jest (Vitest remains installed and unused: owner's call). [High confidence]

### 13.2 What exists now

| Check | Count (rev 1 → rev 2) | What the custom work added |
|---|---|---|
| `ai.check` | router | – |
| `bots.check` | 21 → 21 | – (kills a minute unchanged: 7.8 / 13.8 / 17.8) |
| `ffa.check` | 143 → 152 | respawn bands as shares, kill limit, ammo alone, every pickup off, own rocket |
| `tdm.check` | 163 → 182 | friendly-fire credit, team kills, overtime team kill, kill limit, slow respawn, `clearOfMates`, empty seat |
| `simulation.check` | 28 → 58 | sizes 2 and 12, clock length, friendly fire, own rocket, TDM pickups, empty seat, **2 golden pins (yard)** |
| `loading.check` | 23 | – |
| `protocol.check` | 70 → 90 | settings validator, every lobby message's rules, presence round trip, Classic rows unchanged, `readCode` |
| `chat.check` | 14 | – |
| `matchmaker.check` | 63 | – |
| **`custom.check`** | new, 71 | every lobby action and refusal, sides, owner hand-off, grace, idle close, caps, codes, passwords and lockout, bans, the tally, a list without secrets |
| `fairplay.check` | 10 | – |
| `arena.check` | 64 → 72 | six starts a base, all clear; new digests |
| `server.check` | 133 → 166 | **4 golden pins (real arenas)**, custom rooms (seat plan, empty seats, end to the lobby, abandonment, replay to the bit), custom lobbies over real sockets, idle minute → waiting room, no secrets in logs |
| `client.check` | 32 → 47 | the custom store headless: create, join by code, start, Back to lobby, the end, drops, reloads, too late, invite link, kicked |
| `netplay.check` | 33 → 33 | – |

### 13.3 What is still missing

| New check | Why | Phase |
|---|---|---|
| **Golden wire fixtures** in `protocol.check` | Pins hash end-of-match *state*; nothing pins *bytes*. Fix a combatant (and an absent one), pack a snapshot frame; serialise a welcome with settings, a `ro`, an input, a `LobbyRow`/`LobbyView`; compare with committed fixtures. Phases 2 and 4 move wire code. | **0** |
| **A pin for non-Classic settings** | Custom options are checked by assertions, not pinned: a refactor could change how a 12-seat friendly-fire TDM with pickups plays without failing anything. One whole match: TDM, 12 seats, friendly fire, all pickups, kill limit 25, fast respawn, one seat emptied and taken mid-match; and FFA at 2 seats, pickups off. | **0** |
| **Boundaries check** with ratchets | §14 | **1** |
| **Every `.check.ts` actually runs** | The repo's review found claimed checks that did not exist. A rule in `boundaries.check`: every `*.check.ts` file is invoked by `npm run check` / `server:check` (and bundled ones are in `vite.server.config.ts` `ENTRIES`). Cheap, and it turns "the log says" into "the script runs". | **1** |
| Map × mode matrix at the largest size | `arena.check`: every hosted mode lines up at its largest traits size and builds headless | 1 |
| Traits consistency | `MAX_SEATS` equals `SLOTS`, HUD pools and `BOT_NAMES.length`; `CUSTOM.sizes` equal traits | 3 |
| Lobby rules parity | `startable`/`tallyKey`/`sideOf` against the cases `custom.check` already drives | 3 |
| Codec round trip | Every wire event code encodes and decodes to itself; `owners()` agrees with today's `mine()` (§9.4) | 4 |
| `inputs.check` | The input queue policy (drain window, stale → neutral, repeats and drops) once it is its own module | 7 |

### 13.4 The flaky timing case

`netplay.check`'s rough-link case ("the queue is back to 1 within about a second after a stall's burst") measures wall-clock time with real sockets and timers. The repo's log records it failing in several runs on a loaded machine (1.1–3.9 s against a ~1 s bound) and passing in others; it passed in my run (676 ms). **Do not skip it.** Recommended (Phase 7, Medium confidence): measure the drain in server **steps** rather than milliseconds (the server's loop is fixed-step), or drive the latency simulator's clock by hand as `matchmaker.check` does. Until then, a red netplay run on CI needs one look at this case before anything else.

### 13.5 Determinism and replay

Unchanged layers, now stronger: same-process determinism; **whole-match pins across commits (done)**; room = practice; replay equality (now including custom rooms with seat plans); arena digests. Cross-machine stability of the pins is now shown (macOS arm64 Node 26.7 and 24.21 by the repo; Linux x64 Node 22.22 by me). A Rapier or V8 upgrade may still legitimately move them: re-pin in a separate commit with the reason, as the repo did. [High confidence]

---

## 14. Dependency enforcement

| Option | Enforces | Cost | Verdict |
|---|---|---|---|
| **TypeScript path aliases** (`@sim/…`) | nothing by itself (cosmetic) | **Breaks the plain-node checks**: Node's type stripping does not resolve `tsconfig` paths without a loader | **Reject** [High confidence] |
| **oxlint `no-restricted-imports` + `overrides`** | per-folder import bans | Config only. The installed oxlint (1.85) has `overrides` in its schema; **rule support needs verification.** It cannot express "reachable from the server entry" (transitive), "no `Math.random` in sim", or a ratchet. | **Optional complement** (Medium) |
| **dependency-cruiser** | full graph rules | A new dev dependency + a config DSL; the repo's rules favour no new dependencies | **Reject for now** (Medium) |
| **Separate packages/workspaces** | hard boundaries | Breaks the one-package Vite setup, the build id, the shared server bundle and the checks | **Reject** [High confidence] |
| **Per-layer `tsconfig` projects** (e.g., `sim` without the DOM lib) | compile-time "no DOM in sim" | `@types/three` references DOM types; `skipLibCheck` may make it workable. **Needs a spike.** | **REVIEW (optional, later)** (Low) |
| **Custom node check** (`boundaries.check.ts`) | folder/file rules, **transitive reachability** from server entries, banned globals per folder, cycles, **ratchets**, "every check runs" | A few hundred lines at most, no dependency; runs in `npm run check` | **Recommend** [High confidence on the need; Medium on the exact mechanism] |

The custom work strengthens the case for a check over prose: the repo's own review found agent-written log entries describing checks that did not exist (§0.2, row 2). A rule that the script runs cannot be claimed into existence.

### 14.1 Rules

Written for the target folders; before Phase 6 they are explicit file lists. Each rule below passes on today's code, with the allowances named (I checked each against the `3b0943d` graph).

1. **No value-import cycles** (today: 0). **Type-level cycles: a ratchet**: today 1 strongly connected set of 14 files; the number may only go down (Phase 2 takes it to 0).
2. **Server reachability:** from `server/{main,load,replay-main}.ts`, the value-import closure excludes `view/**`, `runtime/**`, `screens/**`, `hud/**`, client-only mode files (`modes/views.ts`, `modes/scenery.ts`, `modes/*/{hud,results}.tsx`, `modes/items/pickups.ts`, `modes/ffa/zone.ts`), `net/{connection,client,prediction,snapshots,matchmaking,custom,session,chat,sessionSocket}.ts`, React and Nakama JS. **Allowance until Phase 5:** `items/pickups.ts` and `ffa/zone.ts`, which the server reaches today through `modes.ts`. `server/*.check.ts` and `server/browser.ts` are exempt (they emulate pages on purpose).
3. **Gameplay purity:** `sim/**`, `shared/**` and the domain files of `modes/**` (not `*.check.ts`, not scenery or panels) must not import `render/**`, `view/**`, `runtime/**`, `net/**`, React or `three/examples/**`, and their source must not contain `Math.random(`, `Date.now(`, `performance.now(`, `window.`, `document.` or `localStorage`. **Allowance until Phase 2:** `loadout.ts` (its `localStorage` restore moves to `runtime/stored.ts`).
4. **Pure services:** `server/matchmaker.ts` and `server/custom.ts` must not import `ws`, `node:http`, `node:net`, `room.ts` or `lobby.ts`, and must not call `setTimeout`/`setInterval` (they take a clock). Both pass today.
5. **Mode literals: a ratchet.** Outside `modes/<mode>/` (today `game/ffa/`, `game/tdm/`), the traits module and the registry, a line comparing against `'tdm'`/`'ffa'` is counted per file; the counts may only go down. Today: `LobbyForm.tsx` 8, `Lobby.tsx` 7, `matchSettings.ts` 6, `server/custom.ts` 6, `Hud.tsx` 4, `Results.tsx` 2, `room.ts` 1 (34 in all; `maps.ts`' data lists and `App.tsx`'s default pick are allowed). After Phase 3 the server files reach 0; after Phase 5 the HUD and results reach 0.
6. **UI physics ban:** `screens/**` and `hud/**` import no Rapier, no `sim/physics`, no `sim/simulation` values (types are allowed).
7. **Import-time safety:** modules with import-time browser side effects (today `audio.ts`, which runs `window.addEventListener` at import) are reachable only from `view/`, `runtime/`, `screens/`, `hud/`.
8. **Every `*.check.ts` runs:** each one is invoked by `npm run check` or `server:check`, and each bundled one is in `vite.server.config.ts` `ENTRIES`. Today all 15 are.
9. **Invariants as code:** `RATE.step * PHYSICS_STEP === 1`; `MAX_SEATS` equals `SLOTS`, the HUD pools and `BOT_NAMES.length` (from Phase 3).
10. **Layer direction:** a module imports from its own level or lower (§4.2, §20). Before Phase 6 this is the file-list form of rules 2, 3 and 6; after it, folder globs.

A ratchet is a small object in the check file (`{ 'src/screens/Lobby.tsx': 7, … }`): a file over its count fails; a file under it prints "lower the allowance". It lets the check land **enforcing** on today's code without a big cleanup first. The check should print a readable table of violations, and must handle `export … from` and dynamic `import()` (today only `main.tsx → App`).

---

## 15. File-by-file migration map

Legend: **keep** · **move** (`git mv` + import paths) · **split** · **edit** (behaviour-preserving) · **new**. Phases refer to §16. Paths are under `game/src/` unless they start with `server/`.

### 15.1 Simulation core → `sim/`, `shared/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/simulation.ts` | `sim/simulation.ts` | move | 6 | Authoritative core |
| `game/combat.ts` | `sim/combat.ts` + `content/weapons/{types,index}.ts` + per-weapon `spec.ts` | edit (2) → split (6) | 2, 6 | Identity; definitions vs mechanics |
| `game/physics.ts`, `game/vehicle/drive.ts` | `sim/physics.ts`, `sim/drive.ts` (types → `content/vehicles/types.ts`) | move | 6 | |
| `game/rng.ts` | `shared/rng.ts` | move | 6 | |
| `game/scoring.ts` | `sim/scoring.ts` | move | 6 | |
| `game/mode.ts` | `sim/mode.ts` (no `show`, `SupplyView`; `ms`/`clock` → `shared/`) | edit (2, 5) → move (6) | 2, 5, 6 | Contract; cycle fix |
| `Point` ×3 | `shared/types.ts` | new (type-only) | 2 | Dedupe |
| `game/ai.ts` `DIFFICULTIES` | `sim/difficulty.ts` (pure data) | split | 2 | The wire must not import the AI |
| `game/ai.ts` (rest) | `sim/ai/{brain,perception,navigation,think}.ts` | move (6) → split (7) | 6, 7 | Responsibilities; RNG order |
| `game/loadout.ts` | `sim/loadout.ts` + `runtime/stored.ts` | split | 2 | Pure parse shared with `parseClient` |
| checks | follow their subject | move | 6 | |

### 15.2 Modes → `modes/`

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| – | `modes/traits.ts` (ids, `MODE_TRAITS`, `MAX_SEATS`, `sideOf`) | new | 2 (ids), 3 (traits) | Leaf ids break the cycle; traits replace mode literals |
| `game/matchSettings.ts` | `modes/settings.ts` | edit (3: traits) → move (6) | 3, 6 | Per-mode parts from traits |
| `game/modes.ts` | `modes/index.ts` (+ `satisfies Record<Mode, …>`) + `modes/views.ts` + `modes/scenery.ts` | split | 2, 5, 6 | Ids out; client registries |
| `game/roster.ts` | `modes/roster.ts` (`BOT_NAMES` sized by `MAX_SEATS`, checked) | move | 6 | |
| `game/items/{config,items,supply}.ts` | `modes/items/…` | move (+ `share` effects loop, 5) | 5, 6 | Shared by modes |
| `game/items/pickups.ts` | `modes/items/pickups.ts` (client-only; drawn generically from `mode.supply`) | move + edit | 5, 6 | Out of the registry |
| `game/ffa/{config,rules,mode}.ts`, `ffa.check.ts` | `modes/ffa/…` (+ `FFA_TRAITS` in `config.ts`) | move | 3, 6 | Feature folder |
| `game/ffa/zone.ts` | `modes/ffa/zone.ts` (client-only, via `MODE_SCENERY`) | move | 5, 6 | |
| `game/tdm/*` | `modes/tdm/…` (+ `TDM_TRAITS`; `lineUp` uses `sideOf`) | move | 3, 6 | |
| – | `modes/{ffa,tdm}/{hud,results}.tsx` | new (extracted) | 5 | Per-mode panels |

### 15.3 Content, render, view, runtime

**Content → `content/`** (weapon specs: see `game/combat.ts` in §15.1)

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/vehicle/vehicles.ts` | `content/vehicles/index.ts` (`VEHICLES`) + `content/vehicles/razor/spec.ts` + `content/vehicles/types.ts` | split | 6 | Vehicle feature folder |
| `game/vehicle/vehicle.ts` | `content/vehicles/razor/model.ts` + `content/vehicles/models.ts` (`MODELS`) | edit (turret via `TURRETS`) → split | 2, 6 | Removes the turret ternary |
| `game/vehicle/parts.ts` | `content/parts.ts` (wheel, tyre, blade, lamp, spike) + `content/weapons/{minigun,rocketPod}/turret.ts` + `content/weapons/turrets.ts` | split | 2, 6 | Turrets belong to weapons; parts are shared by vehicles and arenas |
| `game/arena/arena.ts` | `content/arenas/arena.ts` | move | 6 | Contract + kit helpers |
| `game/arena/digest.ts` | `content/arenas/digest.ts` | move | 6 | Pure |
| `game/arena/{props,ground,buildings,street}.ts` | `content/arenas/kit/…` | move | 6 | Shared by both maps |
| `game/arena/scrapyard.ts` | `content/arenas/scrapyard/scrapyard.ts` | move | 6 | Feature folder |
| `game/arena/city.ts` | `content/arenas/city/city.ts` | move | 6 | Feature folder |
| `game/maps.ts` | `content/arenas/index.ts` (`MAPS`, `MapId`, `mapsFor`); `loadArena` → `runtime/runtime.ts` | split | 6 | The registry is content; the session cache is runtime |
| `game/{VehicleGenerator,ArenaGenerator,proceduralTexture}.ts` | `content/legacy/` or delete | **owner decision** | 9 | Unused; `AGENTS.md` keeps them as reference |

**Render infrastructure → `render/`**

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/renderer.ts` | `render/renderer.ts` | move | 6 | The one WebGL context; lazy |
| `game/materials/*` | `render/materials/*` | move | 6 | Shared GPU resources; server-reachable (headless path) |
| `game/geometry.ts` | `render/geometry.ts` | move | 6 | Geometry helpers + `disposeGeometries` |
| `game/environment.ts` | `render/environment.ts` | move | 6 | Sky/sun |
| `game/postprocessing.ts` | `render/postprocessing.ts` (owns the `Quality` type) | move | 6 | Fixes the direction of the `settings` ↔ `postprocessing` type edge |

**Presentation → `view/`**

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/view.ts` | `view/view.ts` | edit (`spec.cue`) → move | 2, 6 | – |
| `game/pilot.ts`, `game/input.ts`, `game/camera.ts` | `view/…` | move | 6 | The local control source + camera |
| `game/feed.ts` | `view/feed.ts` | move | 6 | Implements `Feed` (now with team kills) |
| `game/effects.ts`, `game/audio.ts`, `game/sounds.ts` | `view/…` | move | 6 | `audio.ts` has import-time side effects: boundary rule 7 |
| `game/settings.ts` | `view/settings.ts` | move | 6 | Read by camera, audio, HUD and runtime |
| `game/turntable.ts` | `view/turntable.ts` | move | 6 | Garage stage |

**Runtime → `runtime/`**

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `game/runtime.ts` | `runtime/runtime.ts` | move | 6 | Fixes the game↔net inversion |
| `game/match.ts` | `runtime/match.ts` (`playMatch`, `MatchSource`, `Match`) + `runtime/practice.ts` (`createMatch`, which plays `classic(mode)`) | move → split | 6, 7 | Symmetry with `online.ts` |
| `game/online.ts` | `runtime/online.ts` | move | 6 | – |
| `game/loading.ts`, `loading.check.ts` | `runtime/…` | move | 6 | – |

### 15.4 Net

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `net/protocol.ts` | same path | edit: `weaponId → spec.id`, `Seat` → `SeatInfo`, `SLOTS = MAX_SEATS`, `parseLoadout`; lobby part out | 2, 3, 4 | Identity; one constant; split |
| – | `net/lobbyProtocol.ts` | new (moved out of `protocol.ts`) | 4 | Lobby wire on its own |
| – | `net/lobbyRules.ts` | new | 3 | One home for rules both sides use |
| – | `net/events.ts` | new | 4 | Codec |
| – | `net/sessionSocket.ts` | new (shared dial/retry/mark) | 7 | Two stores, one machinery |
| `net/client.ts` | same | edit (decode via codec) | 4 | |
| `net/matchmaking.ts`, `net/custom.ts` | same | edit (use `sessionSocket.ts`) | 7 | |
| others | same | keep | – | |

### 15.5 UI and server

| Current | Proposed | Action | Phase | Reason |
|---|---|---|---|---|
| `hud/Hud.tsx` | shared HUD + mode panels in `modes/*/hud.tsx`; pools from `MAX_SEATS`; chips from `ITEMS`; weapon icon from data (+ optional `hud/{compass,vitals,weapon,feed,markers,scoreboard}.ts`) | edit/split | 2, 3, 5, 7 | Mode branches out; no hard-coded weapons |
| `hud/minimap.ts` | marks and colours passed in by the mode panel / the supply | edit | 5 | Drops the `ffa/rules` type import |
| `hud/Chat.tsx` | same | keep | – | Also docked in the waiting room; cohesive |
| `screens/Results.tsx` | generic frame + `modes/*/results.tsx`; tally key from `lobbyRules` | split/edit | 3, 5 | Mode branches out; one tally rule |
| `screens/Lobby.tsx` | uses `sideOf`/`startable`/traits; later `screens/custom/{SlotGrid,SettingsCard,InviteCard}.tsx` | edit → split | 3, 7 | Two server rules re-implemented; four concerns |
| `screens/LobbyForm.tsx` | defaults and field visibility from traits | edit | 3 | 8 mode-literal lines |
| `screens/{Custom,Lobbies,Lobby,LobbyForm}.tsx` | optionally `screens/custom/` | move | 6 | Feature grouping |
| `screens/GameCanvas.tsx` | same + optional `screens/game/{PauseMenu,ExitConfirm,LoadingOverlay}.tsx` | REVIEW split | 7 | Readability |
| `screens/Garage.tsx` | same; vehicle pager | edit | 8 | Second-vehicle capability |
| other `screens/*`, `App.tsx`, `main.tsx`, `analytics.ts` | same | keep (import paths only) | 6 | – |
| `server/custom.ts` | same; reads traits + `lobbyRules`; imports `lobbyProtocol` | edit | 3, 4 | No mode literals |
| `server/room.ts` | `room.ts` + `server/journal.ts` + `server/inputs.ts` (+ optional `server/matchRecord.ts`); custom options grouped; team chat from traits | edit/split | 3, 7 | A persisted format and a tuned policy get their own modules |
| `server/replay.ts` | same; imports `journal.ts` | edit | 7 | One definition of the format |
| `server/recorder.ts` | same; encodes via `net/events.ts` | edit | 4 | Shared codec |
| `server/server.ts` | same + optional `server/netsim.ts` | REVIEW split | 7 | Dev tool vs production door |
| `server/lobby.ts` | same | keep (REVIEW) | – | Extract the custom hooks only with a third flow |
| `server/{matchmaker,fairplay,records,rewind,auth,arenas,headless,main,load,replay-main,browser}.ts`, `digests.json` | same | keep | – | Cohesive |

### 15.6 Non-source files that reference paths

| File | Reference | Phase |
|---|---|---|
| `game/package.json` `check` script | the `src/game/*.check.ts` paths (`ai`, `bots`, `ffa`, `tdm`, `simulation`, `loading`) | 6 |
| `game/vite.server.config.ts` `ENTRIES` | server check paths (unchanged unless a new bundled check file is added; the Phase 0 pins go into existing files, so none is expected) | 0 (only if needed) |
| `scripts/arena-parity.mjs` | `src/game/maps.ts`, `physics.ts`, `arena/digest.ts` | 6 |
| `.claude/work/ffa/ffa-metrics.js`, `.claude/work/tdm/tdm-metrics.js` (balance probes) | `import('/src/game/ai.ts')`, `combat.ts`, `ffa/config.ts`, `tdm/config.ts` | 6 |
| `scripts/match-smoke.mjs` | reads `PROTOCOL` from `game/src/net/protocol.ts` with a regex: **that path does not move** | – |
| `scripts/browser-match.mjs` (`custom` mode too) | no source paths; it drives the UI by accessible names (`Play`, `+ Create lobby`, `Advanced`, `More players`, `Create lobby`, `Ready`, `Start match`) and reads `window.match` | 5, 7 (UI splits must keep those names) |
| `game/AGENTS.md` code map, root `AGENTS.md`, `.claude/work/arch/*.md`, `.claude/work/net/NET_ARCHITECTURE.md` | many paths | every phase that moves files |
| `.claude/work/custom/PLAN.md` | paths in its file map | 3, 4, 6 |

---

## 16. Migration phases

Global invariants for **every** phase (the definition of "behaviour-preserving"):

- `npm run lint` (0 warnings), `npx tsc -b`, `npm run build` and `npm run check` are all green.
- **The six golden pins unchanged** (and, from Phase 0 on, the two custom pins and the wire fixtures). `server/digests.json` unchanged. **`PROTOCOL` stays 6** (except Phase 8).
- The build id changes with every PR. That is expected: pages open across a deploy are told to reload.
- One concern per PR; moves never share a PR with content edits.
- Every PR updates the docs it invalidates, including `.claude/work/custom/PLAN.md` where paths move.

### Phase 0: Close the characterization gaps (test-only)

| | |
|---|---|
| **Goal** | Pin what the repo's pins don't: wire bytes and non-Classic play. |
| **Files affected** | `src/net/protocol.check.ts` (fixtures), `server/server.check.ts` or `src/game/simulation.check.ts` (custom pins), optional `server/arena.check.ts` (map × mode at the largest size) |
| **What changes** | Committed fixtures for a snapshot frame with an absent row, a welcome with settings, a `ro`, an input, a `LobbyRow`/`LobbyView`. Two whole-match pins with custom settings (§13.3). Optionally: every map × every mode it hosts, lined up at the largest size a lobby may choose, built headless. |
| **What must NOT change** | Production code; the six existing pins; `digests.json`. |
| **Tests/checks** | The new checks green locally, on CI and on one other machine (the pins must be platform-stable, as the existing ones are shown to be). Mutation test in review: flip one clamp in `packCars` → a fixture fails. |
| **Expected risk** | Low. |
| **Rollback** | Revert. |
| **Definition of done** | Fixtures and the two custom pins committed and run by `npm run check`. |

### Phase 1: Boundary check with ratchets (test-only)

| | |
|---|---|
| **Goal** | Mechanically hold today's boundaries and stop new leaks. |
| **Files affected** | New `game/boundaries.check.ts`; `game/package.json` (`check` script) |
| **What changes** | Rules 1–10 of §14.1 on today's file lists, with ratchets for type cycles (1 set) and mode literals (34 lines), and the two named allowances (server-reachable scenery until Phase 5; `loadout.ts` until Phase 2). |
| **What must NOT change** | Production code. |
| **Tests/checks** | Passes on today's code. Negative tests in review, each failing with a clear message: an `audio.ts` import into `simulation.ts`; a new `'tdm'` in `server/custom.ts`; a `setTimeout` in `server/custom.ts`; a check file dropped from `package.json`. |
| **Expected risk** | Low (false positives from `export … from` or dynamic `import()`: handle both). |
| **Rollback** | Remove it from the `check` script. |
| **Definition of done** | In `npm run check`; `MODULE_BOUNDARIES.md` points to it as the source of truth. |

### Phase 2: Identity and leaf types (behaviour-preserving)

| | |
|---|---|
| **Goal** | Fix the weapon identity defect; make weapons plug-and-play; put the ids and shared data at the bottom of the graph; break the type cycle. |
| **Files affected** | `combat.ts` (`id`, `turret`, `icon`, optional `cue`/`ai`); `ai.ts` (`botGun` keeps `id`; `Skill`/`DIFFICULTIES` → a leaf); `net/protocol.ts` (`weaponId` → `spec.id`; `Seat` → `SeatInfo`; `pick()` → `parseLoadout`); `loadout.ts` (`parseLoadout`; storage → `stored.ts`); `vehicle/vehicle.ts` + `parts.ts` (`TURRETS`); `hud/Hud.tsx` (icon from data); `view.ts` (`spec.cue`); a new leaf ids module (`Mode` ids); `modes.ts` (`satisfies Record<Mode, …>`); `matchSettings.ts` (`Mode` and `DIFFICULTIES` from leaves); `mode.ts` (`SupplyView`; `ms`/`clock` to a leaf); the `Point` dedupe; the `weaponId` callers (room, recorder, rewind, records, client) |
| **What changes** | In 3–4 PRs (§22): leaf ids and data; weapon identity; turrets and weapon data; the loadout parse. The type-cycle ratchet goes to 0. |
| **What must NOT change** | Wire bytes (the ids are the same strings), pins, visuals, RNG draws, `PROTOCOL` 6. |
| **Tests/checks** | All; the Phase 0 fixtures; a registry consistency check (every `WEAPONS[k].id === k`; every `turret` key in `TURRETS`); manual garage/HUD parity for both weapons. |
| **Expected risk** | Low–medium (several `weaponId` callers; a missed one is a type error once `model` is renamed). |
| **Rollback** | Revert per PR. |
| **Definition of done** | A scratch-branch third gun reusing the minigun turret touches only its spec + `WEAPONS` and reports its own id everywhere (wire, records, replays, the lobby's one-gun setting); `boundaries.check` reports 0 type cycles. |

### Phase 3: Mode traits and shared lobby rules (new)

| | |
|---|---|
| **Goal** | Code outside a mode's folder learns about modes from one table; the rules both sides use have one home. |
| **Files affected** | New `modes/traits.ts` (before Phase 6: `game/modeTraits.ts`, extending the Phase 2 leaf) with per-mode entries in `ffa/config.ts` and `tdm/config.ts`; new `net/lobbyRules.ts`; `matchSettings.ts` (`classic`, `CUSTOM.sizes`, `checkSettings`), `server/custom.ts` (side, free slot, regroup, slot claims, start rule, tally), `server/room.ts` (team chat), `tdm/mode.ts` (`lineUp` via `sideOf`), `net/protocol.ts` (`SLOTS`), `roster.ts` (`BOT_NAMES` checked against `MAX_SEATS`); then `screens/Lobby.tsx`, `LobbyForm.tsx`, `Results.tsx`, `hud/Hud.tsx` (pools) |
| **What changes** | Two PRs: **3a** server and domain; **3b** UI. The mode-literal ratchet goes down to the data lists only. |
| **What must NOT change** | Behaviour: pins, `custom.check` (71), `client.check` (47), `server.check`'s custom cases, the `protocol.check` fixtures; the server's refusals and their notes; the page's hints and highlights for today's two modes. |
| **Tests/checks** | All; a new traits-consistency check (`MAX_SEATS` = `SLOTS` = HUD pools = `BOT_NAMES.length`; `CUSTOM.sizes` from traits) and a lobby-rules parity check (`startable`/`tallyKey`/`sideOf` over the cases `custom.check` drives); manual waiting-room parity (hints, sides, tally). |
| **Expected risk** | Low–medium: regrouping on a mode change and the tally keys are subtle; `custom.check` covers them. |
| **Rollback** | Revert per PR. |
| **Definition of done** | No `'tdm'`/`'ffa'` comparison left in `server/`, `matchSettings.ts` or `screens/Lobby*.tsx`; a scratch-branch third-mode skeleton (`sides` 0, sizes 2–8) shows up in the lobby form and lines up without editing any shared file. |

### Phase 4: Wire modules: event codec and lobby protocol

| | |
|---|---|
| **Goal** | One definition per wire event; the lobby wire in its own module. |
| **Files affected** | New `net/events.ts` and `net/lobbyProtocol.ts`; `server/recorder.ts`, `net/client.ts` (`play`, `mine`), `net/protocol.ts` (`parseClient` delegates `lb`), `server/custom.ts`, `net/custom.ts` and the lobby screens' imports |
| **What changes** | A table of event kinds with field order, quantisation and which events play at once instead of at the drawn tick (today `mine()` and the `ru`/`go` exception, `net/client.ts:130`), used by both the encoder and the decoder. The lobby types, `LOBBY_ACTIONS`, `INVITE`, `readCode`, `tidy`, `LOBBY_FORM` and the `lb` parser move out of `protocol.ts`. |
| **What must NOT change** | Bytes and JSON (fixtures); `PROTOCOL` 6; event playback order; which events wait for the drawn tick. |
| **Tests/checks** | Fixtures; a codec round trip (every event kind encodes and decodes to itself); an old-vs-new decode table over a recorded match; `client`, `netplay`, `server` (replay). |
| **Expected risk** | Medium (positional indices). |
| **Rollback** | Revert. |
| **Definition of done** | No positional `f[n]` in `net/client.ts`; the lobby wire (≈120 lines) lives in `lobbyProtocol.ts`, and `protocol.ts` is back near its pre-lobby size. |

### Phase 5: Mode seam in the UI and scenery

| | |
|---|---|
| **Goal** | A third mode touches no shared UI file; rendering leaves the gameplay contract. |
| **Files affected** | `mode.ts` (drop `show`), `modes.ts` (drop the scenery wiring), `ffa/mode.ts`, `tdm/mode.ts` (no `scenery`/`pickups` parameter), the runtime (draws pickups from `mode.supply`; `MODE_SCENERY` for FFA's zone), `MODE_VIEWS` and the per-mode panels, `Hud.tsx`, `minimap.ts`, `Results.tsx`, the `supply.share` loop, the HUD chips from `ITEMS`, `net/client.ts` (`seatOnline` without a scene for the mode) |
| **What changes** | Three PRs: scenery out → panels → chips and the `share` loop. |
| **What must NOT change** | HUD/Results visuals and per-frame cost; pickup and zone visuals (pooled, pre-parked for the shader compile); restart/dispose order. |
| **Tests/checks** | All; manual visual parity (practice and custom: TDM with pickups, FFA, 12 seats, the friendly-fire TK column, results with the lobby tally); an allocation spot-check (a Chrome allocation timeline for 10 s). |
| **Expected risk** | Medium (manual UI parity). |
| **Rollback** | Revert per PR. |
| **Definition of done** | No `mode.kind ===` in `hud/` or `screens/`; no mode literal in `room.ts`; the server bundle no longer contains the pickups or hot-zone views (grep `dist-server`); `boundaries.check`'s Phase 5 allowance removed. |

### Phase 6: Folder reorganisation (mechanical)

| | |
|---|---|
| **Goal** | Folders express the boundaries; content becomes feature folders. |
| **Files affected** | Everything under `src/game/` (§15), import paths across `src/` and `server/`, `package.json`, `scripts/arena-parity.mjs`, the balance probes, docs (incl. `.claude/work/custom/PLAN.md`); optionally the lobby screens into `screens/custom/`. |
| **What changes** | `git mv` in **bottom-up order, one PR per layer:** (a) `shared/` + `render/`; (b) `content/` (with the vehicle/weapon/arena feature folders, incl. the `combat.ts` and `vehicles.ts` splits); (c) `sim/` (incl. `ai.ts` moved whole); (d) `modes/` (traits, settings, `items/`, `ffa/`, `tdm/`, registry, roster); (e) `view/`; (f) `runtime/` (removes `src/game/`). Then the boundaries check switches from file lists to folder globs. |
| **What must NOT change** | File contents beyond import specifiers (and the pre-agreed type/registry splits in (b)); the `.ts` import extension convention for node-run modules; `net/protocol.ts`'s path; digests; pins; bytes. |
| **Tests/checks** | Everything; plus `node scripts/arena-parity.mjs` locally (needs global Playwright, per the root `AGENTS.md`), one balance-probe run in the dev build to confirm the probe imports resolve, and `node scripts/browser-match.mjs custom`. |
| **Expected risk** | Low per PR but high churn: merge conflicts with in-flight work. Mitigation: announce a short freeze per layer PR; land each quickly. |
| **Rollback** | Revert the layer PR (pure moves revert cleanly). |
| **Definition of done** | `src/game/` no longer exists; the boundaries check uses folder rules; the `AGENTS.md` code map and `MODULE_BOUNDARIES.md` describe the new tree. |

> **Optional-phase note:** Phase 6 is the **least valuable per unit of risk**. If no third mode, second vehicle or new contributors are planned in the next few months, stop after Phases 0–5 and 7: they deliver the plug-and-play fixes and the enforcement without moving files. [Medium confidence]

### Phase 7: Split oversized modules and share helpers

| | |
|---|---|
| **Goal** | Single-responsibility modules where it pays. |
| **Files affected** | `ai.ts` → `sim/ai/*`; `server/room.ts` → `journal.ts`, `inputs.ts` (+ optional `matchRecord.ts`), grouped custom options; `server/replay.ts`; `net/sessionSocket.ts` shared by both stores; `screens/Lobby.tsx` split; `match.ts` → `practice.ts`; optional `GameCanvas.tsx` split, `server/netsim.ts`, `hud/*` helpers; the netplay timing case measured in steps |
| **What changes** | Code moves between files; **no statement reordering** inside `think()`, `room.step()` or `playMatch.frame()`; the stores keep their own state machines and only call shared helpers. |
| **What must NOT change** | RNG draw order; room step order; the journal line format; replay compatibility of journals written by the same build; the stores' reconnect behaviour (`RETRY`, the 15 s and 20 s graces, the `sessionStorage` marks). |
| **Tests/checks** | Pins; `bots.check`; `server.check` replay; a new `inputs.check`; `client.check` (47) and `scripts/browser-match.mjs custom` for the store helpers; manual: bots in practice look the same over a 2-minute match. |
| **Expected risk** | Medium (the AI split is the riskiest file-level change in this plan; the store helpers touch the layer where the repo's review found four bugs). |
| **Rollback** | Revert per module PR. |
| **Definition of done** | No production file over ~450 lines except content builders (`city.ts`, `scrapyard.ts`, `recipes.ts`) and the cohesive rules files; `netplay.check`'s rough-link case no longer depends on wall-clock load. |

### Phase 8: Second-vehicle capability (a feature, not a refactor; only when scheduled)

| | |
|---|---|
| **Goal** | Seats honour `loadout.vehicle`; bots may draw vehicles; the garage pages vehicles; a lobby's seat plan carries vehicles. |
| **Files affected** | `server/room.ts` (`join`/`leave`/`takeWheel`: replace the car body in place: free the old body, `createCar`, `placeCar` at the current pose; a custom seat's vehicle from the plan), `net/protocol.ts` (`ro` + vehicle, **`PROTOCOL` 7**), `server/journal.ts` (`join` + vehicle), `net/client.ts` (roster refit + body swap), `view/view.ts` (`refit` by vehicle), `modes/roster.ts` (`SeatPlan` + vehicle; the bots' vehicle draw from its own stream), optionally a `vehicles` match setting (one vehicle for everyone, like `weapons`), `screens/Garage.tsx` (pager), `server/arena.check.ts` (every vehicle's shells), `server/server.check.ts` (replace the one-vehicle tripwire at line 368 with a two-vehicle test using a check-only spec) |
| **What changes** | Behaviour changes on purpose, so **re-pin the golden pins in a separate commit**, with the reason, the way the repo re-pinned for team kills. |
| **What must NOT change** | Practice/online parity; replay equality within a build; the rewind (it reads `chassis.shells` per machine); Classic's line-up when everyone keeps the default vehicle. |
| **Tests/checks** | All; a new mid-match takeover with a different vehicle in `server.check`; a custom room with mixed vehicles replaying to the bit; netplay with the second vehicle. |
| **Expected risk** | **High**: Rapier body replacement mid-match (handles, colliders, the vehicle controller, the prediction state machine). |
| **Rollback** | Revert; `PROTOCOL` goes back to 6 with the revert. |
| **Definition of done** | A person can pick vehicle B in the garage and play it online, in Classic and in a custom lobby; bots use both; replays reproduce it. |

### Phase 9: Documentation and cleanup

| | |
|---|---|
| **Goal** | Docs match the code; decide the loose ends. |
| **Files affected** | `game/AGENTS.md`, root `AGENTS.md` (repo layout line), `.claude/work/arch/*` (`ARCHITECTURE.md`, `MODULE_BOUNDARIES.md`, `STATE_OWNERSHIP.md`, `GAME_LOOP.md`, `LOADING_ARCHITECTURE.md`), `.claude/work/net/NET_ARCHITECTURE.md` ("Adding content, online"), `.claude/work/custom/PLAN.md` (paths). Owner decisions: the legacy generators, the Vitest dev dependency. |
| **What changes** | Docs; possibly removing unused files and dependencies (owner's call). |
| **What must NOT change** | Code behaviour. |
| **Tests/checks** | All green; the `boundaries.check` rules quoted verbatim in `MODULE_BOUNDARIES.md`. |
| **Expected risk** | Low. |
| **Rollback** | Revert. |
| **Definition of done** | The "How to add …" sections list exactly the steps in §6 of this plan. |

**Why this order.** Phases 0–1 are test-only and protect everything after them. Phase 2 fixes a real defect and puts the ids at the bottom of the graph, which Phase 3 needs. Phase 3 removes the leaks the custom work introduced while they are few and fresh, before a third mode or a lobby feature copies them. Phases 4–5 finish the seams; 6–7 are housekeeping; 8 waits for a product decision.

---

## 17. Behaviour that must not change

| Behaviour | Where it lives | How it is verified |
|---|---|---|
| Practice match (player + 7 bots, difficulty) and Classic online, exactly as before | `createMatch` with `classic(mode)`, roster, sim, rules | the six golden pins; `simulation.check` (58); manual smoke |
| FFA (rules, pickups through the supply, hot zones, overtime, standings, nemesis) | `ffa/*`, `items/*` | `ffa.check` (152); pins; manual HUD/Results |
| TDM (team score, protection ends on firing, MVP, overtime; pickups only when a lobby turns them on) | `tdm/*`, `items/*` | `tdm.check` (182); pins; manual |
| Bots (targeting, routing, cover, ambushes, per-difficulty skill, weapon draw; holding fire near teammates with friendly fire) | `ai.ts`, `roster.ts`, tactics | `bots.check` (21; kills a minute 7.8 / 13.8 / 17.8 within its ranges); `tdm.check` (`clearOfMates`); pins |
| Vehicle physics (ray-cast car, upend recovery, stuck recovery) | `drive.ts`, `simulation.ts` | pins; `simulation.check` "it drives"; `netplay.check` (prediction error) |
| Weapons (fire rate, magazine, reload, spread, rockets, blast shove/falloff) | `combat.ts`, `simulation.ts` | pins; `simulation.check`; `server.check` fire-rate authority |
| Combat, damage, protection, wrecks; friendly fire decided by the rules | sim + rules | same; `tdm.check` friendly-fire credit |
| Scoring and statistics (incl. `teamKills`) | `scoring.ts` + rules | rules checks; `server.check` records |
| Respawn (waits as shares of the clock, the respawn-speed setting, spawn choice with sight lines) | `respawnWait`, rules `pickSpawn`, sim `respawn` | rules checks; pins |
| Map loading (session cache, headless parity) | `maps.ts`, `server/arenas.ts` | `arena.check` (72; digests `227c4ce7` / `8913ad26`); `scripts/arena-parity.mjs` (manual) |
| Loadout (ids, persistence, server fallback for unknown ids) | `loadout.ts`, `parseClient` | `protocol.check` (90); manual reload |
| Classic matchmaking (queue, groups, ready check, grace, backfill, requeue) | `matchmaker.ts`, `lobby.ts`, `net/matchmaking.ts` | `matchmaker.check` (63); `server.check` (166) |
| Online match (authority, forged/stale input, hold for page loads, next match) | `room.ts`, `server.ts` | `server.check` |
| Prediction/reconciliation | `prediction.ts`, `client.ts` | `netplay.check` (33), `client.check` (47) |
| Interpolation (`DELAY` 4 ticks, events at the drawn tick) | `snapshots.ts`, `client.ts` | `netplay.check` |
| "Reconnect", Classic: a **search** survives a dropped socket or a reload for 15 s (grace + `sessionStorage`); a dropped *match* socket hands the seat to a bot, with no mid-match rejoin of the same seat | `matchmaker.ts`, `net/matchmaking.ts` | `server.check` "a drop and its grace"; manual reload during a search |
| "Reconnect", custom (new): a member keeps their slot, and in a match their seat, for 20 s; coming back gets a fresh welcome for the held seat; a reload resumes; too late → the list with the reason | `server/custom.ts`, `net/custom.ts`, `App.tsx` | `client.check`; manual in Chrome (the repo's review did this with Nakama up) |
| Replay (room journal → identical records, fair-play counts included; custom rooms with settings, lobby and seat plan in the header) | `room.ts`, `records.ts`, `replay.ts` | `server.check` replay (Classic and custom); the `replay.js` tool |
| Chat (channels from the welcome, whispers by uid, typing never drives; a lobby's channel carries on into its match; chat docked in the waiting room) | `net/chat.ts`, `hud/Chat.tsx`, `room.ts` channels, `custom.ts` | `chat.check` (14); `scripts/chat-smoke.mjs` (needs Nakama); manual |
| Fair-play tracking | `fairplay.ts`, `room.watch` | `fairplay.check` (10); `server.check` records |
| Build/protocol compatibility | `PROTOCOL` (6), `BUILD`, `digests.json` | `server.check` door; deploy smoke (`scripts/match-smoke.mjs`) |
| Loading (progress = finished/total, retry, cancel, failure paths) | `loading.ts`, `runtime.ts` | `loading.check` (23) |
| Dev tooling (`window.match`, `window.camera`, `window.tick`, F3 overlay lines) | `runtime.ts`, `match.ts` | manual; the balance probes and `browser-match.mjs` rely on the `window.match` shape |
| Match settings: size 2–12 (TDM even), duration, respawn ×0.5/1/1.5, friendly fire + team kills, pickup groups in both modes, one gun for everyone, kill limit | `matchSettings.ts`, rules, roster, room | `ffa.check`, `tdm.check`, `simulation.check`, `server.check`; the Phase 0 custom pins |
| Empty seats: out of play, a quiet leave, a protected entry at a rules-picked start, invisible everywhere on the page | sim, rules, rewind, client, view, HUD | `simulation.check`, `tdm.check`, `server.check`, `client.check` |
| Lobby list: public lobbies only, no secrets, sent at most twice a second | `custom.ts` | `custom.check` (71) |
| Create; join from the list (password + lockout), by code or by link (no password); bans; kick; owner hand-off (longest present); deletion when the owner leaves alone; idle close (30 min); caps (`MAX_LOBBIES`) | `custom.ts` | `custom.check`, `server.check` |
| Waiting room: ready; slot claims (first wins, ready kept); bots per slot at a difficulty; edit (unreadies everyone, regroups, resets the tally on a mode change) | `custom.ts`, `Lobby.tsx` | `custom.check` |
| Start: ≥ 2 people, every other person ready and connected, TDM someone on each side, a room free | `custom.ts` | `custom.check` |
| A lobby's match on the members' lobby sockets; join in progress; Back to lobby; the end → waiting room and the tally (TDM by side, FFA by player or bot slot; draws and abandoned matches count nothing); abandoned after 10 s with nobody seated; a minute idle → waiting room | `lobby.ts`, `room.ts`, `custom.ts`, `net/custom.ts` | `server.check`, `client.check`, `scripts/browser-match.mjs custom` |
| `?join=` invite links; StrictMode double mounts in development | `net/custom.ts`, `App.tsx` | `client.check`; manual dev-mode run |
| Classic never offered a custom room; a session uses Classic or custom, not both | `room.open()`, `lobby.ts` | `server.check` |
| No invite code or password in the logs | `custom.ts`, `server.ts` | `server.check` |

**Manual browser smoke checklist** (run in Phase 0 and after each UI-touching PR; the repo has headless automation in `scripts/browser-match.mjs`, but it is not in CI and does not judge visuals):

1. Startup → menu (no console errors).
2. Garage: swap weapons; the turret updates.
3. Practice TDM on the Scrapyard: countdown, fight, get wrecked (death board), respawn, Tab scoreboard, pause/resume, settings change, forced end → results → Play again.
4. Practice FFA on The City: pickups, hot zone, effect chips, standings, results (placing, crown).
5. Online Classic: a local server + two tabs (Find Match → ready check → match), chat, leave → a bot takes over.
6. Custom lobby: create one (Advanced: TDM 6 v 6, friendly fire, pickups on), join from a second window by code and from a third by the link, ready, start at 2 of 12, play (the TK column, pickups in TDM), Back to lobby, the end and the tally, Edit with a mode change (everyone unready, regrouped, tally reset), kick a member mid-match, reload in the waiting room and in a match (back within 20 s).
7. Exit to garage: `window.match` cleared; the renderer holds the same geometry/texture counts as before (the `LOADING_ARCHITECTURE.md` method).

`node scripts/browser-match.mjs custom` automates most of step 6 headlessly (needs Playwright installed globally, per the root `AGENTS.md`); it does not cover a reload in a match (`client.check` does).

---

## 18. High-risk areas

| Area | Why it is dangerous | Guard |
|---|---|---|
| **Deterministic simulation / RNG streams** | Four seeded mulberry32 streams: simulation `seed ^ 0x9e3779b9`, bot guns `seed ^ 0x2545f491` (`roster.ts`), FFA rules `createRng(seed)`, TDM rules `createRng(seed)` (the supply's, used only when pickups are on). **Every `random()` call's position in its sequence matters.** Reordering a `think()` branch, iterating combatants in another order, or adding one draw shifts every later draw, so the match diverges from there on. Online, prediction does not use RNG, but replays and practice↔room parity do. | pins; `simulation.check` replay; `server.check` room = practice |
| **Rapier state and creation order** | Collider creation order (ground slab, arena colliders in array order, then one `world.step()` to build query structures), body creation order (`enlist` in seat order, empty seats included), shell order, wheel order (fl, fr, rl, rr), `setCanSleep(false)`, CCD, body-type switches (remote cars kinematic; the prediction toggles Dynamic/Kinematic), and now bodies disabled for absent machines. Handles and solver order change results. A known Rapier 0.20 quirk (`NET_PLAN.md` F5): moved kinematic bodies are invisible to ray casts until `world.step()`. | pins; `netplay.check` |
| **Fixed timestep** | `PHYSICS_STEP = 1/60` (`physics.ts`) and `RATE.step = 60` (`protocol.ts`) are **two constants that must agree**; `world.timestep` is set explicitly; frame dt is clamped at 0.1 s; the server's `CATCH_UP` is 5. Moving either constant without the other breaks the client's `held()` GO-step maths. | `boundaries.check` rule 9 (Phase 1) |
| **Prediction/reconciliation** | The client drives with controls **rounded to hundredths exactly as `parseClient` reads them**; `held(seq)` re-derives the server's pre-match hold step by summing steps the way the rules do; reconciliation replays inputs with `world.step()` on the whole local world; `firm` handling until the first ack. Small refactors (rounding, the order of `drive` vs body placement) cause constant corrections. | `netplay.check` (corrections and errors printed and bounded) |
| **Snapshot interpolation** | `DELAY = 4` ticks; the server-tick reckoning from the least-delayed arrivals; `drawn` never goes backwards; events wait for the drawn tick except the player's own (`mine()`) and `ru`/`go`; catch-up after 1 s drops effects but keeps rules events. The codec refactor (Phase 4) touches exactly this. | `netplay.check`, `client.check`; the old-vs-new `owners()`/`mine()` table (§9.4) |
| **Replay determinism** | Journal semantics: join/leave apply *after* step k, `in` lines apply *at* step k (`ahead()` in `replay.ts`); a given is written only when it changes, and the view is stored as lag. `room.step()` order: release hold → drive people (+ journal notes) → journal `in` → `sim.step` → `rewind.record` → rules events → `report` → outcome, or after the results `end` (custom) / `restart` (Classic) → the custom abandonment check → broadcast → the idle check. Any reorder breaks replay equality. Journals are for fair-play review within `REPLAY_DAYS` (3 by default); a deploy in that window already means replaying old-build journals on new code, a pre-existing limitation: do not make it worse by changing the line format without a version field. | `server.check` replay (Classic and custom) |
| **Settings in the determinism path** (new) | The rules read the settings every step; the replay header and the welcome carry them. A refactor that changes a default, the respawn share arithmetic (`elapsed < share * duration`) or the order of the TDM rules' seeded draws changes play. | pins (Classic); the Phase 0 custom pins |
| **Server/client protocol** | Binary layout (`HEAD_BYTES` 14, `CAR_BYTES` 44, `ME_BYTES` 40; the `absent` flag in the existing flags byte), clamps, JSON event arrays, the lobby messages (`lb`, `lbs`), `PROTOCOL` bump discipline (now 6), `parseClient` clamping. A refactor that changes bytes or a JSON shape without a bump lets old pages mis-parse. | the Phase 0 fixtures; `protocol.check` |
| **Build id** | `build-id.ts` hashes `src/`, `server/` and the lockfile **including relative paths**. Every move changes the id, so every deploy makes open pages reload (expected; announce it). `scripts/match-smoke.mjs` regex-reads `PROTOCOL` from `game/src/net/protocol.ts`: **moving that file breaks the deploy smoke test.** | keep `net/protocol.ts` in place |
| **Arena digests** | Colliders come from `solid()` on props, via `matrixWorld`, rounded to mm. Changing builder call order, a seeded draw, a prop dimension, or when `solid()` is called relative to placement changes the digest; the server and deployed pages then disagree ("Arena mismatch — reload"). The custom work changed the bases on purpose (six starts a base) and re-pinned. | `arena.check` vs `digests.json`; `scripts/arena-parity.mjs` |
| **Headless arena builds** | The server runs visual builders with a fake `document` and no `window`; `materials/library.ts` skips the GPU bake when `window` is undefined. Adding `window` access at import time to any content module, or a material path that ignores headless mode, crashes or diverges the server. | `arena.check`; boundary rules 2 and 7 |
| **Asset lifecycle and Three.js disposal** | Library materials and baked textures are session-wide and **never disposed** by scene code. Geometries are disposed per match (`disposeGeometries`). The cached arena is borrowed by a scene and handed back by `scene.clear()`. `view.refit` rebuilds a model in place (the HUD keeps the `CarView` object). The server's `strip()` disposes headless geometries. Pickup tokens and the hot zone are pooled and pre-parked for the shader compile. A "tidy" refactor that adds `material.dispose()` breaks the next match. **The first glTF assets will bring per-asset materials and textures that *do* need disposal:** the rule must be refined then. | `STATE_OWNERSHIP.md`; the renderer counts in the manual smoke |
| **Shared materials** | `memo` keys dedupe materials across the whole app; wreck charring swaps materials per mesh and restores from `paint`. Mutating a shared material (colour, uniforms) changes every mesh using it. | code review; manual smoke |
| **WebSocket lifecycle (Classic)** | The matchmaking socket *becomes* the match link (`takeSeat`); `link.close()` semantics; StrictMode double mounts in development ("a link whose match never ran is the caller's"); `App` closes the seat on exit; ping interval cleanup. | `client.check`, `server.check`; manual dev-mode run |
| **The lobby ↔ match socket handoff** (new) | One socket serves the lobby, then the match (lobby words go to `aside`), then the lobby again (`release`). The match's `close()` must leave a released socket open; a released link must not report "connection lost"; inputs still in flight after the match are ignored, not struck. The repo's review fixed several bugs here. | `client.check`, `server.check`, `scripts/browser-match.mjs custom` |
| **Matchmaking state** | Ticket grace (15 s), the `sessionStorage` mark, retry backoff `[0, 1000, 2000, 4000, 7000]`, one live connection per user (a second tab takes the ticket), searching while seated refused, Classic or custom per session. | `matchmaker.check`, `server.check` |
| **Coming back to a custom lobby** (new) | The 20 s grace, `back` with the lobby id from `sessionStorage`, a fresh welcome for a held seat. A socket the server hasn't noticed is dead keeps holding a custom seat until the minute without input frees it (a known limit). | `client.check`; documented limit |
| **Empty seats** (new) | A new per-seat reader that forgets `present` shows a ghost car, counts a machine that isn't there, or lets a ray hit nothing. There are many readers (sim, rules, rewind, client, view, HUD, minimap, scoreboard, results, bots, spawn scoring). | the existing checks walk an empty seat through today's readers; the review checklist for new ones (§8.4) |
| **Lobby secrets** (new) | Codes and passwords must never be logged, listed or echoed; a wrong code is a strike; the lockout bounds the scrypt cost on the loop thread. | `server.check` (logs), `custom.check` (list) |
| **Shared room capacity** (new) | Busy custom lobbies can leave Classic with "no room free"; rooms are counted by kind, nothing is reserved yet. | `/health` counts; owner decision |
| **Deploy configuration** (new) | `deploy/compose.yml` passes neither `MAX_ROOMS` nor `MAX_LOBBIES` on purpose: podman-compose 1.3.0 would pass `${VAR:-default}` as text and stop the server. | the `compose.yml` comment; keep plain numbers if ever passed |
| **In-memory lobbies** (new) | A deploy or a crash ends every lobby. | accepted limit (one server); revisit with several servers (the repo plans the list in Nakama then) |
| **`.ts` import convention** | Modules loaded by plain-node checks import local files **with `.ts`**. A refactor that "cleans up" extensions, or adds path aliases, breaks `npm run check` outside Vite. | CI |
| **HUD per-frame writes** | `update()` runs every frame and writes only changed values; no allocations; elements are collected once by `data-hud` (pools of 12 now). Mode panels must scope their queries to their own root, or the TDM panel can grab the FFA panel's slots. | manual performance/allocation spot-check |
| **Loading runner** | Reports must be painted before each task (`flushSync`); tasks must be idempotent for Retry; dispose is safe at any point. Moving loading code must keep these contracts. | `loading.check` |
| **Dev probes** | The balance probes import `/src/game/*.ts` paths and rely on the `window.match` shape (the probes' own headers name `match.mode.rules`, `match.mode.tactics`, `match.chase`); `browser-match.mjs` reads `window.match` too. | Phase 6 updates the probes; one probe run |
| **Re-pinning** | A legitimate behaviour change must re-pin; an illegitimate one must not be "fixed" by re-pinning. | re-pins in their own commit, with the reason and the old hashes shown to match with the change left out, as the repo did |
| **Agent-written claims** | The repo's review found a log describing checks, flows and timings that did not exist. | Phase 1's "every check runs" rule; CI output, not prose, as evidence |

---

## 19. Avoiding over-engineering

**Rejected or postponed, with the concrete reason for this codebase:**

| Idea | Why not here |
|---|---|
| **ECS** | At most 12 machines and a few rockets per match. Combatants are plain objects iterated in arrays; the step order *is* the determinism contract. An ECS would re-express everything, risk the RNG and Rapier order, and speed nothing up (a room step averages ~0.22–0.24 ms in `server.check`'s 30 s runs). |
| **Event bus** | `SimEvents` (direct calls, one interface) and the rules' event queues (drained per step) already give ordering guarantees a bus would lose. Mid-step ordering of effects and sounds matters (`GAME_LOOP.md`). |
| **DI framework / service locator** | Composition happens in three places (`createMatch`, `createOnlineMatch`, `createRoom`) with explicit arguments, plus the services' hooks (`LobbyHooks`, the matcher's). It is readable and checkable. |
| **Redux/Zustand** | React state in `App.tsx` is five UI values; gameplay state must *not* be in React. The three external stores (`settings.ts` 61 lines, `net/matchmaking.ts` 189, `net/custom.ts` 274) are modules with listeners read through `useSyncExternalStore`. |
| **Physics abstraction** | One engine; prediction must run the *same* `drive.ts` on the *same* Rapier; hitscan, AI perception, item placement and aim are Rapier queries. An abstraction would be leaky and would put bit-exactness at risk. |
| **Replacing Three.js maths in the sim** | `THREE.Vector3`/`Quaternion` are used as maths throughout the hot paths; replacing them changes float operation sequences, so every pin, replay and practice↔room parity result changes. Nothing is gained: the server bundles Three.js anyway (for the arena builders). |
| **Separate packages per feature / workspaces** | One Vite package produces two bundles from one build id; the checks run the source directly. Packages would break the build id, the shared server bundle and the check convention. |
| **Microservices** | One Node process runs rooms, matchmaking and lobbies; Nakama is already the separate control plane. |
| **Abstract base classes / class hierarchies** | The codebase is factory functions returning objects (`createX`), typed by `ReturnType`. Stay consistent. |
| **A HUD view-model layer** | Per-mode panels (§7) solve the concrete problem (mode branches) without a per-frame object. |
| **Self-registering plugins** (`registry.add()` at import) | Loses compile-time completeness; depends on import order and bundling (§12). |
| **Generic `Clock`, `Audio`, `AssetLoader` interfaces** | No second implementation exists or is planned (§5). Add the asset loader with the first glTF asset. |
| **TS path aliases** | Break the plain-node checks (§14). |
| **`platform/` and `application/` layers as separate folders** | Too few files; `runtime/` covers the application role (§4.2). |
| **A `RoomKind` interface** | Two kinds; the branches are local and commented. Extract when a third kind is planned (§10). |
| **A generic "session manager" merging the two page stores** | Their state machines differ (Classic hands the socket over; custom keeps it and takes it back), and this layer just had four bugs fixed. Share only the identical helpers (§9.2). |
| **A schema-driven settings form/validator** | Seven options. Revisit past about ten, or when several arrive at once (§6.9). |
| **The lobby list in Nakama now** | One game server; the repo's own plan moves it with matchmaking when there are several. |
| **A class per mode with methods for every lobby question** | Traits are data; `sideOf`/`startable`/`tallyKey` are three pure functions. |
| **Profanity filtering, spectators, regions, pagination** | Product scope (the repo's plan lists them as "not in this work"), not architecture. |

**The test for any new abstraction (use it in review):** *which file would a developer otherwise have to edit to add content, or which bug does this prevent? Name it.* If you cannot, do not add it. The traits table passes it (seven shared files, three duplicated rules); a `RoomKind` interface does not yet.

---

## 20. Target directory tree

```
game/
├── build-id.ts · vite.config.ts · vite.server.config.ts · tsconfig*.json · package.json   (unchanged)
├── boundaries.check.ts                   NEW (Phase 1): rules, ratchets, server reachability, banned globals, every check runs
├── public/                               (unchanged; public/models/ when the first glTF lands)
├── server/                               FLAT, as today
│   ├── main.ts · server.ts · auth.ts · lobby.ts · matchmaker.ts · custom.ts · room.ts
│   ├── journal.ts                        NEW (Phase 7): ReplayLine, Given, row encode/decode (room + replay)
│   ├── inputs.ts                         NEW (Phase 7): per-person input queue / drain / stale policy
│   ├── recorder.ts · rewind.ts · fairplay.ts · records.ts · replay.ts · replay-main.ts
│   ├── arenas.ts · headless.ts · digests.json · load.ts · browser.ts
│   └── *.check.ts                        (custom.check.ts; golden pins in server.check.ts; + custom pins, Phase 0)
└── src/
    ├── main.tsx · App.tsx · analytics.ts · index.css           app shell (unchanged)
    ├── screens/                                                 React screens
    │   ├── custom/                       optional (Phase 6): Custom · Lobbies · Lobby (split: SlotGrid, SettingsCard, InviteCard) · LobbyForm
    │   └── … (GameCanvas optionally split: PauseMenu, ExitConfirm, LoadingOverlay; Avatar, Confirm, Results, …)
    ├── hud/                                                     Hud.tsx (shared panels only) · minimap.ts · Chat.tsx
    ├── runtime/                                                 browser composition (the "application" layer)
    │   ├── runtime.ts            startGame: loading steps, scene, composer, loop, arena session cache, dev globals
    │   ├── match.ts              playMatch, MatchSource, Match
    │   ├── practice.ts           createMatch (practice MatchSource, plays classic(mode))
    │   ├── online.ts             createOnlineMatch (online MatchSource; Classic and custom)
    │   ├── loading.ts            runTasks, STARTUP (+ loading.check.ts)
    │   └── stored.ts             loadout persistence (localStorage)
    ├── view/                                                    what the local player sees, hears and controls
    │   ├── view.ts · feed.ts · pilot.ts · input.ts · camera.ts
    │   └── effects.ts · audio.ts · sounds.ts · settings.ts · turntable.ts
    ├── render/                                                  shared Three.js infrastructure (server-reachable via arenas)
    │   ├── renderer.ts · environment.ts · postprocessing.ts · geometry.ts
    │   └── materials/ (library.ts · bake.ts · recipes.ts · facade.ts · canvasTextures.ts · groundGrime.ts · noise.ts)
    ├── sim/                                                     authoritative, headless, server-shared
    │   ├── simulation.ts · combat.ts · physics.ts · drive.ts · scoring.ts
    │   ├── mode.ts               MatchMode / ModeRules / Feed / ModeTiming / SupplyView (no rendering types)
    │   ├── loadout.ts            Loadout type, DEFAULT_LOADOUT, parseLoadout
    │   ├── difficulty.ts         Skill, DIFFICULTIES (pure data; read by the wire and the lobby screens)
    │   ├── ai/ (skill.ts · brain.ts · perception.ts · navigation.ts · think.ts · ai.check.ts · bots.check.ts)
    │   └── simulation.check.ts
    ├── modes/
    │   ├── traits.ts             Mode ids, MODE_TRAITS, MAX_SEATS, sideOf (near-leaf, plain-node safe)
    │   ├── settings.ts           MatchSettings, classic(), CUSTOM, respawnWait, checkSettings, labels
    │   ├── index.ts              MODES (label, tags, blurb, lineUp, create; satisfies Record<Mode, …>)
    │   ├── views.ts              MODE_VIEWS (client only, UI level: HUD and results panels)
    │   ├── scenery.ts            MODE_SCENERY (client only, view level: FFA's hot zone)
    │   ├── roster.ts             recruits, seat plans, bot names (MAX_SEATS of them), bot vehicle and guns
    │   ├── items/                config.ts · items.ts · supply.ts │ pickups.ts (client: tokens, drawn from mode.supply)
    │   ├── ffa/                  config.ts (+ traits) · rules.ts · mode.ts · ffa.check.ts │ zone.ts · hud.tsx · results.tsx (client)
    │   └── tdm/                  config.ts (+ traits) · types.ts · rules.ts · tactics.ts · mode.ts · tdm.check.ts │ hud.tsx · results.tsx (client)
    ├── content/
    │   ├── parts.ts              wheels, tyres, blades, lamps, spikes (vehicles + arena props)
    │   ├── vehicles/ index.ts (VEHICLES) · models.ts (MODELS) · types.ts · razor/{spec.ts, model.ts}
    │   ├── weapons/  index.ts (WEAPONS) · turrets.ts (TURRETS) · types.ts · minigun/{spec.ts, turret.ts} · rocketPod/{spec.ts, turret.ts}
    │   ├── arenas/   index.ts (MAPS) · arena.ts (contract + kit helpers) · digest.ts
    │   │             kit/{props,ground,buildings,street}.ts · scrapyard/scrapyard.ts · city/city.ts
    │   └── legacy/   (owner decision: VehicleGenerator, ArenaGenerator, proceduralTexture)
    ├── net/                                                     unchanged location
    │   ├── protocol.ts           match wire, PROTOCOL, BUILD (must stay here: the deploy smoke test reads it)
    │   ├── lobbyProtocol.ts      NEW (Phase 4): lobby messages, actions, invite codes, the lb parser
    │   ├── lobbyRules.ts         NEW (Phase 3): startable, tallyKey (server decides, page predicts)
    │   ├── events.ts             NEW (Phase 4): wire-event codec shared with server/recorder.ts
    │   ├── connection.ts · client.ts · prediction.ts · snapshots.ts
    │   ├── sessionSocket.ts      NEW (Phase 7): dial, retry schedule and loop, sessionStorage marks
    │   ├── matchmaking.ts · custom.ts                           the page's two stores (Classic, custom lobbies)
    │   ├── session.ts · chat.ts · chatCommand.ts
    │   └── *.check.ts
    └── shared/
        ├── rng.ts                mulberry32 (sim, content, view)
        ├── types.ts              Point
        └── format.ts             ms, clock
```

**Dependency direction (enforced by `boundaries.check.ts`).** A module imports from its own level or any lower one, never a higher one:

```
Level 6   screens/, hud/, modes/views.ts, modes/*/{hud,results}.tsx                React UI
Level 5   runtime/                                                                  browser composition
Level 4   view/, net/ (stores, client, connection), modes/scenery.ts,
          modes/items/pickups.ts, modes/ffa/zone.ts                                 presentation, client networking, mode 3D
Level 3   modes/ (domain: traits, settings, items, rules, adapters, registry, roster),
          net/{protocol,lobbyProtocol,lobbyRules,events}.ts                         rules, adapters, the wire
Level 2   sim/                                                                      authoritative gameplay
Level 1   content/                                                                  specs, models, arenas
Level 0   render/, shared/                                                          Three.js infrastructure; leaf utilities

server/   imports Levels 0–3 only (Level 0 render/ only through content/arenas, headless);
          browser.ts and the *.check.ts files are exempt (they emulate pages on purpose)
net/{protocol,lobbyProtocol,lobbyRules,events}.ts must additionally stay server-safe (no browser APIs)
```

**One extra rule on top of the levels** (§14.1, rule 3): `sim/` and the domain files of `modes/` must not import `render/` or `three/examples/**`, and use `three` only for maths. `render/` sits at Level 0 because `content/` needs it (arena and vehicle builders use the material library); the level order alone would otherwise let `sim/` reach it.

---

## 21. Architecture decision summary

| Decision | Recommendation | Why | Confidence |
|---|---|---|---|
| Big-bang restructure | **No** | The custom work reused every seam without a fork; a restructure relabels what holds | High |
| Golden pins | **Done** (repo); add wire fixtures + custom pins | Cross-machine stability now shown | High |
| Mechanical boundary enforcement | **Yes**: `boundaries.check.ts` with ratchets | Convention only today; agent claims proved unreliable | High (need) / Medium (mechanism) |
| **Mode traits table** | **Yes** (Phase 3) | 34 mode-literal lines in 7 shared files, server included | High |
| **Shared lobby rules** | **Yes** (Phase 3) | Side ×3, start ×2, tally ×2, 12 seats ×5 | High |
| Leaf ids / type-cycle fix | **Yes** (Phase 2) | Ids at the top of the graph; a 14-file type cycle | Medium-High |
| `DIFFICULTIES` as a leaf | **Yes** (Phase 2) | The wire format should not import the AI | Medium |
| Weapon `id` on the spec | **Yes** | Identity by turret model; now behind the one-gun setting too | High |
| Turret registry + weapon data | **Yes** | Weapons stop editing `vehicle.ts`/`Hud.tsx` | High |
| Wire-event codec | **Yes** | Positional indices across two files | High |
| Lobby protocol module | **Yes** (Phase 4) | `protocol.ts` 604 lines; the lobby wire will grow | Medium |
| Shared session-socket helpers | **Yes, helpers only** | Duplicated machinery; keep two state machines | Medium |
| Merged session manager | **No** | Different flows; fragile layer | Medium |
| `RoomKind` object | **Not yet** | Two kinds; refactor with a third | Medium |
| Schema-driven settings | **Not yet** | Seven options | Medium |
| Remove rendering from `MatchMode` | **Yes** (Phase 5) | The server bundle carries the pickups and hot-zone views | High |
| Per-mode UI panels | **Yes** (Phase 5) | HUD/Results branching | Medium-High |
| Pickups drawn generically from `mode.supply` | **Yes** (Phase 5) | Only FFA's zone needs a per-mode view | High |
| Feature folders / layer folders | **Yes, mechanical, optional** (Phase 6) | Navigation; least value per risk | Medium |
| Physics abstraction, ECS, event bus, DI, state library | **No** | One engine, at most 12 machines, ordered direct calls and three small external stores: each would add indirection and put determinism at risk for no gain (§19) | High |
| Lobby list in Nakama | **Later** (with several servers) | One server today | High |
| Second-vehicle capability | **When scheduled** (`PROTOCOL` 7) | A feature | High |
| Netplay timing case | **Make deterministic** (steps, not ms) | Documented flakiness | Medium |
| Vitest | Not adopted; owner may remove | `.check.ts` works | High |

---

## 22. Final recommendation

### Recommended refactor strategy

**1. Definitely change**
- Close the characterization gaps (wire fixtures, custom pins) and land `boundaries.check.ts` with ratchets (Phases 0–1).
- Weapon identity, `TURRETS`, weapon data; leaf ids and `DIFFICULTIES`; the type cycle (Phase 2).
- **Mode traits and shared lobby rules** (Phase 3): the custom work's leaks, while they are fresh.
- The wire-event codec and the lobby protocol module (Phase 4).
- Rendering out of `MatchMode`; per-mode HUD/Results panels (Phase 5).

**2. Probably change**
- The folder reorganisation (Phase 6), if more content or contributors are coming.
- Split `ai.ts`; `journal.ts` + `inputs.ts` from `room.ts`; shared session-socket helpers; split `Lobby.tsx`; make the netplay timing case deterministic (Phase 7).

**3. Do not change**
- Everything revision 1 listed (the sim/view/pilot/feed split, `MatchSource`, `MatchMode` + pure rules, `SimEvents`, the fixed step, seeded streams, Rapier order, the binary layout, `PROTOCOL`/`BUILD`, `net/protocol.ts`'s path, arena digests, the headless shim, session caches, imperative HUD writes, the `.check.ts` convention, flat `server/`, the pure matcher, Nakama as the control plane).
- **And from the custom work:** `server/custom.ts` pure with hooks; one settings validator; settings, lobby and seat plan in the replay header and records; empty seats keeping per-seat arrays; the absent flag that leaves Classic bytes alone; secrets never logged; custom rooms invisible to Classic; the socket handoff design (`aside`/`release`); the re-pin discipline.

**4. Postpone**
- `RoomKind`, a merged session manager, schema-driven settings, the lobby list in Nakama, second-vehicle seats (Phase 8), a firing-behaviour table, a physics-free arena layout, the glTF asset loader, per-layer tsconfig spikes.

### The first 5 PRs (revised)

| # | PR | Scope | Why it is safe | Verification | Revert |
|---|---|---|---|---|---|
| **1** | **Wire fixtures + custom pins** | `protocol.check.ts`: committed bytes/JSON for a snapshot frame (with an absent row), a welcome with settings, a `ro`, an input, a `LobbyRow`/`LobbyView`. Two whole-match pins with custom settings (TDM 12 seats, friendly fire, all pickups, kill limit 25, fast respawn, a seat emptied and taken; FFA 2 seats, pickups off) | Test-only | Green on CI and one other machine; mutation test: flip one clamp in `packCars` → the fixture fails | trivial |
| **2** | **`boundaries.check.ts`** | Rules 1–10 (§14.1) on today's file lists; ratchets: type cycles 1, mode literals 34; every `*.check.ts` runs; `RATE.step * PHYSICS_STEP === 1` | Test-only; passes on today's code | Negative tests in review (an `audio.ts` import into the sim; a new `'tdm'` in `server/custom.ts`; a check file dropped from `package.json`) | trivial |
| **3** | **Leaf ids and data** | `Mode` ids in a leaf module, `MODES satisfies Record<Mode, …>`; `DIFFICULTIES` to a leaf; `ms`/`clock` to a leaf; `SupplyView` in `mode.ts`; one `Point` | Type-level and moves; no runtime change | Type-cycle ratchet → 0; pins, fixtures unchanged; `tsc -b` | single PR |
| **4** | **Mode traits + shared lobby rules (server side, 3a)** | `MODE_TRAITS` (per-mode entries in each `config.ts`), `MAX_SEATS`, `sideOf`; `net/lobbyRules.ts` (`startable`, `tallyKey`); `matchSettings.ts`, `server/custom.ts`, `server/room.ts`, `tdm/mode.ts lineUp`, `protocol.ts SLOTS` use them | Behaviour-preserving; the server's answers unchanged | Pins; `custom.check` (71), `server.check`'s custom cases, `protocol.check`; mode-literal ratchet lowered by 13 | single PR |
| **5** | **Weapon identity** | `WeaponSpec.id` (== key, asserted); `botGun` and the one-gun setting preserve it; `weaponId(spec) → spec.id`; callers in room, recorder, rewind, records, client | Same id strings on the wire | Fixtures and pins unchanged; `server.check` records; a scaled bot copy keeps its id | single PR |

PR 4 (the server half of Phase 3) comes before the rest of Phase 2 on purpose: it needs only PR 3's leaf ids, and it removes the leaks a third mode or a new lobby feature would copy. The weapon PRs do not depend on it, so the order between them is free.

Then: **3b** (traits in `Lobby.tsx`, `LobbyForm.tsx`, `Results.tsx`, HUD pools), the turret registry and weapon data, the codec + lobby protocol module, the UI seams.

### What would change this recommendation

- **A third mode or a lobby feature is next** → do Phase 3 straight after PR 2; it is what that work would otherwise copy.
- **A third kind of room (ranked, tournament, spectating) is planned** → extract `RoomKind` before building it.
- **A second vehicle is scheduled** → Phase 8 right after Phase 2 (`PROTOCOL` 7).
- **Several game servers are planned** → the lobby list and matchmaking move toward Nakama per the repo's own plan; revisit §10 then, not before.
- **Several people or agents work in parallel** → Phase 1 first (it already is), and Phase 6 sooner.

---

## Appendix A: Import graph (current)

Internal edges of production files at `3b0943d` (`type:` = type-only import). Paths under `game/src/` unless prefixed `server/`; inside `game/`, targets drop the `game/` prefix. Checks are omitted. Generated by the script in Appendix B.

```
App.tsx -> game/loadout, game/maps, game/modes, type:net/connection, net/custom, net/matchmaking, net/protocol, screens/GameCanvas, screens/Garage, screens/Loading, screens/MainMenu, screens/MapSelect, screens/Matchmaking
game/ArenaGenerator.ts -> rng
game/VehicleGenerator.ts -> rng, proceduralTexture
game/ai.ts -> type:arena/arena, combat, type:vehicle/drive
game/arena/arena.ts -> geometry, materials/library, arena/props
game/arena/buildings.ts -> geometry, materials/facade, materials/library, materials/recipes, rng, arena/props
game/arena/city.ts -> geometry, materials/facade, materials/library, rng, arena/arena, arena/buildings, arena/ground, arena/props, arena/street
game/arena/digest.ts -> type:arena/arena
game/arena/ground.ts -> geometry, materials/library
game/arena/props.ts -> geometry, materials/library, vehicle/parts
game/arena/scrapyard.ts -> geometry, materials/library, rng, arena/arena, arena/ground, arena/props, arena/street
game/arena/street.ts -> geometry, materials/library, rng, vehicle/parts, arena/buildings, arena/props
game/audio.ts -> settings, sounds
game/camera.ts -> settings
game/environment.ts -> materials/noise
game/feed.ts -> audio, type:mode, scoring, type:view
game/ffa/mode.ts -> type:ai, type:arena/arena, items/items, type:items/pickups, items/supply, type:matchSettings, mode, ffa/config, ffa/rules, type:ffa/zone
game/ffa/rules.ts -> items/config, items/items, items/supply, matchSettings, mode, rng, scoring, ffa/config
game/ffa/zone.ts -> materials/library, type:ffa/rules
game/items/items.ts -> type:arena/arena, type:matchSettings, items/config
game/items/pickups.ts -> materials/library, items/config, items/items
game/items/supply.ts -> mode, type:scoring, items/config, items/items
game/loading.ts -> physics, renderer
game/loadout.ts -> combat, vehicle/vehicles
game/maps.ts -> type:arena/arena, arena/city, arena/scrapyard, type:modes
game/match.ts -> ai, type:arena/arena, arena/digest, audio, combat, feed, type:loadout, type:maps, matchSettings, modes, physics, pilot, roster, simulation, vehicle/drive, view
game/matchSettings.ts -> ai, combat, ffa/config, type:modes, tdm/config
game/materials/bake.ts -> materials/noise, type:materials/recipes
game/materials/canvasTextures.ts -> rng
game/materials/facade.ts -> materials/recipes
game/materials/library.ts -> renderer, materials/bake, materials/canvasTextures, materials/facade, materials/groundGrime, materials/recipes
game/mode.ts -> type:ai, type:arena/arena, type:items/supply, type:scoring
game/modes.ts -> type:arena/arena, ffa/config, ffa/mode, ffa/zone, items/items, items/pickups, matchSettings, type:simulation, tdm/config, tdm/mode
game/online.ts -> net/client, type:net/connection, type:arena/arena, feed, type:maps, match, simulation, view
game/physics.ts -> type:arena/arena
game/pilot.ts -> type:camera, input, type:simulation
game/postprocessing.ts -> type:settings
game/proceduralTexture.ts -> rng
game/roster.ts -> ai, type:arena/arena, combat, type:matchSettings, modes, rng, type:simulation, type:vehicle/vehicles
game/runtime.ts -> net/connection, type:ai, arena/digest, audio, environment, loading, type:loadout, maps, match, type:modes, online, physics, postprocessing, renderer, settings
game/simulation.ts -> ai, type:arena/arena, combat, type:mode, rng, scoring, vehicle/drive, vehicle/vehicles
game/sounds.ts -> rng
game/tdm/mode.ts -> type:ai, type:arena/arena, items/items, type:items/pickups, items/supply, type:matchSettings, mode, tdm/config, tdm/rules, tdm/tactics, type:tdm/types
game/tdm/rules.ts -> items/config, items/items, items/supply, matchSettings, mode, rng, scoring, tdm/config, type:tdm/types
game/tdm/tactics.ts -> tdm/config, type:tdm/types
game/tdm/types.ts -> type:items/supply, type:matchSettings, type:mode, type:scoring
game/turntable.ts -> type:combat, environment, geometry, materials/library, renderer, vehicle/vehicle, type:vehicle/vehicles
game/vehicle/drive.ts -> physics
game/vehicle/parts.ts -> geometry, materials/library
game/vehicle/vehicle.ts -> type:combat, geometry, materials/library, rng, vehicle/parts, vehicle/vehicles
game/vehicle/vehicles.ts -> type:geometry, type:vehicle/drive
game/view.ts -> type:arena/arena, audio, camera, effects, geometry, materials/library, type:simulation, vehicle/drive, vehicle/vehicle, vehicle/vehicles
hud/Chat.tsx -> type:net/chat, net/chatCommand, screens/search
hud/Hud.tsx -> type:game/ffa/rules, game/items/config, game/items/items, type:game/items/supply, type:game/match, game/mode, game/modes, game/settings, type:game/simulation, game/tdm/config, hud/minimap
hud/minimap.ts -> type:game/arena/arena, type:game/ffa/rules, game/items/items
main.tsx -> screens/Notice
net/chat.ts -> net/chatCommand, type:net/protocol, net/session
net/client.ts -> type:game/arena/arena, game/combat, type:game/mode, game/modes, game/physics, game/simulation, game/vehicle/drive, game/vehicle/vehicles, type:net/connection, net/prediction, net/protocol, net/snapshots
net/connection.ts -> type:game/loadout, net/protocol
net/custom.ts -> type:game/loadout, net/connection, net/protocol, net/session
net/matchmaking.ts -> type:game/loadout, net/connection, net/protocol, net/session
net/prediction.ts -> game/physics, game/vehicle/drive, type:net/protocol
net/protocol.ts -> game/combat, type:game/ai, type:game/loadout, game/matchSettings, type:game/modes, game/scoring, type:game/simulation, game/vehicle/vehicles
net/snapshots.ts -> net/protocol
screens/Confirm.tsx -> game/audio, screens/Menu
screens/Custom.tsx -> type:game/loadout, net/custom, net/matchmaking, screens/Lobbies, screens/Lobby, screens/search
screens/GameCanvas.tsx -> type:game/ai, game/audio, type:game/loading, type:game/loadout, game/maps, type:game/match, type:game/modes, game/runtime, net/chat, net/connection, game/settings, hud/Chat, hud/Hud, screens/Confirm, screens/Drawer, screens/Menu, screens/Results, screens/search, screens/SettingsPanel
screens/Garage.tsx -> game/combat, game/loading, type:game/loadout, game/turntable, game/vehicle/drive, game/vehicle/vehicles, screens/Menu
screens/Loading.tsx -> game/loading, screens/Menu
screens/Lobbies.tsx -> game/maps, game/matchSettings, game/modes, net/custom, net/protocol, net/session, screens/LobbyForm, screens/Menu, screens/search
screens/Lobby.tsx -> game/ai, game/maps, game/matchSettings, game/modes, game/roster, game/tdm/config, hud/Chat, net/chat, net/custom, type:net/protocol, screens/Avatar, screens/Confirm, screens/Lobbies, screens/LobbyForm, screens/search
screens/LobbyForm.tsx -> game/combat, game/maps, game/matchSettings, game/modes, net/custom, net/protocol, screens/Drawer, screens/Menu
screens/MainMenu.tsx -> analytics, game/loading, net/session, screens/Drawer, screens/Menu, screens/PatchNotesPanel, screens/SettingsPanel
screens/MapSelect.tsx -> game/ai, game/loading, type:game/loadout, game/maps, game/modes, net/matchmaking, screens/Avatar, screens/Custom, screens/Menu, screens/search
screens/Matchmaking.tsx -> game/audio, game/maps, game/modes, net/matchmaking, screens/Menu, screens/search
screens/Results.tsx -> game/audio, game/maps, type:game/match, game/modes, type:game/simulation, game/tdm/config, type:net/protocol, screens/Lobby, screens/Menu
screens/SettingsPanel.tsx -> game/postprocessing, game/settings
screens/search.ts -> net/custom, net/matchmaking
server/arenas.ts -> type:game/arena/arena, game/geometry, game/maps
server/browser.ts -> type:game/arena/arena, type:game/mode, game/physics, type:game/simulation, net/client, net/connection, server/auth
server/custom.ts -> type:game/ai, type:game/matchSettings, type:game/modes, type:game/roster, net/protocol
server/load.ts -> game/maps, game/matchSettings, game/physics, net/protocol, server/arenas, server/room
server/lobby.ts -> type:game/arena/arena, type:game/loadout, game/maps, game/modes, net/protocol, server/custom, server/matchmaker, type:server/records, server/room
server/main.ts -> game/maps, game/physics, server/arenas, server/server
server/matchmaker.ts -> type:net/protocol
server/recorder.ts -> type:game/simulation, net/protocol
server/records.ts -> type:server/room
server/replay-main.ts -> game/physics, server/replay, type:server/room
server/replay.ts -> type:game/arena/arena, type:game/maps, game/physics, server/room
server/rewind.ts -> game/combat, type:game/simulation, game/vehicle/vehicles
server/room.ts -> game/ai, game/combat, type:game/loadout, type:game/arena/arena, type:game/maps, game/matchSettings, game/modes, game/physics, game/roster, game/simulation, game/arena/digest, net/protocol, server/arenas, server/fairplay, server/recorder, server/rewind
server/server.ts -> type:game/arena/arena, type:game/maps, game/physics, game/rng, net/protocol, server/auth, type:server/custom, server/lobby, type:server/matchmaker, server/records, server/room
```

Cycles: value imports **none**; type-level: one strongly connected set of 14 files (§3.4).

## Appendix B: Reproducing the analysis

```bash
git clone https://github.com/aasumitro/bbmvc && cd bbmvc && git checkout 3b0943d && cd game
npm ci
npm run lint && npx tsc -b && npm run check                     # baseline (§0.3)
git diff --stat f28082e 3b0943d -- .                             # the custom-lobby commit
grep -rn "'tdm'\|'ffa'" src server --include=*.ts --include=*.tsx | grep -v check.ts | grep -v "src/game/ffa/\|src/game/tdm/"   # mode literals (§1.5)
grep -rn "lobby ?\|lobby &&\|!lobby\|r.lobby\|room.lobby" server --include=*.ts | grep -v check.ts                                 # room-kind branches (§10)
```

The import graph and cycles came from two small Node scripts: one parses `import … from`, `export … from` and side-effect imports, resolves relative specifiers, marks `import type`, and runs Tarjan's SCC; the other prints the shortest cycle through each edge (that is how the two root edges in §3.4 were found). Dynamic `import()` (only `main.tsx → App`) is not counted. The same logic is the natural core of `boundaries.check.ts`.
