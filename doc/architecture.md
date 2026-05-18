# AlifeBalance Architecture

AlifeBalance accelerates respawn at smart terrains whose recipes can produce a faction the player has been killing. It does not spawn anything. Per (level, faction), it counts deaths. Every 60 real-time seconds a scanner walks the counters. When one crosses a threshold and at least one eligible smart on that level has open budget, AlifeBalance subtracts `respawn_idle / waves` game-seconds from that smart's `last_respawn_update` field (floored at one game hour). After the configured number of waves on the same smart, the cooldown has been advanced by the full `respawn_idle` and the engine spawns from the smart's own configured pool on its next alife tick. The engine still owns the spawn, the gates, and the squad selection.

`waves` is a player setting in MCM, range 1 to 8, default 4. At 1 wave, the first push clears the full cooldown for an instant refill. Higher values pace refills across more sustained combat. A second MCM setting, `min_hours`, sets the floor on the per-wave advance.

Built on xlibs. `_ab_deps` asserts `xlibs >= 1.5.1` on load.

---

## Vocabulary

- Smart terrain: an engine-level spawner with a configured pool of squad sections, a per-recipe budget cap, and a cooldown timestamp.
- `respawn_idle`: per-smart cooldown duration in game-seconds, set in LTX (default 43200 = 12 game hours, vanilla range 12 to 24 hours). The engine cooldown gate is `curr_time:diffSec(last_respawn_update) > respawn_idle`.
- Recipe: one entry in a smart's `respawn_params`. Holds a list of squad sections (`squads`) and a numeric condlist (`num`) for the per-recipe budget cap.
- Faction: the value of `squad.player_id` at death. Matches the `faction` field in `squad_descr/*.ltx` for the source section.
- Wave: one trigger event in the 60s scanner. Subtracts `respawn_idle / waves` from one picked smart's `last_respawn_update`.
- Threshold: per (level, faction), the MAX `npc_in_squad` upper bound across every section in any eligible smart's recipes that produces the faction. Engine-grounded, derived from `squad_descr` LTX, cached for the session.
- Eligible smart: a smart on the target level with at least one section in its `respawn_params` recipes whose `squad_descr faction` equals the target faction.
- Open budget: at least one matching recipe has `max > already_spawned[k].num` right now, where `max = pick_section_from_condlist(actor, smart, recipe.num)`.
- Cooldown advance: per-wave amount the cooldown moves back in time. `max(respawn_idle / waves, min_hours * 3600)` game-seconds.

---

## Invariants

- AlifeBalance never calls `SIMBOARD:create_squad`. The engine spawns. AlifeBalance only writes `last_respawn_update`.
- AlifeBalance does not modify `respawn_params`, `already_spawned`, `props`, `max_population`, `faction`, or any other smart configuration.
- No base script edits. No engine patches. Runtime callbacks only.
- Per-wave cooldown advance is at least `min_hours` game hours. `waves` and `respawn_idle` are clamped against this floor in `_get_advance_delta`.
- Refill <= death per (level, faction), between consecutive engine spawns at any eligible smart for that pair. See "Population invariant" for the proof.

---

## Pipeline

```
DEATH (engine fires squad_on_npc_death)
  |
  v
_on_npc_death(squad, npc, killer)
  - if disabled: return
  - _stats.deaths += 1
  - if xsquad.is_protected:        _stats.protected += 1, return
  - if xcreature.is_cowardly_species: _stats.trash    += 1, return
  - faction  = squad.player_id
  - level_id = xlevel.get_level_id(npc or squad)
  - _stats.counted += 1
  - _deaths[level_id][faction] += 1

TICK (every 60 wall-seconds via CreateTimeEvent)
  |
  v
_periodic_tick()
  - for each (level_id, faction) in _deaths:
      smarts, threshold = _get_eligible_and_threshold(level_id, faction)
      if #smarts == 0:                              [NOELIG], no action
      else if count < threshold:                    [BELOW],  no action
      else:
        with_budget = filter smarts by _evaluate_budget_for_faction
        if #with_budget == 0:                       [DEFER],  counter holds
        else:
          smart = random pick from with_budget
          delta = _get_advance_delta(smart)         lazy: max(respawn_idle/waves, min_hours h) CTime
          smart.last_respawn_update:sub(delta)      lru -= delta, clamps at 0
          counter resets to 0
          _stats.waves += 1                         [WAVE]

ENGINE (its own alife tick at the advanced smart)
  - se_smart_terrain:update -> try_respawn
  - cooldown gate (smart_terrain.script:1651): diff > respawn_idle ? proceed : skip
  - per-recipe budget gate (smart_terrain.script:1696)
  - picks recipe at random from open ones
  - picks squad section at random from recipe.squads
  - SIMBOARD:create_squad(smart, section)
  - sets squad.respawn_point_id = smart.id
  - increments already_spawned[k].num
  - writes last_respawn_update = curr_time
```

---

## Engine gates inside try_respawn

`smart_terrain.script:1597-1762` has eight gates. AlifeBalance affects one.

