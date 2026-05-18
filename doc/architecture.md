# AlifeBalance Architecture

## The big idea

We do not spawn. The engine already spawns. When combat thins a faction on a level, we nudge a smart on that level whose recipes can produce that faction to refill sooner.

The nudge is a single nil-write to one server-entity field. The engine does the rest on its own schedule, using its own configured spawn pool.

## The signal

Per level, one death counter per faction. Twelve stalker factions (stalker, bandit, csky, dolg, freedom, ecolog, killer, monolith, renegade, army, greh, isg) plus mutant prop keys (monster, monster_predatory_day, monster_predatory_night, monster_vegetarian, monster_zombied_day, monster_zombied_night, monster_special) plus zombied. Faction is taken from `squad.player_id` at death time, level from `xlevel.get_level_id(npc or squad)`.

Protected squads (story, companions, task targets) and cowardly-species mutants (rats, tushkanos, flesh, zombies, karliks) do not count.

## The threshold

Per (level, faction), lazy-computed on first death for that pair, cached for the session:

`threshold = MAX(npc_in_squad upper bound) across every section that produces the faction, in every recipe of every eligible smart on the level`.

A section "produces the faction" when `xsmart.section_faction(section) == faction`. The function reads the section's `faction` INI field from `configs/misc/squad_descr/*.ltx` -- the same value the engine assigns to `squad.player_id` at `SIMBOARD:create_squad` time.

The engine creates ONE squad per `try_respawn` (`smart_terrain.script:1714-1722`) with `npc_in_squad` random within range. Setting the threshold to this MAX guarantees, between any two nudges of the same (level, faction), that at least as many NPCs of that faction died as the next refill could possibly spawn. Refill never outpaces death, in absolute terms, per level and faction.

A level with rat-lair smarts producing the dying faction has threshold around 20. A stalker-only level around 4-5. Derived from engine LTX, not user-configurable.

## The pick

Eligibility has two layers:

**Static (cached once per pair):** the smart is on the level AND at least one section across all its `respawn_params` recipes matches the faction via `xsmart.section_faction(section) == faction`. Recipe content is static across the session, so the eligible list is computed once and cached in `_eligible[level_id][faction]`.

**Dynamic (re-evaluated every tick):** the smart has at least one matching recipe with open budget right now. "Open budget" means `max > already_spawned[k].num` where `max = tonumber(pick_section_from_condlist(db.actor, smart, recipe.num))` -- the same condlist eval the engine itself runs at `smart_terrain.script:1669`.

When the counter crosses threshold, AlifeBalance filters the cached eligible list by the dynamic budget check and picks a random smart from the filtered set. If the filtered set is non-empty, nudge fires and the counter resets. If the filtered set is empty (every recipe-eligible smart on the level is at budget cap), AlifeBalance defers: no nudge, counter holds, retry next tick. The counter holds pressure across ticks; when any eligible smart frees a slot, the next tick fires the deferred nudge.

Props (`accepts_faction`, `has_faction`) are not checked. Routing acceptance via props is decoupled from recipe content -- a smart can accept a faction for routing via `props.all = 1` while its recipes spawn entirely different factions. Only recipe content controls what a refill at this smart can actually produce.

## The nudge

One write on the chosen smart's server entity: `smart.last_respawn_update = nil`.

nil short-circuits the engine cooldown gate at `smart_terrain.script:1651` ("== nil or diffSec > respawn_idle") and the ZCP cooldown gate at `smr_pop.script:352-354`. Save round-trips nil correctly via `utils_data.w_CTime` / `r_CTime` at `utils_data.script:253-286`.

That is the entire nudge. No backdated timestamp. No budget decrement. No injection into `respawn_params`. No engine bookkeeping touched.

## What the engine does

On its next alife tick at the nudged smart, `try_respawn` reads `last_respawn_update`, sees nil, passes the cooldown gate. Engine walks its recipes, finds those with open budget (`max > already_spawned[k].num`), picks one at random, picks a squad section at random from that recipe's pool, calls `SIMBOARD:create_squad`. The squad spawns. The engine increments `already_spawned[k].num`. The cooldown timestamp is set to "now."

We never touch budget. The engine's own death-decrement at `sim_squad_scripted.script:1027-1029` owns it.

