# AlifeBalance Architecture

## The big idea

We do not spawn. The engine already spawns. When a squad takes losses in combat, we tell the engine that the smart that spawned that squad is allowed to refill it sooner.

The nudge is a single write to one server-entity field. The engine does the rest on its own schedule, using its own configured spawn pool.

## The signal

Per origin smart, one death counter. When an NPC dies, we look at `squad.respawn_point_id` -- the smart that spawned the squad. We increment that smart's counter.

Squads without `respawn_point_id` (scripted, save-loaded with broken link, Warfare-managed) are skipped. There is no origin to nudge.

## The threshold

Per smart, the threshold is `MAX(npc_in_squad upper bound)` across every squad section the smart can spawn. Computed lazily on first death from that smart, cached.

Engine creates ONE squad per `try_respawn` (`smart_terrain.script:1714-1722`) with `npc_in_squad` random within range. Setting the threshold to this MAX guarantees that, between any two nudges of the same smart, at least as many NPCs died as the next refill could possibly spawn. Refill never outpaces death, in absolute terms.

A smart whose recipes include rat-lair squads has a threshold around 20. A stalker-only smart has a threshold around 4-5. No MCM knob: the threshold is derived from engine data.

## The tick

A real-time 60-second tick walks the death counters. For each smart over its threshold, the tick nudges that smart and resets that smart's count.

The death handler does not trigger nudges. It only increments counters. Producer and consumer are decoupled. Bursts of deaths cannot produce bursts of nudges.

## The nudge

One write on the chosen smart's server entity. Set `last_respawn_update = nil`.

`nil` short-circuits the engine's cooldown gate at `smart_terrain.script:1651`:
```
if self.last_respawn_update == nil or curr_time:diffSec(...) > self.respawn_idle then
```

Same short-circuit in ZCP at `smr_pop.script:352-354`. Save round-trips `nil` correctly via `utils_data.w_CTime` / `r_CTime` (`utils_data.script:253-286`): nil writes a `false` bool, reads back as nil.

That is the entire nudge. No backdated timestamp. No budget decrement. No injection into `respawn_params`. No engine bookkeeping touched.

## What the engine does

On its next alife tick at the nudged smart, `try_respawn` reads `last_respawn_update`, sees nil, passes the cooldown gate. Engine walks its recipes, finds those with open budget (`max > already_spawned[k].num`), picks one at random, picks a squad section at random from that recipe's pool, calls `SIMBOARD:create_squad`. The squad spawns. The engine increments `already_spawned[k].num`. The cooldown timestamp is set to "now."

We never touch budget. The engine's own death-decrement at `sim_squad_scripted.script:1027-1029` owns it.

## What we own

- `_deaths[smart_id]`: death counter per origin smart, reset on nudge.
- `_thresholds[smart_id]`: cached MAX npc_in_squad upper bound per smart.
- `_stats`: 4 diagnostic counters (deaths, counted, ticks, nudges).
- Callbacks: `squad_on_npc_death`, `on_option_change`, `load_state`, `actor_on_first_update`.
- One `CreateTimeEvent` timer at 60-second wall interval.

No persistence. State resets on game load. Already-written `last_respawn_update = nil` values survive in the engine's own save data.

## What we do not own

Which faction the engine spawns from the smart's pool (random over eligible recipes). Which squad section within that recipe (random). NPC count within the section's `npc_in_squad` range (random). Substitution (ZCP `smr_handle_spawn` may swap squad sections). Online cap enforcement (Dynamic Despawner, AlifeGuard).

## Pipeline

