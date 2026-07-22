# AlifeBalance Architecture

AlifeBalance steers each level's population toward what the level's own spawn configs declare, measured in the engine's own units. The sensor is the respawn slot ledger: every smart terrain's `respawn_params` declares how many squads each recipe may hold concurrently, and the engine's `already_spawned[k].num` counts how many are alive (incremented at spawn, `smart_terrain.script:1759`; decremented when the squad unregisters, `sim_squad_scripted.script:1028`; both persisted in the save). Summed per (level, bin) — a bin is a human faction or the single MUTANTS aggregate — actual vs capacity gives verdicts with an error fraction, and every correction is proportional to that error. 3 systems act on the verdicts: Respawn Pacing (cooldown advance/delay), Spawn Size (member draw for new squads), Squad Refill (repair of weakened squads). A shared crowded-area check gates all flows. The engine keeps owning the spawn itself, recipes, squad selection, and budgets; the mod writes exactly one engine field (`last_respawn_update`) and uses the engine's own `add_squad_member`.

Built on xlibs. `_ab_deps` asserts the minimum xlibs version on load.

Part of a three-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** modulates rates and counts the engine already owns and never releases anything (this mod), **AlifeGuard** owns all release work and repairs alife state.

## Invariants

- **No steady-state per-frame work.** Throttled tick (60s) or discrete engine events only; the refill drain frame-spreads a bounded batch via xslice and stops. Full rule: `doc/standards/code-standards.md` "No Per-Frame Work".
- **Never a release.** Spawn Size and Squad Refill only add members, inside each section's own `npc_in_squad` range. Release work belongs to AlifeGuard.
- **Spawns delayed, never blocked.** Delays never leave more than `max_minutes` remaining, clamp age >= 0, skip fresh smarts, never write the `on_try_respawn` disable flag.
- **Only the controllable population.** The slot ledger excludes what the spawners do not own (the starting population from `fill_start_position`, event and mod spawns — none set `respawn_point_id`, `sim_board.script:352` vs `smart_terrain.script:1720`). The mod neither boosts nor suppresses population no lever can replace.

## The sensor: slot verdicts

Per level, `ab_smart_recipe.get_level_model` walks the smarts once (cached, recomputed every `MODEL_REFRESH_PASSES` = 3 passes, so condlist flips and AlifePlus conquest/infestation mutations converge):

```
per recipe (one filter, both sides: bookkeeping present; faction_controlled with
            the default_faction seeding at smart_terrain.script:1659, read-only):
  max_eff = common ? round_idp(raw * pop_factor) : raw   -- engine math verbatim (:1675-1686)
  cap    += ceil(max_eff) * section fraction per bin      -- ceil is an identity with the
                                                          -- engine's `max > num` gate
  reader  = { smart, key, fracs }                          -- for the live actual read
```

`read_actual(model)` sums `max(0, already_spawned[k].num)` through the readers by the same fractions (the floor guards the unfloored decrement; mixed recipes — common for zombied/mutant mixes — attribute cap and num identically, so per-bin comparisons stay exact). Verdict per bin:

```
band = max(1, cap * 0.25)          -- 1 slot = integer noise floor
open = cap - actual
UNDER  iff open > band  or (actual == 0 and cap > 0)   -- a wiped bin is UNDER at any size
OVER   iff -open > band
err  = |open| / cap, clamped to 1
```

World scope: the same sums aggregated across all levels, served per bin by `get_world_deficit` (feeds Spawn Size). Known bound, stated plainly: member-only massacres (every squad survives at 1-2 members) hold their slots, so the verdicts read healthy; the refill floors those squads at their minimum, and squads that then die open slots, handing the case to pacing — the corner self-drains. A member-level deficit sensor was rejected: every reference (section max, midpoint) reads permanently false under ZCP's post-spawn scaling.

## Respawn Pacing (map scope)

