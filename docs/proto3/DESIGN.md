# Proto3 — Design Guideline

Status: design approved in discussion (2026-11-14). No code yet. This file moves to the Proto3 repo as `docs/DESIGN.md` on day 1.
Every rule below exists because of a concrete failure mode. When code disagrees with this document, either the code is wrong or this document must be updated in the same commit (with an ADR if the decision changes).

---

## 0. Goals and non-goals

Goals, in priority order:
1. **Same game.** Proto3 plays and looks like the current `shmup-fighter.html` (balance values identical; only invisible float decimals may differ).
2. **Sandbox engine.** Everything except a tiny kernel is a *pack*. The base game is a set of packs in the exact format mods use. Anything can be added, replaced or removed without touching unrelated code.
3. **AI-editable.** Any request maps to a small, predictable set of files. One file answers one question.
4. **Debuggable by construction.** Every bug is reproducible from a replay; every desync names the field that diverged.
5. **Online-safe.** Deterministic simulation, rollback netcode, 2–32 players.
6. **One link, no install.** Static site on GitHub Pages.

Non-goals (for now): mod editor UI, auto-translation, leaderboard server, mod sandboxing. Formats must not prevent them later.

---

## 1. Stack (decided)

| Concern | Choice | Why / rejected |
|---|---|---|
| Language | TypeScript, `strict`, no `any` | Errors caught before running. Rust/WASM rejected: slower iteration, game is not CPU-bound. |
| Build | Vite | Static output, HMR in dev. |
| Menus | Svelte (pinned major) | One file per screen with markup+logic+CSS+transitions. React rejected (heavier), raw DOM rejected (mess at 15+ screens). |
| Fight rendering | Own WebGL2 renderer, Canvas2D fallback | Current game already batches bullets in WebGL. PixiJS only if a benchmark proves it faster (ADR required). |
| Tests | Vitest (headless sim), Playwright (UI, mobile, screenshots) | |
| Hosting | GitHub Pages, deploy by GitHub Actions only after all checks pass | Pages cannot set COOP/COEP headers → no SharedArrayBuffer → sim stays on main thread (see §9). |
| Online | Keep current PeerJS + WebRTC + Cloudflare TURN (`sf-turn` worker), host relay (star) | Works today. |
| Workers | `sf-turn` (TURN creds), `sort-maker` (YouTube playlist/search/thumb proxy) live in `workers/` | Deployment mode decided later. |
| Runtime deps | PeerJS, small TS transpiler for mods (lazy-loaded), small schema validator | Minimise deps: 10–20 year horizon. |

---

## 2. Kernel vs packs

### 2.1 Kernel (frozen, tiny, no gameplay)
- Fixed-step loop (60 Hz) + render interpolation hook.
- State container built from pack schemas (§4).
- Deterministic math (`sim.math`) and seeded RNG streams (§3).
- Snapshot / restore / hash / diff generated from schemas.
- Pack loader: manifest validation, dependency resolution, deterministic load order, overrides.
- Event bus + hook phases (§5).
- Input frame format and command format (§6).
- Entity handle allocator (§4.3).
- Error isolation (§5.5).

The kernel never imports a pack. Packs import only the kernel API (`@kernel`) and the public API of packs they declare as dependencies.

### 2.2 Packs (everything else)
```
packs/
  core-combat/   players, bullets, collisions, statuses, damage, KO, parry, spells, items framework
  touhou/        characters/, items/, maps/, rules/      (Touhou content)
  modes/         versus, training, playground, online-match, cpu-battle
  ui/            screens and menus (Svelte)
  hud/           in-fight HUD (drawn by render, not Svelte — §8.3)
  render/        fight renderer, effects, camera
  audio/         sfx, music, special tracks priority
  input/         keyboard, gamepad, touch, mouse-buttons, bindings
  online/        transport, lobby, relay, rollback driver, spectators
  youtube/       playlist player, video background
  lab/           bug lab (secret code "ewekeks" on title screen)
  dev-tools/     inspector, logs, overlays (disabled by default in Options)
  i18n/          string tables (en, fr), language switch
```