| Gate | Source line | Our effect |
|------|-------------|-----------|
| Disabled / on_try_respawn callback | 1603 | Untouched |
| Peace info | 1607 | Untouched |
| Level filter (`respawn_only_level`) | 1611 | Untouched |
| Actor distance (`respawn_radius`) | 1619 | Untouched |
| `respawn_params and already_spawned` exist | 1625 | Untouched |
| Simulation availability | 1630 | Untouched |
| Cooldown timer | 1651 | Subtract `respawn_idle / waves` from `last_respawn_update` per wave |
| Per-recipe budget | 1696 | Untouched |

---

## The wave math

Per smart, lazy-cached in `_delta_cache[smart_id]`:

```
advance_per_wave = max(respawn_idle / waves, min_hours * 3600)
delta            = CTime of advance_per_wave seconds
```

With the floor in place, the effective wave count never exceeds `respawn_idle / (min_hours * 3600)`. For a 12-hour smart at the default `min_hours = 1`, the effective cap is 12. For a 24-hour smart at the same floor, the effective cap is 24. For typical vanilla smarts and `waves` within 1 to 8, the floor never engages.

The CTime delta is built via two CTime instances. One is zeroed via `setHMSms(0, 0, 0, 0)`, the other has the H/M/S of `advance_per_wave` via `setHMSms(h, m, s, 0)`. The (year=1, month=1, day=1) base of the engine's `generate_time(...)` cancels under subtraction; the result is a pure duration usable with `xrTime:sub`.

Per wave:

```
lru = smart.last_respawn_update or game.get_game_time()
lru:sub(delta)              -- clamps at 0 on underflow (= fully expired)
smart.last_respawn_update = lru
```

Three luabind calls per wave. The delta cache prevents repeat CTime construction. A change to `waves` or `min_hours` in MCM invalidates the cache.

After the full wave count on the same smart, the cooldown has been advanced by `waves * advance_per_wave >= respawn_idle`. The engine's next alife tick at that smart finds `diff > respawn_idle` and runs `try_respawn`. The engine resets `last_respawn_update = curr_time`; the next wave starts from a fresh cooldown.

---

## Population invariant

For any (level, faction) pair, between consecutive engine spawns at any eligible smart for that pair, the engine refills no more NPCs of that faction than have died on that level.

Proof. A spawn at an eligible smart fires when the smart's cooldown has aged `respawn_idle` game-seconds. AlifeBalance contributes to that age in `waves` increments of `respawn_idle / waves` each (floored at `min_hours` hours). Each wave fires after the (level, faction) counter reaches `threshold = MAX(npc_in_squad upper)` across matching sections. Total deaths between consecutive spawns at the same smart `>= waves * threshold`. The spawn creates at most one squad. If the engine picks a matching section (`squad_descr faction == target faction`), the squad has at most `MAX npc_in_squad` NPCs of the target faction. If a non-matching section in a mixed-faction pool, zero NPCs of the target faction spawn. Either way: `deaths_since_last_spawn >= MAX >= refill`.

The vanilla cooldown ages in parallel from real time. If the player ignores a level for `waves * tick_interval` game-time, the cooldown expires on its own and the engine spawns at vanilla pace. Pacing accelerates the cooldown. It never delays vanilla.

---

## Why advance, not clear

Setting `last_respawn_update = nil` bypasses the cooldown gate entirely. The engine then fires on its next alife tick. The signal "combat happened" becomes "spawn now," with no middle ground. Combat intensity stops mattering once threshold is crossed once.

Per-wave advance keeps the gate engaged. Each wave is a small push, the full wave count is a full cycle, and sustained combat produces sustained refill. Burst combat that crosses threshold once produces `1 / waves` of a cycle, which the vanilla cooldown then ages out at vanilla pace if combat stops. The pacing matches death rate without overriding the engine's own gate.

The minimum advance per wave protects against meaningless nudges. A smart with `respawn_idle = 7200` (2 hours) at `waves = 8` would otherwise advance by 15 game-minutes per wave, which the engine alife tick would barely notice. Floored at one game hour, the effective wave count caps at `respawn_idle / 3600`.

---

## What we own

- `_deaths[level_id][faction]`: death counter per (level, faction). Reset on a fired wave. Held on defer.
- `_eligible[level_id][faction]`: cached array of smart ids whose recipes produce the faction. Recipe-content derived, static for the session.
- `_thresholds[level_id][faction]`: cached MAX `npc_in_squad` upper bound across matching sections.
- `_delta_cache[smart_id]`: cached CTime of `max(respawn_idle / waves, min_hours * 3600)` seconds per smart. Invalidated on `waves` or `min_hours` change in MCM and on `_reset_state`.
- `_smart_stats[smart_id]`: per-smart wave bookkeeping for the right-click "Show stats" tip.
- `_seen_squads[squad.id]`: dedup set for spawn tracing (DEBUG only).
- `_markers[smart_id]`: linger expiry per marked smart.
- `_stats`: 7 diagnostic counters (`deaths, counted, protected, trash, ticks, waves, spawns`).
- Callbacks: `squad_on_npc_death`, `squad_on_npc_creation`, `on_option_change`, `load_state`, `actor_on_first_update`, `map_spot_menu_add_property`, `map_spot_menu_property_clicked`.
- One `CreateTimeEvent` timer at 60-second wall interval.