At pass end, per (level, bin) verdict, one step per smart per pass enforced across bins (an `acted` set; a declined try does not consume the slot):

- **Advance** (UNDER): every eligible smart passing `_can_advance` + the crowded-area check + an open recipe budget for the bin (`evaluate_budget_for_bin`) gets `push = step * err`, where `step = (resolved_idle - min_minutes*60) / advances`. Clamped to the `min_minutes` floor; pushes under 60s skipped.
- **Delay** (OVER): mirror direction for smarts whose every produced bin is OVER, clamped so remaining cooldown never exceeds `max_minutes` and age never drops below 0.
- **Crowd delay**: a smart inside a crowded area is delayed at full step regardless of verdicts — crowding is local overpopulation, and slowing local spawners is the delay lever's native answer.

`_can_advance` mirrors every engine gate that would waste the push (`smart_terrain.script` lines): disabled (`:1601`), `respawn_only_level` vs actor level (`:1611`), actor distance vs `respawn_radius` (`:1619`), `simulation_objects.available_by_id == false` (`:1630`, nil passes — engine semantics). The peace wish (`:1607`) idles all pacing at pass level. Residual waste is only the unpredictable pair — a budget closing between advance and engine tick, a chance-recipe roll — both self-correcting next pass. `_resolve_idle` mirrors ZCP's gate selection exactly (`smr_pop.script:1246-1249`, including disabled-at-86400 and the -1 disable). CTime deltas are built positionally (`date_time.cpp:118-140`, valid past 24h); the full-step delta is cached per smart.

## Spawn Size (world scope)

Function-level patch of `sim_squad_scripted:create_npc` (byte-identical body across vanilla unpacked, the demonized overlay, and ZCP's slot — one wrap serves all installs). After the base draw, a squad whose faction's WORLD deficit exceeds the band is raised toward `min + strength * deficit * (max - min)` of its own `npc_in_squad` range via `add_squad_member`. Gates: local bin not OVER, area not crowded, `npc_random` present, no story id. World scope by design: a faction dented on one map spawns vanilla squads (pacing covers the local hole); Zone-wide collapse brings reinforcements at strength — the 2 spawn-side levers stack only in the total-massacre case. Verdicts are member-blind, so this lever cannot feed the sensor that drives it. ZCP's `adjust_squad_size` rescales the result after creation; the relative bias survives and the verdicts are unaffected.

## Squad Refill (squad scope)

Owns the staggered sweep — the mod's only squad walk (snapshot via `xsquad.collect_squad_ids`, 50 squads per 60s tick, a pass over ~400 squads in ~8 minutes; the pass cadence IS the controller's damping). Per offline squad: body count into its density cell; below its section's declared `npc_in_squad` minimum with a `npc_random` pool = candidate. At pass end candidates gated by bin not OVER + area not crowded drain through xslice, 1 squad per frame, 1 member per squad per pass, via `add_squad_member` at the commander's position + `register_npc` + `setup_squad_and_group` (the vanilla member-add shape); every state re-verified fresh in the drain frame. Repairs stop at the declared minimum. This is the repair for the engine's structural blind spot: nothing in vanilla ever refills a squad, and a lone survivor holds its respawn slot forever. On GAMMA, ZCP can spawn squads below their LTX minimum (0.55 scaling); the refill enforces the LTX floor — bounded, one-directional, no tug (ZCP scales at spawn only).

## The crowded-area check

