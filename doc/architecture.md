# AlifeBalance Architecture

AlifeBalance steers each level's population toward the composition the level's own spawn configs declare. A rolling census counts live NPCs per (level, bin) — a bin is a human faction or the single MUTANTS aggregate — and compares them against the declared capacity computed from the level's `respawn_params`. Under-capacity bins get eligible smarts' `last_respawn_update` advanced (cooldown shortened) and their squads topped up to full size at spawn; over-capacity bins get the cooldown delayed, never beyond one full vanilla cooldown. The engine still owns the spawn itself, the recipe choice, the squad section, and the budgets. AlifeBalance writes exactly one engine field on the timing lever (`last_respawn_update`) and uses the engine's own `add_squad_member` on the strength lever.

Two MCM knobs shape the timing lever: `advances` (1-8, default 4) sets how many census passes carry one smart from full cooldown to the floor (delays use the same step); `Min Minutes` (10-360, default 120) sets the floor on remaining cooldown that the advance direction never pushes below. A third knob, `size_bias` (default on), toggles the strength lever.

Built on xlibs. `_ab_deps` asserts the minimum xlibs version on load.

Part of a three-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** modulates rates and counts the engine already owns and never releases anything (this mod), **AlifeGuard** owns all release work, entities and items, and repairs alife state.

![Layered position](img/layers.png)

---

## Invariants