### 2.3 Pack manifest (`pack.ts`)
```ts
export default definePack({
  id: 'touhou',                 // namespace; all ids inside are 'touhou:<name>'
  version: '1.0.0',
  requires: ['core-combat@^1'], // semver ranges
  provides: ['characters', 'items', 'maps'],
  overrides: [],                // explicit ids this pack replaces, e.g. 'touhou:reimu'
  state: touhouStateSchema,     // this pack's state slice (§4)
});
```
Content files are discovered by folder convention (`import.meta.glob` for built-in packs, zip listing for mods). No hand-maintained registries.

### 2.4 Overrides and load order
- Ids are namespaced: `touhou:reimu`, `mymod:reimu`.
- Replacing content is explicit: `overrides: ['touhou:reimu']`. Silent collisions are a load error.
- A load-order list in Options decides conflicts. Same list + same pack hashes are required for online play (§10.2).

---

## 3. Determinism rules (the online contract)

1. Sim code uses only `+ - * / %`, comparisons, `Math.sqrt`, `Math.floor/ceil/round/trunc/abs/min/max/sign`, and `sim.math` (`sin, cos, atan2, hypot, pow, exp, log, lerp, angle helpers`). `Math.sin/cos/tan/atan/atan2/hypot/pow/exp/log/cbrt/random` are banned in `sim` and `packs/*/sim`. Enforced by lint.
2. No `Date`, `performance`, `setTimeout`, DOM, `localStorage`, or network in sim code. Enforced by import-boundary lint.
3. **RNG streams.** `rng.stream('touhou:items')` — each pack/system has its own seeded stream derived from match seed + stream id. Adding a random call in one mod never shifts another pack's randomness (keeps replays and parity stable).
4. **Sorting.** Only `sim.sort(arr, key)` which breaks ties by entity id. `Array.prototype.sort` banned in sim.
5. **Iteration order.** Iterate arrays or id-ordered collections. No iteration over object keys that may differ by insertion history.
6. **Floats.** Float64 everywhere in sim (`Float64Array` for typed storage). No Float32 in sim state.
7. **Runtime sanitizer** (dev, lab): during each sim step, `Math.random`, `Date.now`, `performance.now` and banned `Math.*` are trapped and throw with file/line. Catches mods, which lint cannot see.
8. **Input quantisation.** Analog values are int8 before entering the sim.
9. Everything that affects the sim is an **input** or a **command** at a frame (§6). Nothing else.

---

## 4. State

### 4.1 Data only
State contains numbers, strings, booleans, arrays, typed arrays and plain objects. Never functions, class instances, DOM nodes or closures. Behaviour is referenced by id (`{ item: 'touhou:black-hole', t0: 900, params }`). This makes snapshots, hashing, replays, spectator joins, and hot reload work.

### 4.2 Schemas
Each pack declares its state slice with a schema (field types, defaults, sizes). The kernel generates from schemas:
- preallocated snapshot slots and copy functions (no per-frame allocation),
- hash function (whole state, per-section hashes),
- field-level diff (desync reports),
- inspector UI (dev-tools),
- validation (dev asserts, loaded saves).
Slices are namespaced: `state.packs['touhou']`. A pack may only write its own slice; reading another pack's slice goes through that pack's public API.

### 4.3 Entities
- Handles = index + generation. A stale handle (entity died, slot reused) is detected, never confused with the new entity.
- Kinds stay separate: players (≤32, objects), bullets (thousands, struct-of-arrays typed storage behind a small API: `spawn`, `kill`, `forEach`, `query(area)`), world entities (hundreds, objects).
- Spatial uniform grid for collision queries, rebuilt per frame from typed arrays.
- Hard cap `MAX_BULLETS` enforced in the single `spawn` function (old bug: `spawnH` bypassed the cap). Cap policy is explicit and tested (heavy shots replace the oldest light bullet).

### 4.4 Statuses and stat pipeline
All rule-changing effects are statuses in state: `{ id, owner, source, t0, duration, stacks, params }`. Derived values are computed: `stat(player, 'damageOut')` = base × all active modifiers, in a deterministic order. Items, spells, rules and mods add statuses; they never sprinkle `if (flag)` checks through other systems.