Cells keyed by `xlevel.cell_key` (the shared grid convention with AlifeGuard's offline cull) at `switch_distance` granularity, offline bodies only, rebuilt each pass. Crowded = own cell at the MCM threshold (default 60) OR the 3x3 neighborhood at double (catches piles straddling a border). Offline-only is deliberate: the mod's levers act in offline space; online density is AlifeGuard's online guard and the engine's own `respawn_radius`. The threshold sits at half AlifeGuard's default cull trigger (120) so the 2 systems keep a dead zone instead of meeting at one line; that assumption is a comment, not a cross-mod read.

## Pipeline

```
TICK (60s, CreateTimeEvent)
  ab_squad_refill.sweep_chunk(50): cells += offline bodies, candidates += below-min
  on wrap -> _finalize_pass:
    every 3rd pass: invalidate_models
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

All in-memory, reset on load. ab_smart_balance: `_verdicts`, `_world`, `_last_actual`, `_delta_cache`, `_smart_stats`, `_advance_pending`, `_seen_squads`, `_stats`. ab_smart_recipe: `_models`. ab_squad_refill: `_sweep`, `_cells`, `_candidates`, `_pool_cache` (session-lifetime, LTX is static). Callbacks: `squad_on_npc_creation` (spawn trace), `on_option_change`, `load_state`, `actor_on_first_update`, `on_try_respawn` (debug trace), map-spot menu pair. One 60s timer, one xslice job (`ab_refill`), one function patch (`create_npc`, ab_spawn_size). Moved cooldowns persist in the engine's own save data.

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, 5-tab tree, reset button |
| `gamedata/scripts/ab_smart_balance.script` | Verdicts, world sums, pacing actuators, public `get_verdict` / `get_world_deficit` / `marker_label` / `show_smart_stats` |
| `gamedata/scripts/ab_smart_recipe.script` | Slot capacity model, readers, `read_actual`, bin classification, budget eval, side-effect-free condlist walker |
| `gamedata/scripts/ab_squad_refill.script` | Staggered sweep, density cells, `is_crowded`, refill drain |
| `gamedata/scripts/ab_spawn_size.script` | `create_npc` wrap: world-deficit-proportional member draw |
| `gamedata/scripts/ab_smart_map.script` | PDA marker render-state, right-click menu |
| `gamedata/scripts/ab_test.script` | Console probes + flow test + staged synthetic-massacre system test |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/configs/text/rus/ui_st_mcm_ab.xml` | MCM strings (Russian; pending translation batch) |
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
| Sweep, per tick | 50 squads x ~4-6 luabind (resolve, npc_count, permanence, level); cells pure Lua |
| Verdicts, per pass | model cached; `read_actual` = pure Lua field reads over readers; per-level judge O(bins) |
| Model rebuild, per level per refresh | O(smarts x recipes); 1 `pick_value_readonly` per recipe; duration in `[MODEL]` |
| Per correction | cooldown read + CTime build (cached at full step) + write; durations in `[SUMMARY]` |
| Crowded check | <= 10 hash lookups, pure Lua |
| Refill drain | 1 squad per frame via xslice; per-frame and completion durations in `[REFILL]`/`[DRAIN]` |
| Spawn Size | spawn-time only; a handful of cached reads per new squad |

All debug logging noops at WARN; counters are plain field writes.

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla `try_respawn` | Both directions move `last_respawn_update`; capacity/budget eval mirror the engine's per-recipe selection verbatim |
| ZCP | `_resolve_idle` reads ZCP's gate cvar; slot units make its `squad_size` post-scaling invisible to verdicts; `smr_handle_spawn` substitution means a slot's occupant can be another species — verdicts regulate DECLARED slots, occupants drift (documented, not fought) |
| AlifePlus | Conquest/infestation mutate `respawn_params`; the set-point follows within a model refresh. AP never writes `last_respawn_update` |
| AlifeGuard | Culls free slots or thin members like any loss; the crowded-area dead zone keeps refill and cull spatially separated |
| Squad Filler | Superseded (readme Conflicts); ungated flat-size fill vs verdict-gated declared-bounded repair |
| Warfare | Not supported alongside Smart Balance (readme Compatibility) |
| Mods patching `create_npc` | ab_spawn_size wraps at `on_game_start`; fn-patches compose, and a full-file override that wins MO2 still composes — the wrap applies to the winning body |