- **No steady-state per-frame work.** Ongoing work runs on a throttled tick (a fixed interval) or on a discrete engine event (hit, shot, spawn, option change); it never runs continuously every frame. Full rule and rationale: `doc/standards/code-standards.md` "No Per-Frame Work".
- **Never a release.** The size lever only adds members (up to the section's own `npc_in_squad` upper bound); over-capacity is handled by the delay lever alone. Release work belongs to AlifeGuard.
- **Spawns delayed, never blocked.** The delay direction clamps age to >= 0 (one full vanilla cooldown is the ceiling), skips fresh smarts (`last_respawn_update == nil` means the engine gate is already open), and never writes the engine's `on_try_respawn` disable flag.

---

## Bins

A bin is the unit the census counts and the controller corrects:

- Every human faction is its own bin, keyed by its `squad_descr` faction string (`stalker`, `bandit`, `dolg`, ... including `zombied` — zombied squads are combatants).
- Every monster faction (`monster`, `monster_predatory_day/night`, `monster_vegetarian`, `monster_special`, `monster_zombied_*`, `zoo_monster`) folds into the single MUTANTS bin (`ab_smart_recipe.MUTANTS`). Mutant ecology has no per-class parity target; mutants balance against the map's declared mutant capacity as one group.

Squad classification reads `squad.player_id`; section classification reads `xsmart.section_faction`, both through `xcreature.is_monster_faction`. Vermin need no special casing: rat and tushkano squads inflate the actual count and the declared capacity symmetrically.

---

## The declared capacity model

Each level's target comes from its own smart terrains' `respawn_params`, not from anything AlifeBalance authors. Per recipe the engine would consider (mirroring the `try_respawn` gates: bookkeeping present, `faction_controlled` filter):

```
eff_num      = spawn_num condlist value (pick_value_readonly) * mirrored pop factor
per_section  = eff_num / #sections            (engine picks a section uniformly)
capacity    += per_section * (npc_in_squad min + max) / 2   per section, into its bin
```

`spawn_num` caps squads CONCURRENTLY alive from that recipe: the engine increments `already_spawned[k].num` at spawn (`smart_terrain.script:1759`) and decrements it when a squad from the recipe unregisters (`sim_squad_scripted.script:1022-1029`). Summed capacity is therefore a genuine concurrent-population target the engine itself enforces per recipe; AlifeBalance only aggregates it per bin per level. Fractional `spawn_num` (a spawn chance in the engine) stays fractional here as an expected value.

The same walk records, per bin, the eligible smarts (those with a recipe producing the bin) and per smart the set of bins it serves (`smart_bins`, the reverse map the delay gate reads).

The model is cached per level and recomputed every `MODEL_REFRESH_PASSES` (3) census passes, because `respawn_params` is not static: `spawn_num` condlists flip with story infos (`{-bar_deactivate_radar_done} 3, 0`), and runtime mutators (AlifePlus territory conquest / infestation) inject and remove entries. The set-point follows those mutations automatically.

---

## The census

A staggered walk over a `xsquad.collect_squad_ids` snapshot, `CENSUS_PER_TICK` (50) squads per 60-second tick; a pass over ~400 squads completes in ~8 minutes of real time. Per squad: resolve, read `npc_count`, classify `player_id` into a bin, read the level, accumulate. Skipped: empty squads, permanent squads (`xsquad.is_permanent_squad`: story, trader, named-NPC garrisons — seeded outside the respawn system, they would permanently distort actual vs respawn-declared capacity), and companions.

A level enters the judged set (`_known_levels`) when the census first sees a squad there, and stays for the session. A level with zero squads and no session history recovers at vanilla pace until squads reappear.

---

## Verdicts and corrections

At pass end, per known level, per bin the level's recipes can produce:

```
band = max(2, capacity * 0.25)
capacity - actual > band  ->  UNDER   (advance cooldowns, top up squads at spawn)
actual - capacity > band  ->  OVER    (delay cooldowns)
otherwise                 ->  in band (untouched on both levers)
```

Bins whose squads roam the level but which no recipe produces have no actuator and are ignored. Verdicts persist until the next pass and are served to the size lever via `get_verdict(level_id, bin)`.

**Advance** (UNDER): every eligible smart that passes `_can_advance` (floor gate) and has an open recipe budget for the bin (`evaluate_budget_for_bin`) gets one step: `last_respawn_update:sub((resolved_idle - Min Minutes*60) / advances)`. One step per smart per pass.

**Delay** (OVER): every eligible smart that passes `_can_delay` gets one step in the opposite direction: `last_respawn_update:add(step)`, clamped so age never goes below 0. A smart whose recipes also serve an under- or in-band bin is never delayed (`smart_bins` check) — no cross-bin punishment at mixed smarts. Fresh smarts are skipped in both directions.

Worked example at ZCP-resolved idle 21600 (6 game-hours), `advances=4`, `Min Minutes=120`:

```
usable   = 21600 - 7200 = 14400 s = 4 game-hours
step     = 14400 / 4    =  3600 s = 1 game-hour per pass
floor    =  7200 s      = 2 game-hours the engine always ages out itself
ceiling  = age 0        = one full 6-hour vanilla cooldown (delay direction)
```

`_resolve_idle` reads `smr_amain_mcm.get_config("respawn_idle")` when ZCP is loaded and enabled, `smart.respawn_idle` otherwise, so the step always sizes against the gate the engine actually uses.

---

## The size lever (ab_squad_size)

Function-level patch of `sim_squad_scripted:create_npc`, applied at `on_game_start` (the patched body is byte-identical across vanilla unpacked, the demonized overlay, and ZCP's slot, so one wrap serves all installs). After the base draw, a squad whose bin is UNDER on its level is topped up to its section's own `npc_in_squad` upper bound via the engine's `add_squad_member` (registers the member, fires `squad_on_add_npc`; the caller's post-`create_npc` member loop in `sim_board:create_squad` sets the added members up like the drawn ones).

Add-only: members are never removed. Skipped: fixed-roster sections (no `npc_random`), story squads, spawns without a smart (named-location path), unknown bins, and any verdict other than UNDER.

---

## Pipeline

```
TICK (every 60 wall-seconds via CreateTimeEvent)
  |
  v
_census_chunk()
  - pass start: snapshot squad ids (xsquad.collect_squad_ids)
  - walk 50 squads: skip empty/permanent/companion, counts[level][bin] += npc_count
  - on wrap: _finalize_pass(counts)
      - every 3rd pass: ab_smart_recipe.invalidate_models()
      - per known level: model = get_level_model(level)
          per producible bin: verdict = UNDER / OVER / in-band   [PASS]
      - per (level, bin) verdict:
          UNDER: per eligible smart: _can_advance + evaluate_budget_for_bin
                 -> lru:sub(step)                                [ADVANCE]
          OVER:  per eligible smart: _can_delay + all its bins OVER
                 -> lru:add(step), age clamped >= 0              [DELAY]

SPAWN (engine, its own alife tick at any smart whose gate opens)
  - se_smart_terrain:update -> try_respawn
  - cooldown gate: diff > resolved idle ? proceed : skip
  - recipe budget gates, recipe + section picked at random
  - SIMBOARD:create_squad -> squad:create_npc
      -> ab_squad_size wrap: bin UNDER on this level?
         top up to npc_in_squad max via add_squad_member         [SIZE]
  - already_spawned[k].num += 1
```

---

## Engine gates inside try_respawn

`smart_terrain.script:try_respawn` has eight gates (line numbers are vanilla unpacked Anomaly 1.5.3). AlifeBalance affects one.

| Gate                                          | Source line | Our effect |
|-----------------------------------------------|-------------|------------|
| Disabled / on_try_respawn callback            | 1601        | Untouched |
| Peace info                                    | 1607        | Untouched |
| Level filter (`respawn_only_level`)           | 1611        | Untouched |
| Actor distance (`respawn_radius`)             | 1619        | Untouched |
| `respawn_params and already_spawned` exist    | 1625        | Untouched |
| Simulation availability                       | 1630        | Untouched |
| Cooldown timer                                | 1651        | Advance: subtract one step per pass while the bin is UNDER. Delay: add one step while OVER, age clamped >= 0 |
| Per-recipe budget                             | 1696        | Untouched (read-mirrored for eligibility and capacity) |

---

## Population equilibrium

The controller converges each level toward its declared composition, bounded on every side by engine-owned limits:

- **Refill bound**: spawns only happen through the engine's own recipes, so refill can never exceed the per-recipe `spawn_num` caps (concurrent squads) regardless of how many advances accumulate. An advance only moves WHEN the engine reads its own gate, never past the `Min Minutes` floor.
- **Suppression bound**: a delay moves age toward 0 and stops; the worst case for an over-capacity bin is one full vanilla cooldown per cycle, exactly the pace of a freshly-spawned smart. Spawns are never blocked.
- **Massacre response**: a wiped bin reads actual 0 against full declared capacity — the strongest possible correction on both levers. Uniform depletion of a whole level reads every bin UNDER (absolute comparison, not shares), so the level as a whole recovers faster.
- **In band, silence**: on a healthy level every bin sits inside the deadband and AlifeBalance does nothing at all.

If the player ignores a level, its cooldowns still age from game-time alone and the engine spawns at vanilla pace; corrections only redistribute recovery speed within engine bounds.

---

## Design rationale: why advance, not clear

Setting `last_respawn_update = nil` bypasses the cooldown gate entirely — the engine fires on its next alife tick. "Depleted" becomes "spawn now," with no middle ground. Per-step subtract keeps the gate engaged: each pass moves a depleted bin's smarts a bounded step closer, and the `Min Minutes` floor guarantees the engine fires the last leg on its own clock. The delay direction is the exact mirror with the age-0 ceiling. Both directions preserve the engine's ownership of the actual spawn decision.

---

## State and callbacks

Owned state, all in-memory, reset on game load:

| Owner | State | Purpose |
|---|---|---|
| ab_smart_balance | `_census` | staggered pass: id snapshot, cursor, accumulating counts |
| ab_smart_balance | `_verdicts[level_id][bin]` | UNDER/OVER from the last pass; read by ab_squad_size via `get_verdict` |
| ab_smart_balance | `_last_counts` | last pass counts (status display) |
| ab_smart_balance | `_known_levels[level_id]` | judged level set, survives model refresh |
| ab_smart_balance | `_delta_cache[smart_id]` | cached `{ delta, advance, usable }`, invalidated on option change or reset |
| ab_smart_balance | `_smart_stats[smart_id]` | per-smart advance/delay bookkeeping for the right-click "Show stats" tip |
| ab_smart_balance | `_advance_pending[smart_id]` | set on advance, consumed on next spawn at the smart; drives `from_us` in `[SPAWN]` |
| ab_smart_balance | `_seen_squads[squad.id]` | dedup set for spawn tracing (DEBUG only) |
| ab_smart_balance | `_stats` | diagnostic counters (censused, skipped, passes, advances, delays, size_adds, spawns, ...) |
| ab_smart_recipe | `_models[level_id]` | capacity + eligibility + smart_bins per level, recomputed on cadence |
| ab_smart_map | `_markers[smart_id]` | linger expiry per marked smart |

Callbacks: `squad_on_npc_creation`, `on_option_change`, `load_state`, `actor_on_first_update`, `on_try_respawn`, `map_spot_menu_add_property`, `map_spot_menu_property_clicked`. One `CreateTimeEvent` timer at 60-second wall interval. One function-level patch: `sim_squad_scripted:create_npc` (ab_squad_size).

No persistence for AlifeBalance state. Advanced or delayed `last_respawn_update` values survive in the engine's own save data via `utils_data.w_CTime / r_CTime`.

Not owned by AlifeBalance: which recipe the engine picks (random over open-budget), which squad section within that recipe (random), ZCP `smr_handle_spawn` substitution and `adjust_squad_size` scaling, online cap enforcement (GAMMA Dynamic Despawner, AlifeGuard).

---

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, UI definition, button handlers |
| `gamedata/scripts/ab_smart_balance.script` | Census walk, verdicts, advance/delay actuators, per-smart CTime delta cache, public `get_verdict` + `marker_label` + `show_smart_stats` |
| `gamedata/scripts/ab_smart_recipe.script` | Per-level capacity/eligibility model, bin classification, single-pass budget evaluation, side-effect-free condlist walker |
| `gamedata/scripts/ab_squad_size.script` | `create_npc` fn-patch: add-only squad top-up for under-capacity bins |
| `gamedata/scripts/ab_smart_map.script` | PDA marker render-state, right-click menu (teleport, show stats) |
| `gamedata/scripts/ab_test.script` | Console probes: `census()` forces a full pass, `status()` prints per-level bin state |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/configs/text/rus/ui_st_mcm_ab.xml` | MCM strings (Russian) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner (512x50) |

---

## MCM

| Setting | Tab | Type | Default | Range | Effect |
|---|---|---|---|---|---|
| `enabled` | Smart Balance | check | true | - | Master toggle |
| `advances` | Smart Balance | track | 4 | 1-8 | Census passes from full cooldown to the floor; delay step is the same size |
| `Min Minutes` | Smart Balance | track | 120 | 10-360 | Minimum cooldown remaining after every advance, in game minutes |
| `size_bias` | Smart Balance | check | true | - | Top up under-capacity squads to their section max at spawn |
| `log_level` | Development | list | WARN | - | ERROR / WARN / INFO / DEBUG |
| `show_markers` | Development | check | false | - | Green PDA marker on every corrected smart, 5-min linger, right-click teleport / stats |
| `btn_reset_all` | Development | button | - | - | Restore all MCM settings to factory defaults (closes via On_Cancel, see ab_mcm.script comment) |

Deadband width, census chunk size, and model refresh cadence are tuning constants (`BAND_FRAC`, `BAND_MIN`, `CENSUS_PER_TICK`, `MODEL_REFRESH_PASSES`), not knobs.

---

## Performance

| Operation | Cost |
|-----------|------|
| Census, per tick | 50 squads x ~4-6 luabind each (alife_object, npc_count, permanence check, level lookup); pure Lua accumulation |
| `get_level_model`, per level per refresh | O(S * R * Q) smarts x recipes x sections; 1 `pick_value_readonly` per recipe, cached `section_faction` / `npc_range` reads per section |
| `evaluate_budget_for_bin`, per UNDER bin per eligible smart per pass | O(R * Q); 1 `pick_value_readonly` per recipe |
| `_get_advance_params`, first call per smart | 2 CTime constructions + 2 `setHMSms` + 1 operator-. Cached |
| Per advance / delay | 3 luabind: `last_respawn_update` read, `:sub`/`:add` with cached delta, write back |
| Size top-up, per under-capacity spawn | `add_squad_member` per added NPC (engine's own member-add path); zero cost for in-band/over bins |
| Pass cadence | corrections run once per completed pass (~8 min real time at ~400 squads), not per tick |

---

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla `try_respawn` | Reads `last_respawn_update` inside the cooldown gate; both levers move the same field. Capacity and budget eval mirror the engine's per-recipe factor selection verbatim (`smart_terrain.script:1675-1686`). |
| ZCP (forked `try_respawn`) | Gates against `smr_amain_mcm.get_config("respawn_idle")`; `_resolve_idle` reads the same cvar so steps size against ZCP's actual gate. Division of labor: ZCP owns which section/species spawns (`smr_handle_spawn` substitutes BEFORE `create_npc`, so the size lever classifies the final section) and global size factors (`adjust_squad_size` scales AFTER `create_npc`, multiplicative, so the top-up carries through). AlifeBalance owns only the per-bin correction. |
| ZCP `smr_handle_spawn` | Substitution ignores which smart was advanced, so per-bin targeting degrades to a statistical tendency; the census sees the post-substitution population and self-corrects. Documented, not fought. |
| AlifePlus smart mutator | Conquest/swarm/infest inject entries into the same `respawn_params` the capacity model reads: the set-point follows AP's mutations (AlifeBalance populates a conquest, and unwinds it after decay) with no coordination code. AP never writes `last_respawn_update`. |
| AlifeGuard | Density cull and this controller are spatially orthogonal and both saturate (cull stops at its density target, advance stops at capacity/floor). Despawns lower the census like any other loss; no event coupling. |
| Redone Collection, GAMMA NPC Spawns | Pure LTX configuration — exactly what the capacity model reads as the target. No conflict. |
| Night Mutants | Parallel `SIMBOARD:create_squad` path; spawned squads are censused normally. |
| Nocturnal Mutants | Raw `alife_create`, bypasses smart terrains; censused if squads register, otherwise independent. |
| Warfare | Not supported alongside Smart Balance; toggle AlifeBalance off when running Warfare (see readme Compatibility). |
| Mods patching `sim_squad_scripted:create_npc` | ab_squad_size wraps the function at `on_game_start`; other fn-patches compose (each wraps the previous). A full-file override of `sim_squad_scripted.script` that wins the MO2 conflict still composes — the wrap applies to whatever body won. |
