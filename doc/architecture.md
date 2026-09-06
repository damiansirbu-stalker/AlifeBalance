# AlifeBalance Architecture

AlifeBalance steers each level's population toward what the level's own spawn configs declare, measured in the engine's own units.
The sensor is the respawn slot ledger. Every smart terrain's `respawn_params` declares how many squads each recipe may hold concurrently.
The engine's `already_spawned[k].num` counts how many are alive.
That count is incremented at spawn (`smart_terrain.script:1759`), decremented when the squad unregisters (`sim_squad_scripted.script:1028`), and persisted in the save.
Summed per (level, bin), actual against capacity gives verdicts with an error fraction, and every correction is proportional to that error. A bin is a human faction or the single MUTANTS aggregate.
The verdicts drive Respawn Pacing (cooldown advance and delay), Spawn Size (member draw for new squads), and Squad Refill (repair of weakened squads). A shared crowded-area check gates all flows.
The engine keeps owning the spawn itself, the recipes, squad selection, and budgets. The mod writes exactly one engine field (`last_respawn_update`) and uses the engine's own `add_squad_member`.

Built on xlibs. `_ab_deps` asserts the minimum xlibs version on load.

Part of a three-mod alife family. **AlifePlus** extends A-Life with new behaviors.
**AlifeBalance** (this mod) modulates rates and counts the engine already owns, and never releases anything. **AlifeGuard** owns all release work and repairs alife state.

## Invariants

- **No steady-state per-frame work.** A throttled 60s timer or discrete engine events only. The refill drain frame-spreads a bounded batch via xslice and stops.
  Full rule: `doc/standards/stalker-code.md` "No Per-Frame Work".
- **Never a release.** Spawn Size and Squad Refill only add members, inside each section's own `npc_in_squad` range. Release work belongs to AlifeGuard.
- **Spawns delayed, never blocked.** Delays never leave more than `max_minutes` remaining, clamp age >= 0, skip fresh smarts, never write the `on_try_respawn` disable flag.
- **Only the controllable population.** The slot ledger counts only spawner-owned squads. The starting population from `fill_start_position`, and event and mod spawns, are excluded.
  None of them set `respawn_point_id` (`sim_board.script:352` vs `smart_terrain.script:1720`). The mod neither boosts nor suppresses a population no lever can replace.
- **Performance first.** Performance is the top priority and outranks features.
  A feature that cannot meet the budget is reworked to fit or removed, including with an X-Ray engine modification. It is never kept at the cost of the budget.
  Only correctness and "never break base gameplay" rank above it. See `doc/standards/stalker-code.md` "Performance is the priority".
- **Use the engine, don't work around it.** Every capability comes from the engine and the Anomaly layer first, always through xlibs.
  Our own code enters only where stock behavior falls short. It escalates from a nudge to a correction, and as a last resort changes the layer itself.
  A layer change is an engine modification or a full-file override. Never reimplement in script what the engine already does.
  See `doc/standards/stalker-code.md` "Use the engine, don't work around it".

## The sensor: slot verdicts

Per level, `ab_smart_recipe.get_level_model` walks the smarts once. It caches the model and recomputes every `MODEL_REFRESH_PASSES` = 3 passes, so condlist flips and AlifePlus mutations converge:

```
per recipe (one filter, both sides: bookkeeping present; faction_controlled with
            the default_faction seeding at smart_terrain.script:1659, read-only):
  max_eff = common ? round_idp(raw * pop_factor) : raw   -- engine math verbatim (:1675-1686)
  cap    += ceil(max_eff) * section fraction per bin      -- ceil is an identity with the
                                                          -- engine's `max > num` gate
  reader  = { smart, key, fracs }                          -- for the live actual read
```

`read_actual(model)` sums `max(0, already_spawned[k].num)` through the readers by the same fractions. The floor guards the unfloored decrement.
Mixed recipes, common for zombied and mutant mixes, attribute cap and num identically, so per-bin comparisons stay exact. Verdicts per bin:

```
band = max(1, cap * 0.25)          -- 1 slot = integer noise floor
open = cap - actual
UNDER  iff open > band  or (actual == 0 and cap > 0)   -- a wiped bin is UNDER at any size
OVER   iff -open > band
err  = |open| / cap, clamped to 1
```

World scope: the same sums aggregated across all levels, served per bin by `get_world_deficit` (feeds Spawn Size).
One known bound: member-only massacres (every squad survives at 1-2 members) hold their slots, so the verdicts read healthy.
The refill floors those squads at their minimum, and squads that then die open slots, handing the case to pacing. Pacing drains that corner on its own.
A member-level deficit sensor was rejected. Every reference (section max, midpoint) reads permanently false under ZCP's post-spawn scaling.

## Respawn Pacing (map scope)

At pass end, per (level, bin), one step per smart per pass enforced across bins (an `acted` set, where a declined try does not consume the slot):