### 4.5 Special cases that must be designed for (from the old game)
- **History Rewritten** (restores past state) → kernel service `history` storing snapshots *inside* state (schema type `blob`), so rollback stays correct.
- **Body Swap / control remap** → sim-level controller indirection table (`controllerOf[player]`) in state.
- **View-affecting items** (darkness, fog, inverted colours, camera swap) → sim exposes *view modifiers* in state; render reads them. Render never decides gameplay.
- **Time stop, slow motion, x2 speed** → kernel time-scale owned by the sim (frames still advance deterministically).
- **Echoes/clones repeating attacks** (Four of a Kind) → scheduled actions queue in state.

---

## 5. Events and hooks

### 5.1 Events
Typed events emitted by sim systems: `hit`, `parry`, `ko`, `spell:start`, `item:pickup`, `bullet:spawn`, `frame`, ... Each event type is declared in one file with its payload schema and documentation (catalog generated).

### 5.2 Phases instead of priorities
Hooks attach to named phases (`damage:before`, `damage:apply`, `damage:after`) with optional `after: ['core-combat:parry']`. Ties are broken by pack id. Numeric priorities are banned (they degenerate into priority wars).

### 5.3 Mutation contract
`*:before` hooks may modify declared fields of the event (e.g. damage multiplier) or cancel it. `*:after` hooks are read-only for that event. Contract per event documented in its file.

### 5.4 Re-entrancy
Events emitted inside hooks are queued and processed in deterministic FIFO order with a max chain depth. Exceeding it is an error naming the hook chain.

### 5.5 Error isolation
An exception in a hook disables that hook from that frame on, records a report, and the match continues. Same code + same state ⇒ every client disables it at the same frame ⇒ no desync.

### 5.6 Hot paths
No per-bullet-per-frame dynamic hook dispatch. Systems iterate typed arrays directly. Packs act on events, or replace a whole system (allowed via override).

---

## 6. Input and commands
- `InputFrame` per player per frame: buttons bitmask + quantised analog.
- `Command` = `{ frame, type, payload }` for anything else that changes the sim (host rule change mid-match, pause in offline, debug spawns in training). Commands travel in the net stream and are stored in replays.
- Devices (keyboard, pads, touch, mouse buttons) live in `packs/input` and produce InputFrames. Bindings are UI/settings, never sim.
- Online input delay is configurable; prediction repeats last input.

---

## 7. Presentation: FX, audio, interpolation
- Sim emits events; render and audio derive particles, flashes, shakes, sounds from them. FX are not in sim state → lighter rollback, trivial *Effects OFF*.
- **Rollback cancel:** each FX/sound is tagged with its source event key `(frame, eventId)`. After a resimulation, FX whose event no longer exists are removed immediately; long sounds stop. Short sounds already played cannot be undone (accepted).
- Duplicate events from resimulation are deduplicated by the same key.
- **Interpolation:** render draws between the last two sim states for 120/144 Hz screens. Must never be worse than the old game; can be disabled in Options.
- *Effects* setting: FULL / REDUCED / OFF (OFF removes shakes, flashes, zooms, inversions, particles; keeps gameplay-critical visuals). Epilepsy safety is a requirement, tested by Playwright.
- Special music (results, last stock) has priority over the YouTube player (pause video, resume after).

---

## 8. UI
### 8.1 Screens
One folder per screen in `packs/ui/screens/<screen>/` (screen component + panels as separate files + its strings). Navigation is a small router with a stack; every screen works with keyboard, gamepad, mouse and **touch only** (tested).
### 8.2 Style
Reproduce the current look (skewed cards, italic heavy titles, glass sidebar, Limbus-like frames). Shared design tokens (colors, fonts, easing) in one file.
### 8.3 HUD
In-fight HUD is drawn by `render`/`hud` (canvas/WebGL), not Svelte, to avoid DOM work and GC every frame.
### 8.4 Text
No user-visible string in code. All text via `t('pack:key')`, string tables per pack per language (`en.json`, `fr.json`), Rivals-style. Auto-translation may be added later as a fallback for missing keys.

---

## 9. Threads and timing
- Sim on main thread (no SharedArrayBuffer on GitHub Pages).
- Hidden tab: rAF is throttled → a Web Worker clock keeps net ticks alive (as today).
- Budget: sim step ≤ 4 ms at 32 players + max bullets on a mid laptop; measured in CI benchmark and in the lab.
- Zero allocations per frame in sim hot paths (pools, preallocated snapshots).