## What we own

- `_deaths[level_id][faction]`: death counter per (level, faction), reset on a fired nudge (NOT on a deferred tick).
- `_eligible[level_id][faction]`: cached array of smart_ids whose recipes produce the faction. Recipe-content derived, static for the session.
- `_thresholds[level_id][faction]`: cached MAX npc_in_squad upper bound across matching sections only.
- `_smart_stats[smart_id]`: per-smart nudge bookkeeping for right-click "Show stats" tip.
- `_seen_squads[squad.id]`: dedup set for spawn tracing (DEBUG only).
- `_markers[smart_id]`: linger expiry per marked smart.
- `_stats`: 7 diagnostic counters (deaths, counted, protected, trash, ticks, nudges, spawns).
- Callbacks: `squad_on_npc_death`, `squad_on_npc_creation`, `on_option_change`, `load_state`, `actor_on_first_update`, `map_spot_menu_add_property`, `map_spot_menu_property_clicked`.
- One `CreateTimeEvent` timer at 60-second wall interval.

No persistence. State resets on game load. Already-written `last_respawn_update = nil` values survive in the engine's own save data (utils_data.w_CTime / r_CTime are nil-aware).

## What we do not own

Which recipe the engine picks (random over eligibles). Which squad section within that recipe (random). NPC count within the section's `npc_in_squad` range (random). Substitution (ZCP `smr_handle_spawn` may swap squad sections). Online cap enforcement (Dynamic Despawner, AlifeGuard).

## Pipeline

```
DEATH (engine fires squad_on_npc_death)
  |
  v
_on_npc_death(squad, npc, killer)
  - if disabled: skip
  - _stats.deaths += 1
  - if xsquad.is_protected: _stats.protected += 1, skip
  - if xcreature.is_cowardly_species: _stats.trash += 1, skip
  - faction  = squad.player_id
  - level_id = xlevel.get_level_id(npc or squad)
  - _stats.counted += 1
  - _deaths[level_id][faction] += 1


TICK (every 60 wall-seconds via CreateTimeEvent)
  |
  v
_periodic_tick()
  - ResetTimeEvent first
  - _stats.ticks += 1
  - for each (level_id, by_faction) in _deaths:
      for each (faction, count) in by_faction:
        smarts, threshold = _get_eligible_and_threshold(level_id, faction)  -- lazy cached, recipe-derived
        if count >= threshold and #smarts > 0:
          with_budget = filter smarts by _can_spawn_faction(smart, faction)  -- recipe match + open budget
          if #with_budget > 0:
            smart = with_budget[random]
            smart.last_respawn_update = nil
            by_faction[faction] = 0       -- counter resets only when nudge fires
            _stats.nudges += 1
            if show_markers: mark with linger
          -- else: defer, no nudge, counter holds, retry next tick
  - prune expired markers


ENGINE (its own alife tick on the nudged smart)
  - se_smart_terrain:update() -> try_respawn()
  - cooldown gate (smart_terrain.script:1651): nil -> pass
  - budget gate (smart_terrain.script:1696): max > already_spawned[k].num
  - picks recipe at random from eligibles
  - picks squad section at random from recipe.squads
  - SIMBOARD:create_squad(smart, squad_section)
  - sets squad.respawn_point_id = smart.id
  - increments already_spawned[k].num
  - sets last_respawn_update = curr_time
```

## Engine gates inside try_respawn

`smart_terrain.script:1597-1762` has eight gates. We affect one:

| Gate | Source line | Our effect |
|------|-------------|-----------|
| Disabled / on_try_respawn callback | 1603 | Untouched |
| Peace info | 1607 | Untouched |
| Level filter (`respawn_only_level`) | 1611 | Untouched |
| Actor distance (`respawn_radius`) | 1619 | Untouched |
| `respawn_params and already_spawned` exist | 1625 | Untouched |
| Simulation availability | 1630 | Untouched |
| Cooldown timer | 1651 | Set `last_respawn_update = nil` |
| Per-recipe budget | 1696 | Untouched |

## ZCP integration

ZCP's `smr_pop.smart_can_respawn` (`smr_pop.script:342-368`) replaces the cooldown gate. It returns `true` immediately when `last_respawn_update == nil` (`:352-354`). Our nudge works identically under ZCP.

