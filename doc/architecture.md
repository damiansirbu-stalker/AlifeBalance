# AlifeBalance Architecture

AlifeBalance accelerates respawn at smart terrains whose recipes can produce a faction the player has been killing. It does not spawn anything. Per (level, faction), it counts deaths. Every 60 real-time seconds a scanner walks the counters. For each pair where `count >= threshold` and at least one eligible smart can still accept an advance, AlifeBalance subtracts `respawn_idle / advances` game-seconds from that smart's `last_respawn_update` field, then subtracts `threshold` from the counter and repeats while pressure remains. The same constant amount per advance per smart. 100 kills with threshold 3 produces up to 33 advances in one tick, distributed across eligible smarts until each one hits its `min_hours` floor. AlifeBalance stops pushing once a smart has less than `min_hours` of cooldown remaining; from there the engine's own clock ages out the last leg and fires `try_respawn` on its next alife tick. The engine still owns the spawn, the gates, and the squad selection. AlifeBalance never triggers the spawn directly.

`advances` is a player setting in MCM, range 1 to 8, default 4. Higher values pace refills across more sustained combat. A second MCM setting, `min_hours` (default 1), sets the floor on the remaining cooldown that AlifeBalance will never push below.

Built on xlibs. `_ab_deps` asserts `xlibs >= 1.5.1` on load.

![Layered position](img/layers.png)

---

## The cooldown clock

**The one engine field AlifeBalance writes is `last_respawn_update`. Nothing else.**

Every smart terrain holds two engine fields that together define when it can spawn next:

- `respawn_idle` — cooldown DURATION in game-seconds. Set once from LTX (`smart_terrain.script:231`), per-smart. Vanilla default 43200 (12 game hours), ZCP under GAMMA is 21600 (6 game hours), Redone varies. **AlifeBalance only reads this field. Never writes.**
- `last_respawn_update` — TIMESTAMP of the smart's last spawn. Reset to `curr_time` by the engine after every `SIMBOARD:create_squad` at `smart_terrain.script:1719`. **This is the single field AlifeBalance writes — one subtract per advance, in `_advance_smart` at `ab_pacing.script:269-277`.**

The engine's cooldown gate at `smart_terrain.script:1651`:

```
elapsed         = curr_time - last_respawn_update
gate_open       = elapsed > respawn_idle
hours_remaining = respawn_idle - elapsed       (the engine debug overlay reports this)
```

So `hours_remaining` is derived, not stored. It is the headline number a player would see in the debug overlay or in the marker hint, and it is what AlifeBalance moves indirectly.

AlifeBalance accelerates the gate by aging `last_respawn_update` backward: one advance does

```
smart.last_respawn_update :sub( (respawn_idle - min_hours * 3600) / advances seconds )
```

That makes `elapsed` artificially bigger, which makes `hours_remaining` artificially smaller, which brings the gate closer. The cooldown DURATION (`respawn_idle`) never changes — only the timestamp does. The engine's own `respawn_idle` setting remains the ground truth for cycle length.

Worked example at vanilla 12h cooldown, MCM `advances=4`, `min_hours=1`:

```
respawn_idle         = 43200 s   = 12.0 game-hours
min_hours*3600       = 3600 s    = 1.0 game-hours
usable               = 39600 s   = 11.0 game-hours      (the part we may advance)
per-advance subtract    = 9900 s    = 2.75 game-hours      (usable / advances)
advances to floor       = 4                                (usable / per-advance)
floor remaining      = 3600 s    = 1.0 game-hours       (engine ages this naturally)
```

Same math at ZCP `respawn_idle=21600`, MCM `advances=4`, `min_hours=1`:

```
usable               = 18000 s   = 5.0 game-hours
per-advance subtract    = 4500 s    = 1.25 game-hours
advances to floor       = 4
floor remaining      = 3600 s    = 1.0 game-hours
```

Advances-to-floor stays equal to `advances` (by construction of `advance = usable / advances`). Per-advance subtract scales with `respawn_idle`, so on shorter-cooldown modpacks each advance is a smaller game-time move while the kill cost stays the same.

---

## Vocabulary