---

## 10. Online
### 10.1 Netcode
Rollback with max prediction window (stall beyond it, as today), star topology with host relay, bundled messages, bandwidth guard, heartbeat drop. Spectators receive the input stream (not a video); mid-match join = full state snapshot (from schema serializer) + inputs.
### 10.2 Handshake
Protocol version + game version + content hash (all packs, mods, load order). Mismatch → refused with the exact differing pack named. Later: host can send mods.
### 10.3 Locks
Hot reload disabled during online matches. Content locked per match.
### 10.4 Desync
Periodic hash per section. On mismatch: exchange snapshot, field-level diff, report names system + field + frame; replay attached.
### 10.5 Scale
32 players: inputs bit-packed and run-length encoded; host bandwidth measured; warning above 12 players kept.

---

## 11. Mods
- Mods = zip files in a `mods/` folder chosen once via the File System Access API (Chrome/Edge/Opera). Fallback: drag & drop zip (Firefox/Safari/mobile). Browser re-asks folder permission each session (one click).
- Zip layout mirrors packs: `pack.ts|js`, `characters/`, `items/`, `maps/`, `rules/`, `i18n/`, `assets/`.
- JS or TS; TS transpiled in-browser by a lazy-loaded transpiler. A `.d.ts` of the kernel + core-combat API ships for editor autocompletion.
- Hot reload key re-reads the folder; offline/training only. Schema changes on reload: new fields get defaults, removed fields dropped.
- Load-time validation: missing fields, impossible values, id collisions → clear panel before playing.
- Runtime error panel: file, line, attack/item, values at failure; the faulty hook is disabled.
- Security: mods run with page privileges. Warning before enabling a mod from an unknown source. Sandboxing later.
- Built-in packs are written against the same public API (dogfooding guarantees mods can do anything the game does).

---

## 12. Debug system (`packs/dev-tools`, off by default in Options)
- **Logger:** categories × levels, ring buffer, stripped from production builds (zero cost).
- **Event journal:** last N seconds of sim events (who, what, values, frame).
- **Inspector:** click an entity → live state from schema; frame step; rewind using rollback snapshots; trigger any item/status.
- **Invariants:** NaN, out-of-range HP/positions, stale handles, schema violations → first failure produces a report.
- **Determinism sanitizer:** §3.7.
- **Profiler:** time per system per frame, allocation counter.
- **Bug report = replay** (seed, settings, content hash, inputs, commands) + logs + last snapshots. Headless CLI replays it to a given frame and dumps state.
- **Error file auto-download** option (kept from the old game).

---

## 13. Bug lab (`packs/lab`, secret code `ewekeks` on title screen)
Config screen (number of parallel matches, characters, items, rules, duration, forced random rollbacks, simulated lag/loss) → runs matches in Web Workers with CPUs → each match is verified twice (straight replay, and with random rollbacks); any hash difference, invariant failure or exception is collected with its replay. Live counters; export one report file. Same lab runs headless in Node for me.

---

## 14. Tests and CI
- Scenario DSL: `scenario().players('touhou:reimu','touhou:marisa').give(0,'touhou:black-hole').run(120).expect(...)`. One test file next to every attack/item.
- Determinism tests (fuzz inputs twice, and with random rollbacks) — mandatory for every pack.
- Golden replays with stored final hashes; intentional balance change = explicit golden update in the same commit.
- Parity harness vs old game: same scenarios run in the old HTML (headless) and Proto3; compare observables (positions, HP, statuses) with tolerance.
- Playwright: every screen with keyboard, pad emulation and touch-only; mobile viewport; screenshots.
- Benchmarks: sim step time, snapshot time, bullets throughput.
- CI order: typecheck → lint (boundaries, determinism bans, file size) → unit/scenario → determinism → goldens → build → Playwright → deploy. Any failure blocks deploy.

---