ZCP's `smr_handle_spawn` (`smr_pop.script:288-339`) substitutes the squad section in flight. The replacement squad still gets `respawn_point_id` and `respawn_point_prop_section` set, so the engine's own budget accounting stays intact. The substituted section may have a different `faction` field than the original; ZCP's design assumes intentional substitution by the modpack author, and AlifeBalance has no opinion on the outcome.

ZCP does not ship its own squad_descr overrides for vanilla sections; `xsmart.section_faction` reads the same values under vanilla and ZCP.

## Population invariant

For any single (level, faction) pair, between any two consecutive nudges of that pair, the engine cannot refill more NPCs of that faction than have died.

Proof: a nudge fires only when `_deaths[level_id][faction] >= MAX(npc_in_squad upper bound)` across the matching sections in eligible smarts on the level. A nudge causes at most one `try_respawn` at the chosen smart, which selects one recipe with open budget and creates at most one squad. If the engine picks a matching section from the recipe's pool, that squad has at most MAX in NPCs of the target faction; if the engine picks a non-matching section (mixed-faction pool), zero NPCs of the target faction spawn. Either way: refill <= death.

After a nudge fires, the counter resets to 0. After a deferred tick (all eligibles at budget cap), the counter is unchanged -- pressure accumulates until any eligible smart frees a slot.

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, UI definition, button handlers |
| `gamedata/scripts/ab_pacing.script` | Death handler, periodic tick, threshold cache, nudge, markers, teleport |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner (512x50) |

## MCM Settings

| Setting | Tab | Default | Effect |
|---------|-----|---------|--------|
| `enabled` | General | true | Master toggle |
| `show_markers` | Development | false | Green PDA marker on every nudged smart, 5-min linger, right-click teleport |
| `log_level` | Development | WARN | Logger verbosity (ERROR/WARN/INFO/DEBUG) |

No threshold knob. Threshold is per (level, faction), derived from `ini_sys:r_string_ex(squad_section, "npc_in_squad")` over the level's eligible smarts.

## Performance

| Operation | Cost |
|-----------|------|
| Per death | O(1): protection check, cowardly check, faction read, level lookup, counter increment |
| _get_eligible_and_threshold (first call per pair) | O(S * K) over smarts on level * squads per recipe, 2 luabind `ini_sys:r_string_ex` per squad (faction + npc_in_squad, medium each). Cached. |
| _get_eligible_and_threshold (cache hit) | O(1) |
| _can_spawn_faction (per tick per eligible smart) | O(K) over recipes * squads, 1 luabind `pick_section_from_condlist` per recipe + 1 `ini_sys:r_string_ex` per squad until first match. Not cached -- budget is dynamic. |
| Per tick (60s) | O(L * F) over death pairs * O(E * K) budget check per pair, plus a one-time compute per new pair |
| Per nudge | O(1): one field write + optional mark |
| Marker prune | O(M) markers per tick |
| Persistence | None. State resets on game load. |

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla try_respawn | Reads `last_respawn_update` at `smart_terrain.script:1651`. nil short-circuits. |
| ZCP (forked try_respawn) | Reads the same field at `smr_pop.script:352-354`. nil short-circuits. |
| Redone Collection | Pure LTX configuration. AlifeBalance writes runtime state. No conflict. |
| GAMMA NPC Spawns | Pure LTX. No conflict. |
| Night Mutants | Parallel `SIMBOARD:create_squad` path. Spawned squads still get `respawn_point_id` set. No conflict. |
| Nocturnal Mutants | Raw `alife_create`. Bypasses smart terrains entirely. Independent. |
| GAMMA Dynamic Despawner | Enforces online cap on the despawning side. Despawn does not fire `squad_on_npc_death`, no false nudges. |
| AlifeGuard | Squad-aware despawner. Same: despawn != death. |
| AlifePlus | Reactive A-Life framework. Different fields and patterns. No overlap. |
| Warfare | `faction_controlled` smarts (16 of ~490) have an injected respawn_params entry `[faction_controlled_<faction>]` whose `.squads` field is the vanilla `squads_by_faction[faction]` list. Our `xsmart.section_faction` reads those vanilla sections normally. Works. |

No base script edits. No engine patches. Runtime callbacks only.