- **Advance** (UNDER): every eligible smart passing `_can_advance`, the crowded-area check, and an open recipe budget (`compute_bin_budget`) gets `push = step * err`.
  Here `step = (resolved_idle - min_minutes*60) / advances`, clamped to the `min_minutes` floor. Pushes under 60s are skipped.
- **Delay** (OVER): mirror direction for smarts whose every produced bin is OVER, clamped so remaining cooldown never exceeds `max_minutes` and age never drops below 0.
- **Crowd delay**: a smart inside a crowded area is delayed at full step regardless of verdicts. Crowding is local overpopulation, and slowing local spawners is the delay lever's native answer.

`_can_advance` mirrors the engine gates that would waste the push (`smart_terrain.script`): disabled (`:1601`), `respawn_only_level` vs actor level (`:1611`).
It also checks actor distance vs `respawn_radius` (`:1619`) and `simulation_objects.available_by_id == false` (`:1630`, nil passes). The peace wish (`:1607`) idles all pacing at pass level.
Residual waste is only the unpredictable case: a budget closing between an advance and the next engine pass, or a chance-recipe roll. Both are corrected on the next pass.
`_resolve_idle` mirrors ZCP's gate selection (`smr_pop.script:1246-1249`, including disabled-at-86400 and the -1 disable).
CTime deltas are built positionally (`date_time.cpp:118-140`, valid past 24h). The full-step delta is cached per smart.

## Spawn Size (world scope)

Function-level patch of `sim_squad_scripted:create_npc`. The body is byte-identical across vanilla unpacked, the demonized overlay, and ZCP's slot, so one wrap serves all installs.
After the base draw, a squad whose faction's WORLD deficit exceeds the band is raised toward `min + strength * deficit * (max - min)` of its own `npc_in_squad` range via `add_squad_member`.
Gates: local bin not OVER, area not crowded, `npc_random` present, no story id.
World scope by design. A faction dented on one map spawns vanilla squads, and pacing covers the local hole.
A Zone-wide collapse brings reinforcements at strength, so the spawn-side levers stack only in the total-massacre case. Verdicts are member-blind, so this lever cannot feed the sensor that drives it.
ZCP's `adjust_squad_size` rescales the result after creation. The relative bias survives and the verdicts are unaffected.

## Squad Refill (squad scope)

Owns the staggered sweep, the mod's only squad walk. It snapshots via `xsquad.collect_squad_ids` and walks 50 squads per 60s timer, a pass over ~400 squads in ~8 minutes.
The pass cadence is the controller's damping.
Per offline squad: body count into its density cell. A squad below its section's declared `npc_in_squad` minimum with a `npc_random` pool becomes a candidate.
At pass end, candidates gated by bin not OVER and area not crowded drain through xslice, 1 squad per frame, 1 member per squad per pass.
Each add uses `add_squad_member` at the commander's position with `register_npc` and `setup_squad_and_group`, the vanilla member-add shape. Every state is re-verified fresh in the drain frame.
Repairs stop at the declared minimum.
This is the repair for the engine's structural blind spot. Nothing in vanilla ever refills a squad, and a lone survivor holds its respawn slot forever.
On GAMMA, ZCP can spawn squads below their LTX minimum (0.55 scaling). The refill enforces the LTX floor, bounded, one-directional, no tug (ZCP scales at spawn only).

## The crowded-area check