```
DEATH (engine fires squad_on_npc_death)
  |
  v
_on_npc_death(squad, npc, killer)
  - if not enabled: skip
  - if no squad: skip
  - _stats.deaths += 1
  - smart_id = squad.respawn_point_id
  - if nil (no origin): skip
  - _stats.counted += 1
  - _deaths[smart_id] += 1


TICK (every 60 wall-seconds via CreateTimeEvent)
  |
  v
_periodic_tick()
  - ResetTimeEvent first
  - _stats.ticks += 1
  - for each (smart_id, count) in _deaths:
      smart = alife_object(smart_id)
      if not smart: skip (entity gone)
      threshold = _get_threshold(smart_id, smart)  -- lazy compute + cache
      if count >= threshold:
          smart.last_respawn_update = nil
          _deaths[smart_id] = 0
          _stats.nudges += 1


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

After both touchable gates pass, the engine's own random pick from `available_sects` (line 1714) plus `SIMBOARD:create_squad(self, ...)` (line 1716) does the actual spawn.

## ZCP integration

ZCP's `smr_pop.smart_can_respawn` (`smr_pop.script:342-368`) replaces the cooldown gate. It returns `true` immediately when `last_respawn_update == nil` (`:352-354`). Our nudge works identically under ZCP.

ZCP's `smr_handle_spawn` (`smr_pop.script:288-339`) substitutes the squad section in flight. The replacement squad still gets `respawn_point_id` and `respawn_point_prop_section` set (`smart_terrain.script:1726-1727` in ZCP fork), so the engine's own budget accounting stays intact. When that substituted squad eventually dies, its deaths still count against our origin smart's threshold.

## Population invariant

For any single origin smart, between any two consecutive nudges of that smart, the engine cannot refill more NPCs than have died since the last nudge.

Proof: a nudge fires only when `_deaths[smart_id] >= MAX(npc_in_squad upper bound)`. A nudge causes at most one `try_respawn`, which creates at most one squad with at most MAX(npc_in_squad upper bound) NPCs. After the nudge, `_deaths[smart_id] = 0`. The next nudge requires another full threshold of deaths to accumulate.

Refill <= death, in absolute terms, for every smart.

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, UI definition, button handlers |
| `gamedata/scripts/ab_pacing.script` | Death handler, periodic tick, threshold cache, nudge |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner (512x50) |

## MCM Settings

| Setting | Default | Effect |
|---------|---------|--------|
| `enabled` | true | Master toggle |
| `log_level` | WARN | Logger verbosity (ERROR/WARN/INFO/DEBUG) |

No threshold knob. Threshold is per-smart, derived from `ini_sys:r_string_ex(squad_section, "npc_in_squad")`.

## Performance

| Operation | Cost |
|-----------|------|
| Per death | O(1): nil-guard, field read (`squad.respawn_point_id`), counter increment |
| Threshold compute (first death per smart) | O(recipes * squads_per_recipe), 1 luabind per squad (`ini_sys:r_string_ex` medium). Cached. |
| Threshold lookup | O(1) cache hit |
| Per tick (60s) | O(N) over `_deaths`, N = smarts with recent deaths |
| Per nudge | O(1): one field write |
| Persistence | None. State resets on game load. |

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla try_respawn | Reads `last_respawn_update` at `smart_terrain.script:1651`. nil short-circuits. |
| ZCP (forked try_respawn) | Reads the same field at `smr_pop.script:352-354`. nil short-circuits. |
| Redone Collection | Pure LTX configuration. AlifeBalance writes runtime state. No conflict. |
| GAMMA NPC Spawns | Pure LTX. No conflict. |
| Night Mutants | Parallel `SIMBOARD:create_squad` path. Spawned squads still get `respawn_point_id` set. No conflict. |
| Nocturnal Mutants | Raw `alife_create`. Bypasses smart terrains entirely. Those squads have no `respawn_point_id`, ignored. |
| GAMMA Dynamic Despawner | Enforces online cap. Does not touch respawn fields. Despawned squads do not fire `squad_on_npc_death`, so they don't trigger nudges. Independent. |
| AlifeGuard | Squad-aware despawner. Same: despawn != death, no nudge. |
| AlifePlus | Reactive A-Life framework. Different fields and patterns. No overlap. |
| Warfare | Manages its own squads. Warfare squads typically have no `respawn_point_id`, ignored. |

No base script edits. No engine patches. Runtime callbacks only.
