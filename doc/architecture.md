# AlifeBalance Architecture

## TODO: Diagrams

Two diagrams to add to `doc/img/`:

1. **Internal flow.** Producer/consumer split: death callback fills `_deaths` table, periodic 60-second tick reads it, picks one faction over threshold, filters SIMBOARD, weighted-picks one smart, backdates its cooldown timestamp. Engine's own try_respawn fires later and refills from the smart's own LTX pool.

2. **Layered position.** X-Ray engine at the base. AlifeBalance one layer above (reads/writes smart-terrain server fields, no spawn calls). Spawner mods (vanilla LTX, ZCP, Redone, GAMMA NPC Spawns) above that or beside, configuring WHAT spawns. AlifeBalance does not sit above them; it sits beside them on the same vanilla state layer. Show that AB and ZCP/Redone are peers, both consuming the engine's state, both written by vanilla's configuration.

## The big idea

We do not spawn. The engine already spawns. When combat thins a faction's presence on a level, we nudge a quiet smart on that level to refill sooner. The nudge is a single write to one server-entity field. The engine does the rest on its own schedule using its own configured spawn pools.

## The signal

Per level, one death counter per faction. Twelve stalker factions (stalker, bandit, csky, dolg, freedom, ecolog, killer, monolith, renegade, army, greh, isg) plus mutant prop keys (monster_predatory_day, monster_predatory_night, monster_vegetarian, monster_zombied_day, monster_zombied_night, monster_special). Cowardly-tier mutants (rats, tushkanos, flesh, zombies, karliks) do not count. Protected squads (story, companions, task targets) do not count.

## The trigger

A real-time 60-second tick scans every counter. One faction with a count at or above the threshold wins (random among ties). The intervention runs once. That faction's counter resets to zero.

The death handler does not trigger interventions. It only increments counters. Producer and consumer are decoupled. Bursts of deaths cannot produce bursts of interventions because the consumer runs on its own clock.

## The pick

Among smarts on the dying faction's level, three conditions must hold:

- Has `respawn_params` (is a respawn point at all).
- Accepts the faction (engine's own props check via `xsmart.has_faction` or `xsmart.accepts_mutant`).
- Has at least one matching-kind recipe with currently zero alive squads. The smart already lost something there; the slot is open.

If no smart satisfies all three, the intervention drops. The counter stays where it was, ready for next tick.

If multiple smarts qualify, weighted random pick by `1 / (1 + npc_info count)`. Emptier smarts win more often.

## The nudge

One write on the chosen smart's server entity. Overwrite `last_respawn_update` with `current_game_time - 24h`.

That is the entire intervention. No budget decrement. No injection into `respawn_params`. No timestamp set to nil. 24h exceeds vanilla `respawn_idle` (12h) and ZCP (6h), so the cooldown gate passes on the next engine alife tick regardless of which modpack configures the cooldown.

## What the engine does

On its next alife tick, the smart's `update()` runs. Inside, `try_respawn` reads `last_respawn_update`. Our backdate makes the engine see "more time has passed since the last respawn than really has." The cooldown gate (`diffSec > respawn_idle`) passes earlier than it otherwise would.

After the cooldown gate, the engine walks its recipes, finds the ones with open budget (which we know exists because we filtered for it), picks one at random, picks a squad section at random from that recipe's pool, calls `SIMBOARD:create_squad`. The squad spawns. The engine increments `already_spawned[k].num`. The cooldown timestamp is set to "now."

We never touch budget. The engine's own death tracking owns it.

## What the player observes

A firefight thins duty squads on Garbage. Within a game-hour or two, one duty squad shows up at a quiet duty-friendly smart on Garbage. The smart was already low because combat killed its previous squads; the cooldown timer was the only thing holding refill back. We removed the hold.

Different smarts each time, different squad sections each time, different game-hours-of-delay each time. Nothing scripted, nothing predictable per single death.

## What we own

Two integer tables, a few timer values, one callback subscription.

- `_deaths[level_id][faction_key]`: 14 levels x ~18 factions worst case, sparse in practice.
- `_stats`: 8 diagnostic counters.
- `_last_tick`, `_last_summary`: real-time wall clock values.
- Callbacks: `squad_on_npc_death`, `on_option_change`, `load_state`, `actor_on_update`.

No persistence. Death counters reset on game load. In-flight cooldown writes survive because the engine saves smart fields in its own state.

## What we do not own

Squad type that spawns (smart's LTX pool). Squad count within a section (engine random in section's `npc_in_squad` range). Spawn timing within the cooldown window (engine alife tick cadence). Substitution (ZCP `smr_handle_spawn` may replace squad sections). Online cap enforcement (Dynamic Despawner's territory).

## Why it stays compatible with anything

The mod writes one vanilla engine field: `smart.last_respawn_update`. Every spawner mod in GAMMA either reads this field through the same code path the engine itself uses (vanilla, ZCP), or does not interact with smart-terrain respawn at all (Night Mutants's parallel path, Nocturnal Mutants's raw alife_create, despawn mods).

The only field we write is one the engine itself writes at every successful respawn. We are not introducing a new interface. We are operating on the engine's own state in the engine's own pattern.

## Producer / consumer pipeline

```
DEATH (engine fires squad_on_npc_death)
  |
  v
_on_npc_death(squad, npc, killer)
  - filter: protected? skip
  - filter: cowardly species? skip (rat/tushkano/flesh/zombie/karlik)
  - faction_key = squad.player_id
  - level_id = xlevel.get_level_id(npc or squad)
  - _deaths[level_id][faction_key] += 1
  - END (no further action)


TICK (every 60 wall-seconds, via actor_on_update interval gate)
  |
  v
_periodic_tick()
  - candidates = factions where _deaths[L][F] >= threshold
  - if no candidates: return
  - pick = random(candidates)
  - _intervene(pick.level_id, pick.faction_key, is_mutant)
  - if ok: _deaths[pick.level_id][pick.faction_key] = 0
  - END


_intervene(level_id, faction_key, is_mutant)
  - eligible = SIMBOARD.smarts filter:
      xlevel.get_level_id(smart) == level_id
      xsmart.has_faction(smart, faction_key) [stalker] or
      xsmart.accepts_mutant(smart, faction_key) [mutant]
      _check_smart_budget(smart, is_mutant): any matching recipe with
          pick_section_from_condlist(actor, smart, v.num) > already_spawned[k].num
  - if empty: return false
  - smart = weighted random pick by 1 / (1 + npc_info count)
  - _backdate_cooldown(smart):
      backdated = xtime.game_time()
      backdated:sub(0, 0, 0, 0, 0, 86400, 0)
      smart.last_respawn_update = backdated
  - return true


ENGINE (on its own alife tick)
  - se_smart_terrain:update() -> try_respawn()
  - cooldown gate (smart_terrain.script:1651): diffSec(last_respawn_update) > respawn_idle
      our backdate makes this pass earlier
  - budget gate (smart_terrain.script:1696): max > already_spawned[k].num
      already passing because we filtered for it
  - picks recipe at random from eligibles
  - picks squad section at random from recipe.squads
  - SIMBOARD:create_squad(smart, squad_section)
  - increments already_spawned[k].num
  - sets last_respawn_update = curr_time
```

## Engine gates inside try_respawn

The engine's own respawn check at `smart_terrain.script:1597-1762` has eight gates. We affect one:

| Gate | Source line | Our effect |
|------|-------------|-----------|
| Disabled / on_try_respawn callback | 1603 | Untouched |
| Peace info | 1607 | Untouched |
| Level filter (`respawn_only_level`) | 1611 | Untouched |
| Actor distance (`respawn_radius`) | 1619 | Untouched |
| `respawn_params and already_spawned` exist | 1625 | Untouched (we only target smarts that pass) |
| Simulation availability | 1630 | Untouched |
| Cooldown timer | 1651 | Backdated by our nudge |
| Per-recipe budget | 1696 | Untouched (engine's own decrement on death owns this) |

After both touchable gates pass, the engine's own random pick from `available_sects` (line 1714) plus `SIMBOARD:create_squad(self, ...)` (line 1716) does the actual spawn. Counter restored at line 1759.

## ZCP integration

ZCP's `smr_pop.smart_can_respawn` (smr_pop.script:342-368) replaces the cooldown gate inside its forked smart_terrain.script. It reads `smart.last_respawn_update` the same way vanilla does and compares against ZCP's MCM `respawn_idle` value (default 6 game-hours) instead of vanilla's 12. Our backdate works under ZCP identically: the timestamp is what both vanilla and ZCP read.

ZCP's `smr_handle_spawn` (smr_pop.script:288-338) replaces the squad section in flight. Under GAMMA defaults, all substitution features are off and the squad section passes through. The squad still gets `respawn_point_id` and `respawn_point_prop_section` set (smart_terrain.script:1726-1727 in ZCP fork), so the engine's own budget accounting stays intact.

## Variance sources

| Source | Where |
|--------|-------|
| Counter accumulation timing | Deaths arrive irregularly |
| Threshold value | MCM knob (deaths_per_nudge) |
| Random pick among over-threshold factions | When multiple factions cross threshold in the same tick |
| Weighted smart pick | `1 / (1 + npc_info count)` |
| Engine's own recipe pick | Random across `available_sects` |
| Engine's own squad pool pick | Random across recipe's `squads[]` |
| Engine's own NPC count pick | Random across squad section's `npc_in_squad` range |
| ZCP substitution | If user enabled it |

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, UI definition, button handlers |
| `gamedata/scripts/ab_pacing.script` | Death handler, periodic tick, smart filter, backdate |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner (512x50) |

## MCM Settings

| Setting | Default | Range | Effect |
|---------|---------|-------|--------|
| `enabled` | true | bool | Master toggle |
| `deaths_per_nudge` | 5 | 5-20 | Deaths in one faction/level needed before one intervention |
| `log_level` | WARN | ERROR/WARN/INFO/DEBUG | Logger verbosity |

Backdate amount is hardcoded at 24 game-hours (`BACKDATE_SEC` in `ab_pacing.script`). 24 exceeds vanilla `respawn_idle` (12h) and ZCP (6h), so the cooldown gate passes on the next engine alife tick under any known modpack. No min/max randomization: variance lives in the picker (faction, smart) and the engine (recipe, squad section, NPC count).

## Performance

| Operation | Cost |
|-----------|------|
| Per death | O(1): protection check, cowardly check, counter increment |
| Per tick (60s) | O(L * F) candidate scan + O(S) smart filter + O(K) recipe scan per smart. L = levels, F = factions per level, S = smarts on level, K = recipes per smart. Sub-millisecond in practice. |
| Per intervention | O(S) eligibility scan, O(S_eligible) weighted pick, 3 luabind (xtime.game_time, CTime:sub, field write) |
| Persistence | None. Counters reset on game load. |

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla try_respawn | Reads `last_respawn_update` at smart_terrain.script:1651. Our backdate works. |
| ZCP (forked try_respawn) | Reads the same field at ZCP smart_terrain.script:1703. Our backdate works. |
| Redone Collection | Pure LTX configuration. AlifeBalance writes runtime state. No conflict. |
| GAMMA NPC Spawns | Pure LTX. No conflict. |
| Night Mutants | Parallel SIMBOARD:create_squad path. Does not touch `last_respawn_update`. No conflict. |
| Nocturnal Mutants | Raw `alife_create`. Bypasses smart terrains. No conflict. |
| GAMMA Dynamic Despawner | Enforces online cap. Does not touch respawn fields. Independent. |
| AlifeGuard | Squad-aware despawner. Different layer. No overlap. |
| AlifePlus | Reactive A-Life framework. Different fields and different patterns. No overlap. |

No base script edits. No engine patches. Runtime callbacks only.