- Smart terrain: an engine-level spawner with a configured pool of squad sections, a per-recipe budget cap, and a cooldown timestamp.
- `respawn_idle`: per-smart cooldown duration in game-seconds, set in LTX (default 43200 = 12 game hours, vanilla range 12 to 24 hours). The engine cooldown gate is `curr_time:diffSec(last_respawn_update) > respawn_idle`.
- Recipe: one entry in a smart's `respawn_params`. Holds a list of squad sections (`squads`) and a numeric condlist (`num`) for the per-recipe budget cap.
- Faction: the value of `squad.player_id` at death. Matches the `faction` field in `squad_descr/*.ltx` for the source section.
- Advance: one trigger event in the 60s scanner. Subtracts `respawn_idle / advances` from one picked smart's `last_respawn_update`. Constant per smart.
- Threshold: kills required to fire one advance for a (level, faction) pair. Set to the largest `npc_in_squad` upper bound across all squad sections in eligible smarts' recipes that produce the faction. Examples: Cordon stalker = 3 (one stalker squad maxes at 3 NPCs); Swamp boar = 15 (one boar squad maxes at 15). Read from `squad_descr` LTX on first death for the pair, cached for the session. No MCM knob; the value is engine-grounded by construction.
- Eligible smart: a smart on the target level with at least one section in its `respawn_params` recipes whose `squad_descr faction` equals the target faction.
- Open budget: at least one matching recipe has `max > already_spawned[k].num` right now, where `max = pick_section_from_condlist(actor, smart, recipe.num)`.
- Can-advance: the smart's current cooldown age plus one full per-advance subtract still leaves at least `min_hours` of cooldown remaining (`age <= respawn_idle - min_hours * 3600 - respawn_idle / advances`). A smart that fails this check is skipped by the picker.
- Picker order: `with_budget` is sorted descending by actor-to-smart distance once per tick. Advances pile onto the farthest qualifying smart first, then walk back toward the actor as smarts hit their floor. Soft preference, not a hard filter — if only near-actor smarts are eligible, they still receive advances (the engine's own `respawn_radius` gate at `smart_terrain.script:1619` defers the spawn until the actor moves out).

---

## Invariants

- **`last_respawn_update` is the only engine field AlifeBalance writes. One subtract per advance. Nothing else.**
- AlifeBalance never calls `SIMBOARD:create_squad`. The engine spawns.
- AlifeBalance does not modify `respawn_params`, `already_spawned`, `props`, `max_population`, `faction`, `respawn_idle`, or any other smart configuration.
- No base script edits. No engine patches. Runtime callbacks only.
- AlifeBalance never leaves a smart with less than `min_hours` of cooldown remaining. The picker filters out smarts whose age would exceed the cap after another advance, and `_advance_smart` always subtracts the same per-smart amount with no partial steps.
- Refill <= death per (level, faction), between consecutive engine spawns at any eligible smart for that pair. See "Population invariant" for the proof.

---

## Pipeline

![Pipeline](img/pipeline.png)

```
DEATH (engine fires squad_on_npc_death)
  |
  v
_on_npc_death(squad, npc, killer)
  - if disabled: return
  - _stats.deaths += 1
  - if xsquad.is_protected:        _stats.protected += 1, return
  - if _is_vermin_squad (rat / tushkano): _stats.vermin += 1, return
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
        with_budget = filter smarts by _can_advance AND _evaluate_budget_for_faction
        if #with_budget == 0:                       [DEFER],  counter holds
        else:
          sort with_budget by actor->smart distance descending
          while count >= threshold and #with_budget > 0:
            smart = with_budget[1]                  (farthest from actor still eligible)
            params = _get_advance_params(smart)     lazy: delta = respawn_idle/advances CTime; max_age = cap
            smart.last_respawn_update:sub(params.delta)
            count -= threshold
            _stats.advances += 1                    [ADVANCE]
            if not _can_advance(smart):
              remove smart from with_budget         (next-farthest becomes [1])
          by_faction[faction] = count               (leftover < threshold carries to next tick)
          if burst_advances > 1: log [BURST]

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
| Cooldown timer | 1651 | Subtract `respawn_idle / advances` from `last_respawn_update` per advance |
| Per-recipe budget | 1696 | Untouched |

---

## The advance math

Per smart, lazy-cached in `_delta_cache[smart_id]`:

```
advance_seconds = respawn_idle / advances
max_age         = respawn_idle - min_hours * 3600 - advance_seconds
delta           = CTime of advance_seconds (built once, reused every advance)
```

`advance_seconds` is constant per smart. Every advance subtracts the same amount from `last_respawn_update`.

`max_age` is the highest age at which one more advance still leaves at least `min_hours` of cooldown remaining. The picker calls `_can_advance(smart)` which returns true when `age <= max_age`, and any smart that fails is dropped from the with-budget set as the loop walks farthest-to-near. AlifeBalance never crosses the floor; the engine's own clock ages out the last `min_hours` and fires `try_respawn` from there.

The CTime delta is built via two CTime instances. One is zeroed via `setHMSms(0, 0, 0, 0)`, the other has the H/M/S of `advance_seconds` via `setHMSms(h, m, s, 0)`. The (year=1, month=1, day=1) base of the engine's `generate_time(...)` cancels under subtraction; the result is a pure duration usable with `xrTime:sub`.

Per advance:

```
lru = smart.last_respawn_update or game.get_game_time()
lru:sub(delta)              -- constant subtract, never crosses the cap
smart.last_respawn_update = lru
```

Three luabind calls per advance. The delta cache prevents repeat CTime construction. A change to `advances` or `min_hours` in MCM invalidates the cache.

Total advances that can apply on one smart before the engine fires `respawn_idle / advance_seconds - 1` at most (the last advance that would still leave `min_hours` of remaining cooldown). After that point the smart sits at the floor; the engine ages the cooldown the rest of the way naturally and fires `try_respawn`, which resets `last_respawn_update = curr_time` and clears the smart for advances again.

---

## Population invariant

For any (level, faction) pair, between consecutive engine spawns at any eligible smart for that pair, the engine refills no more NPCs of that faction than have died on that level.

Proof. A spawn at an eligible smart fires when the smart's cooldown has aged past `respawn_idle`. AlifeBalance contributes to that age in equal increments of `respawn_idle / advances` per advance but stops once the remaining cooldown would drop below `min_hours`. The engine ages the remaining `min_hours` naturally and fires. Each advance fires after the (level, faction) counter reaches `threshold = MAX(npc_in_squad upper)` across matching sections, and the picker rejects smarts already at the floor. Total deaths between consecutive spawns at the same smart `>= floor(applied_advances) * threshold`, where `applied_advances` is the number of advances the smart accepted before hitting the floor. The spawn creates at most one squad. If the engine picks a matching section (`squad_descr faction == target faction`), the squad has at most `MAX npc_in_squad` NPCs of the target faction. If a non-matching section in a mixed-faction pool, zero NPCs of the target faction spawn. Either way: `deaths_since_last_spawn >= MAX >= refill`.

The vanilla cooldown ages in parallel from real time. If the player ignores a level for `advances * tick_interval` game-time, the cooldown expires on its own and the engine spawns at vanilla pace. Pacing accelerates the cooldown. It never delays vanilla.

---

## Why advance, not clear

Setting `last_respawn_update = nil` bypasses the cooldown gate entirely. The engine then fires on its next alife tick. The signal "combat happened" becomes "spawn now," with no middle ground. Combat intensity stops mattering once threshold is crossed once.

Per-advance advance keeps the gate engaged. Each advance is a small constant push, and sustained combat produces sustained refill. Burst combat that crosses threshold once produces `1 / advances` of a cycle, which the vanilla cooldown then ages out at vanilla pace if combat stops. The pacing matches death rate without overriding the engine's own gate.

The `min_hours` floor on remaining cooldown guarantees the engine always fires the last leg on its own. AlifeBalance pushes the cooldown closer but never to expiry; from the floor the engine's natural ageing closes the gap and `try_respawn` runs from the engine's clock, not ours. The picker drops smarts whose age would cross the floor on the next advance, so the floor is a hard guarantee, not a soft target.

---

## What we own

- `_deaths[level_id][faction]`: death counter per (level, faction). Reset on a fired advance. Held on defer.
- `_eligible[level_id][faction]`: cached array of smart ids whose recipes produce the faction. Recipe-content derived, static for the session.
- `_thresholds[level_id][faction]`: cached MAX `npc_in_squad` upper bound across matching sections.
- `_delta_cache[smart_id]`: cached `{ delta, advance, max_age }` per smart. `delta` is the CTime of `respawn_idle / advances` seconds, `advance` is the same value as a scalar (seconds), `max_age` is `respawn_idle - min_hours * 3600 - advance`. Invalidated on `advances` or `min_hours` change in MCM and on `_reset_state`.
- `_smart_stats[smart_id]`: per-smart advance bookkeeping for the right-click "Show stats" tip.
- `_seen_squads[squad.id]`: dedup set for spawn tracing (DEBUG only).
- `_markers[smart_id]`: linger expiry per marked smart.
- `_stats`: 11 diagnostic counters (`deaths, counted, protected, vermin, no_squad, no_faction, no_level, ticks, advances, spawns, spawns_from_us`).
- Callbacks: `squad_on_npc_death`, `squad_on_npc_creation`, `on_option_change`, `load_state`, `actor_on_first_update`, `map_spot_menu_add_property`, `map_spot_menu_property_clicked`.
- One `CreateTimeEvent` timer at 60-second wall interval.

No persistence for AlifeBalance state. Counters and caches reset on game load. Advanced `last_respawn_update` values survive in the engine's own save data via `utils_data.w_CTime / r_CTime`.

---

## What we do not own

Which recipe the engine picks (random over open-budget recipes). Which squad section within that recipe (random). NPC count within `npc_in_squad` range (random). ZCP `smr_handle_spawn` substitution. Online cap enforcement (GAMMA Dynamic Despawner, AlifeGuard).

---

## ZCP integration

ZCP's `smr_pop.smart_can_respawn` (`smr_pop.script:342-368`) reads the same `last_respawn_update` field but applies its own configured `respawn_idle` (default 86400 = 24h game-time). Our per-advance subtract is sized at `smart.respawn_idle / advances` (vanilla 12h / 4 = 3h), so under default ZCP the advance covers only ~half the gate. The min_hours floor still holds. The remainder ages naturally.

ZCP's `smr_handle_spawn` substitutes the squad section in flight. The replacement still gets `respawn_point_id` and `respawn_point_prop_section` set, so engine budget accounting stays intact. The substituted section may have a different `faction` field than the original; ZCP's design assumes intentional substitution by the modpack author, and AlifeBalance has no opinion on the outcome.

ZCP does not ship its own `squad_descr` overrides for vanilla sections; `xsmart.section_faction` reads the same values under vanilla and ZCP.

---

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, UI definition, button handlers |
| `gamedata/scripts/ab_pacing.script` | Death handler, periodic tick, eligibility / threshold cache, advance advance, public `marker_label` + `show_smart_stats` for ab_map |
| `gamedata/scripts/ab_map.script` | PDA marker render-state, right-click menu (teleport, show stats). Calls back into ab_pacing for label + stats formatting. |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner (512x50) |

---

## MCM

| Setting | Tab | Type | Default | Range | Effect |
|---------|-----|------|---------|-------|--------|
| `enabled` | General | check | true | - | Master toggle |
| `advances` | General | track | 4 | 1-8 | Per-advance subtract is `respawn_idle / advances` |
| `min_hours` | General | track | 1 | 1-6 | Minimum cooldown remaining after every advance, in game hours |
| `show_markers` | Development | check | false | - | Green PDA marker on every advanced smart, 5-min linger, right-click teleport / stats |
| `log_level` | Development | list | WARN | - | ERROR / WARN / INFO / DEBUG |

Threshold has no knob. It is engine-grounded, read from `squad_descr` LTX per (level, faction) and cached.

---

## Performance

| Operation | Cost |
|-----------|------|
| Per death | O(1). Protection check, vermin species lookup (cached), faction read, level lookup, counter increment. |
| `_get_eligible_and_threshold`, first call per pair | O(S * R * Q) over smarts on level * recipes per smart * sections per recipe. 2 luabind `ini_sys:r_string_ex` per section (faction + npc_in_squad, both medium). Cached. |
| `_get_eligible_and_threshold`, cache hit | O(1) |
| `_evaluate_budget_for_faction`, per tick per eligible smart | O(R * Q) over recipes * sections. 1 luabind `pick_section_from_condlist` per recipe + 1 `ini_sys:r_string_ex` per section until first faction match. |
| `_get_advance_params`, first call per smart | 2 CTime constructions + 2 `setHMSms` + 1 operator-. About 5 luabind. Cached. |
| `_get_advance_params`, cache hit | O(1) |
| `_can_advance`, per tick per eligible smart | 1 luabind: `xtime.game_time():diffSec(lru)`. |
| Per advance | 3 luabind: `last_respawn_update` read, `:sub` with cached delta, write back. |
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
| GAMMA Dynamic Despawner | Enforces online cap on the despawning side. Despawn does not fire `squad_on_npc_death`. No false advances. |
| AlifeGuard | Squad-aware despawner. Same: despawn is not death. |
| AlifePlus | Reactive A-Life framework. Different fields and patterns. No overlap. |
| Warfare | `faction_controlled` smarts (16 of about 490) have an injected `respawn_params` entry whose `.squads` field is the vanilla `squads_by_faction[faction]` list. `xsmart.section_faction` reads those vanilla sections normally. Works. |