No persistence for AlifeBalance state. Counters and caches reset on game load. Advanced `last_respawn_update` values survive in the engine's own save data via `utils_data.w_CTime / r_CTime`.

---

## What we do not own

Which recipe the engine picks (random over open-budget recipes). Which squad section within that recipe (random). NPC count within `npc_in_squad` range (random). ZCP `smr_handle_spawn` substitution. Online cap enforcement (GAMMA Dynamic Despawner, AlifeGuard).

---

## ZCP integration

ZCP's `smr_pop.smart_can_respawn` (`smr_pop.script:342-368`) replaces the cooldown gate but uses the same field. It returns true when `diff > respawn_idle` (or its own configured value). An advanced `last_respawn_update` advances the same diff under ZCP.

ZCP's `smr_handle_spawn` substitutes the squad section in flight. The replacement still gets `respawn_point_id` and `respawn_point_prop_section` set, so engine budget accounting stays intact. The substituted section may have a different `faction` field than the original; ZCP's design assumes intentional substitution by the modpack author, and AlifeBalance has no opinion on the outcome.

ZCP does not ship its own `squad_descr` overrides for vanilla sections; `xsmart.section_faction` reads the same values under vanilla and ZCP.

---

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, UI definition, button handlers |
| `gamedata/scripts/ab_pacing.script` | Death handler, periodic tick, threshold cache, wave advance, markers, teleport |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner (512x50) |

---

## MCM

| Setting | Tab | Type | Default | Range | Effect |
|---------|-----|------|---------|-------|--------|
| `enabled` | General | check | true | - | Master toggle |
| `waves` | General | track | 4 | 1-8 | Waves to fully advance one smart's cooldown |
| `min_hours` | General | track | 1 | 1-6 | Floor on the per-wave advance, in game hours |
| `show_markers` | Development | check | false | - | Green PDA marker on every advanced smart, 5-min linger, right-click teleport / stats |
| `log_level` | Development | list | WARN | - | ERROR / WARN / INFO / DEBUG |

Threshold has no knob. It is engine-grounded, read from `squad_descr` LTX per (level, faction) and cached.

---

## Performance

| Operation | Cost |
|-----------|------|
| Per death | O(1). Protection check, cowardly check, faction read, level lookup, counter increment. |
| `_get_eligible_and_threshold`, first call per pair | O(S * R * Q) over smarts on level * recipes per smart * sections per recipe. 2 luabind `ini_sys:r_string_ex` per section (faction + npc_in_squad, both medium). Cached. |
| `_get_eligible_and_threshold`, cache hit | O(1) |
| `_evaluate_budget_for_faction`, per tick per eligible smart | O(R * Q) over recipes * sections. 1 luabind `pick_section_from_condlist` per recipe + 1 `ini_sys:r_string_ex` per section until first faction match. |
| `_get_advance_delta`, first call per smart | 2 CTime constructions + 2 `setHMSms` + 1 operator-. About 5 luabind. Cached. |
| `_get_advance_delta`, cache hit | O(1) |
| Per wave | 3 luabind: `last_respawn_update` read, `:sub`, write back. |
| Per tick (60s) | O(L * F) over death pairs * O(E * R * Q) budget evaluation per pair, plus one-time computes per new pair or new smart. |
| Marker prune | O(M) markers per tick. |
| Persistence | None for AlifeBalance state. The engine persists `last_respawn_update` via its own save path. |

---

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla `try_respawn` | Reads `last_respawn_update` at `smart_terrain.script:1651`. The advance moves the same field. |
| ZCP (forked `try_respawn`) | Reads the same field at `smr_pop.script:352-368`. The advance applies identically. |
| Redone Collection | Pure LTX configuration. AlifeBalance writes runtime state. No conflict. |
| GAMMA NPC Spawns | Pure LTX. No conflict. |
| Night Mutants | Parallel `SIMBOARD:create_squad` path. Spawned squads still get `respawn_point_id` set. No conflict. |
| Nocturnal Mutants | Raw `alife_create`. Bypasses smart terrains entirely. Independent. |
| GAMMA Dynamic Despawner | Enforces online cap on the despawning side. Despawn does not fire `squad_on_npc_death`. No false waves. |
| AlifeGuard | Squad-aware despawner. Same: despawn is not death. |
| AlifePlus | Reactive A-Life framework. Different fields and patterns. No overlap. |
| Warfare | `faction_controlled` smarts (16 of about 490) have an injected `respawn_params` entry whose `.squads` field is the vanilla `squads_by_faction[faction]` list. `xsmart.section_faction` reads those vanilla sections normally. Works. |