## 15. Conventions
- Files `kebab-case`, file name = content id. One file answers one question; split when a file exceeds ~300 lines or holds two concepts; never create files of a few trivial lines.
- Character layout:
```
packs/touhou/characters/reimu/
  character.ts  shot.ts  heavy/{neutral,forward,back}.ts
  melee/{jab,lunge,strong,dash-attack}.ts  spell.ts
  skills/{skill-1,skill-2}.ts  passive.ts  cpu.ts
  visuals/{body,weapon}.ts  sounds.ts  i18n/{en,fr}.json  tests/
```
- Items: `packs/touhou/items/<family>/<item>/{item.ts, item.test.ts, visuals.ts?}`.
- Balance numbers live in the content file that uses them (typed `params`), never as magic numbers in systems. Shared constants in `packs/core-combat/tuning/`.
- No global mutable variables outside the kernel state container.

---

## 16. Documentation
- `CLAUDE.md`: rules for working in the repo, commands, checklist per change.
- `PROJECT_MAP.md`: "to change X, open Y" table.
- `docs/ARCHITECTURE.md`, short `README.md` per pack (entry points, public API, events used).
- `docs/adr/NNNN-title.md`: one per important decision with context and rejected options.
- Generated (CI fails if stale): catalog of packs, characters, attacks, items, events, statuses, commands, settings, strings.

---

## 17. Porting plan (validated phase by phase)
1. **Audit** the old game: inventory of every system, character move, item, mode, option, net message, worker route, special case. Output: `docs/INVENTORY.md` with checkboxes (the parity checklist).
2. **Skeleton:** repo, CI, kernel, schemas, snapshot/hash, determinism tests, replay format, lab skeleton, docs skeleton.
3. **core-combat + 2 characters + training mode**, with parity harness.
4. **Online** (rollback, lobby, TURN), determinism under rollback.
5. **Remaining characters**, then **items** in families, each with tests and parity checks.
6. **UI screens**, HUD, audio, YouTube pack, map maker, results, select screen.
7. **Mods loader**, hot reload, i18n FR.
8. Full parity review against INVENTORY.md; switch the main link only then.

---

## 18. Risk register (keep adding)
| # | Risk | Mitigation |
|---|---|---|
| R1 | Transcendental Math differs across browsers | §3.1 ban + `sim.math` |
| R2 | Mods call non-deterministic APIs | §3.7 sanitizer + lab double runs |
| R3 | Unstable sort tie order | §3.4 |
| R4 | Old replays after updates | content hash in replay; refuse with message |
| R5 | Mid-match setting changes | commands only (§6) |
| R6 | Hot reload online | locked (§10.3) |
| R7 | Stale entity references | handles with generation (§4.3) |
| R8 | Snapshot cost / GC at 32 players | schema-generated preallocated copies (§4.2) |
| R9 | Mod state/name collisions | namespaces + explicit overrides (§2.4, §4.2) |
| R10 | Priority wars between hooks | phases + constraints (§5.2) |
| R11 | Infinite event loops | queued events + max depth (§5.4) |
| R12 | Mod exception mid-match online | deterministic disable (§5.5) |
| R13 | Dependencies dying | minimal pinned deps; static output |
| R14 | Over-engineering up front | fix formats now, implement on demand (§17) |
| R15 | RNG shift when content changes | per-pack RNG streams (§3.3) |
| R16 | DOM/GC hitches during fights | HUD in render, Svelte idle during match (§8.3) |
| R17 | Items that rewind/inspect state | `history` service inside state (§4.5) |
| R18 | Render deciding gameplay | view modifiers in sim state (§4.5) |
| R19 | Bullet cap bypass | single spawn path + test (§4.3) |
| R20 | Rollback ghost FX/sounds | event-keyed FX cancel (§7) |
| R21 | Interpolation making things look worse | toggle + screenshot tests (§7) |
| R22 | Mods unavailable on Firefox/Safari/mobile folder API | zip drag & drop fallback (§11) |
| R23 | Host bandwidth at 32 players | packed inputs, measurement (§10.5) |
| R24 | Spectator mid-join needs full state | schema serializer (§10.1) |
| R25 | Docs drifting from code | generated catalogs + CI check (§16) |
| R26 | Hidden-tab throttling stalls net | worker clock (§9) |
| R27 | Parity impossible to exact decimal | tolerance-based parity harness (§14) |
| R28 | localStorage size limits | saves/settings in localStorage (small, versioned); mods/replays in IndexedDB |
| R29 | Save/settings format changes | schema version + migrations |
| R30 | Photosensitive effects | Effects OFF tested; no full-screen flash in OFF |