Cells keyed by `xlevel.cell_key` (the shared grid convention with AlifeGuard's offline cull) at `switch_distance` granularity, offline bodies only, rebuilt each pass.
Crowded means the own cell at the MCM threshold (default 60), or the 3x3 neighborhood at double (which catches piles straddling a border).
Offline-only is deliberate. The mod's levers act in offline space. Online density is AlifeGuard's online guard and the engine's own `respawn_radius`.
The threshold sits at half AlifeGuard's default cull trigger (120), so the two systems keep a dead zone and do not meet at one line. That assumption is a comment, not a cross-mod read.

## Pipeline

```
TICK (60s, CreateTimeEvent)
  ab_squad_refill.iterate_squads(50): cells += offline bodies, candidates += below-min
  on wrap -> _execute_pass:
    every 3rd pass: clear_models
    per level: model + read_actual -> verdicts {v, err}, world sums    [PASS]
    peace wish? -> skip corrections
    per (level, bin) verdict, acted-set enforced:
      UNDER: _can_advance + !crowded + budget -> lru:sub(step*err)     [ADVANCE]
      OVER:  _can_delay + all bins OVER       -> lru:add(step*err)     [DELAY]
    crowded smarts: _can_delay -> lru:add(step)                        [DELAY crowd]
    refill drain: !OVER + !crowded -> xslice, +1 member/squad          [QUEUE/REFILL/DRAIN]

SPAWN (engine)
  try_respawn gates -> SIMBOARD:create_squad -> create_npc
    -> ab_spawn_size wrap: world deficit past band? raise toward target [SIZE]
  already_spawned[k].num += 1
```

## State and callbacks

All in-memory, reset on load.
State: ab_smart_balance holds `_verdicts`, `_world`, `_last_actual`, `_delta_cache`, `_smart_stats`, `_advance_pending`, `_seen_squads`, `_stats`.
ab_smart_recipe holds `_models`. ab_squad_refill holds `_sweep`, `_cells`, `_candidates`, `_pool_cache` (session-lifetime, LTX is static).
Callbacks: `squad_on_npc_creation` (spawn trace), `on_option_change`, `load_state`, `actor_on_first_update`, `on_try_respawn` (debug trace), and the map-spot menu pair.
One 60s timer, one xslice job (`ab_refill`), one function patch (`create_npc`, ab_spawn_size). Moved cooldowns persist in the engine's own save data.

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, 5-tab tree, reset button |
| `gamedata/scripts/ab_smart_balance.script` | Verdicts, world sums, pacing actuators, public `get_verdict` / `get_world_deficit` / `build_marker_label` / `show_smart_stats` |
| `gamedata/scripts/ab_smart_recipe.script` | Slot capacity model, readers, `read_actual`, bin classification, budget eval, side-effect-free condlist walker |
| `gamedata/scripts/ab_squad_refill.script` | Staggered sweep, density cells, `is_crowded`, refill drain |
| `gamedata/scripts/ab_spawn_size.script` | `create_npc` wrap: world-deficit-proportional member draw |
| `gamedata/scripts/ab_smart_map.script` | PDA marker render-state, right-click menu |
| `gamedata/scripts/ab_test.script` | Console probes + flow test + staged synthetic-massacre system test |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/configs/text/rus/ui_st_mcm_ab.xml` | MCM strings (Russian, pending translation batch) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner |

## MCM

| Setting | Tab | Default | Effect |
|---|---|---|---|
| `enabled` | General | true | Master toggle |
| `crowd_threshold` | General | 60 | Offline bodies per area that make it crowded (20-200) |
| `pacing` | Respawn Pacing | true | Cooldown corrections on/off (verdicts still measured) |
| `advances` | Respawn Pacing | 6 | Passes from full cooldown to floor at maximum depletion (2-8) |
| `min_minutes` | Respawn Pacing | 120 | Cooldown floor after advances (30-360) |
| `max_minutes` | Respawn Pacing | 720 | Remaining-cooldown ceiling after delays (120-1440) |
| `spawn_size` | Spawn Size | true | World-deficit member draw on/off |
| `spawn_size_strength` | Spawn Size | 50 | Scales the deficit before it shapes the draw (0-100) |
| `refill` | Squad Refill | true | Squad repairs on/off |
| `log_level`, `show_markers`, `btn_reset_all` | Development | WARN / false / - | Diagnostics |

`BAND_FRAC`, `BAND_MIN`, `SWEEP_PER_TICK`, `MODEL_REFRESH_PASSES`, `MIN_PUSH_SEC` are tuning constants, not knobs.

## Performance

| Operation | Cost |
|-----------|------|
| Sweep, per timer | 50 squads x ~4-6 luabind (resolve, npc_count, permanence, level). Cells are pure Lua |
| Verdicts, per pass | model cached. `read_actual` is pure Lua field reads over readers. Per-level judge O(bins) |
| Model rebuild, per level per refresh | O(smarts x recipes). 1 `select_value_readonly` per recipe. Duration in `[MODEL]` |
| Per correction | cooldown read, CTime build (cached at full step), write. Durations in `[SUMMARY]` |
| Crowded check | <= 10 hash lookups, pure Lua |
| Refill drain | 1 squad per frame via xslice. Per-frame and completion durations in `[REFILL]` and `[DRAIN]` |
| Spawn Size | spawn-time only. A handful of cached reads per new squad |

All debug logging noops at WARN. Counters are plain field writes.

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla `try_respawn` | Both directions move `last_respawn_update`. Capacity and budget eval mirror the engine's per-recipe selection |
| ZCP | `_resolve_idle` reads ZCP's gate cvar. Slot units make its `squad_size` post-scaling invisible to verdicts. `smr_handle_spawn` substitution means a slot's occupant can be another species, so verdicts regulate DECLARED slots and occupants drift (documented, not fought) |
| AlifePlus | Conquest and infestation mutate `respawn_params`, and the set-point follows within a model refresh. AP never writes `last_respawn_update` |
| AlifeGuard | Culls free slots or thin members like any loss. The crowded-area dead zone keeps refill and cull spatially separated |
| Squad Filler | Superseded (readme Conflicts). Ungated flat-size fill vs verdict-gated declared-bounded repair |
| Warfare | Not supported alongside Smart Balance (readme Compatibility) |
| Mods patching `create_npc` | ab_spawn_size wraps at `on_game_start`. Fn-patches compose, and a full-file override that wins MO2 still composes, so the wrap applies to the winning body |
