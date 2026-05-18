AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
- Version: 1.0.0 (xlibs 1.5.1)
- Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
- Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
- Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Please use the RESET button in MCM when updating to a new version !

Smart terrains run respawn cooldowns of 12 to 24 game-hours, and in heavy combat squads die faster than those cooldowns can refill them. Active zones drain across a play session, and whole regions hollow out before the engine's own pace catches up.

AlifeBalance counts NPC deaths per (level, faction) and runs a 60-second scanner over those counters. When one squad's worth of kills has accumulated for a (level, faction) pair, AlifeBalance picks one open-budget smart on that level whose recipes can produce that faction and subtracts a fraction of that smart's respawn cooldown. After the configured number of waves on the same smart, the cooldown has been moved back by a full respawn period, and the engine spawns one squad from the smart's own configured pool on its next alife tick.

The wave count is a player setting in MCM with range 1 to 8 and default 4. At 1, a single wave clears the full cooldown for an instant refill. At 8, the refill is paced across eight waves of sustained combat. The per-wave advance is floored at one game hour, so short cooldowns and high wave counts still produce a meaningful nudge.

AlifeBalance does not spawn anything. Each wave is one write to one field on one smart, where the field is last_respawn_update, the same CTime field the engine itself writes after every successful spawn. The arithmetic uses xrTime:sub, an engine API, and the new value rides save data through the engine's own w_CTime / r_CTime path.

The engine still owns every gate: peace info, level filter, actor distance, simulation availability, per-recipe budget. The engine still picks the recipe, the squad section, and the NPC count within the configured range. AlifeBalance only moves the cooldown closer to expiry, and the engine decides whether and what to spawn when that cooldown finally expires.

There are no base script edits, no engine patches, no parallel scheduler, and no entity creation calls of any kind. Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, and Redone.

Example:

A bandit firefight thins bandit presence on Cordon, and the death counter for (cordon, bandit) climbs. Once it crosses the per-(level, faction) threshold (around 3 deaths on Cordon, derived from the bandit squad sections' npc_in_squad upper bound), AlifeBalance picks one open-budget bandit-spawning smart on Cordon and subtracts a fraction of that smart's cooldown. The counter resets and the next wave starts fresh. After the configured number of waves on the same smart, the cooldown has been advanced by the full respawn_idle, and the engine's try_respawn at that smart spawns one bandit squad from the smart's own pool on its next alife tick.

If every bandit-spawning smart on Cordon is at budget cap when a wave would fire, the wave defers and the counter holds until the next tick, at which point AlifeBalance retries with whatever budget has opened in the meantime.

Structural invariants:

- Event-driven: AlifeBalance subscribes to engine callbacks and runs only when the engine fires one, and is idle between waves.
- Engine-native: every wave is one CTime subtraction on the same field the engine itself writes after every successful spawn, with no SIMBOARD calls and no gate bypass.
- No fabrication: AlifeBalance never creates entities of any kind, and the engine remains the sole spawner.
- Refill <= death: per (level, faction), between consecutive engine spawns at any eligible smart for that pair, the engine refills no more NPCs of that faction on that level than have died.
- Configuration-respect: vanilla, ZCP, Redone, and modpack-configured spawn pools, cooldowns, props, and max populations remain authoritative, and AlifeBalance only moves the cooldown timestamp on smart terrains that already produce the dying faction.

What gets counted:

AlifeBalance tracks deaths for twelve stalker factions (stalker, bandit, csky, dolg, freedom, ecolog, killer, monolith, renegade, army, greh, isg), plus zombied, plus seven mutant faction keys (monster, monster_predatory_day, monster_predatory_night, monster_vegetarian, monster_zombied_day, monster_zombied_night, monster_special), each with its own counter per level. Protected squads (story, companions, task targets) and cowardly-tier mutants (rats, tushkanos, flesh, zombies, karliks) are filtered out before counting.

How the threshold is derived:

The threshold for a (level, faction) pair is the MAX npc_in_squad upper bound across the squad sections in eligible smarts' recipes that produce the dying faction. It is read from squad_descr LTX on first death for the pair and cached for the session. A level with rat-lair smarts producing the faction sits around 15 to 20, and a stalker-only level sits around 3 to 5.

How eligibility is decided:

A smart is eligible to refill faction F when at least one section across all its respawn_params recipes has its squad_descr faction field equal to F. This is the same vocabulary as squad.player_id at spawn time and xcreature.community at runtime. Smart-terrain props are not used, since they control where a squad can move to and not what a smart's recipes spawn. After eligibility, AlifeBalance filters smarts down to those with open spawn budget right now under the engine's own already_spawned[k].num < max gate, and if no eligible smart has open budget the wave defers and pressure accumulates until a slot frees naturally when one of the engine's tracked squads dies.

How the wave gets applied:

The death event handler only increments counters and is O(1) per death. The 60-second scanner walks the counters, picks every (level, faction) at or above threshold, applies the budget filter, advances the cooldown of one open-budget eligible smart by max(respawn_idle / waves, one game hour), and resets only that pair's counter. The picker is random across open-budget eligibles, and the engine resets the picked smart's last_respawn_update to curr_time once it finally spawns, which naturally rotates the picker onto other smarts on the next cycle.

What the engine still owns:

The engine picks the recipe at random from the smart's open ones, picks one squad section at random from that recipe's pool, calls SIMBOARD:create_squad, sets respawn_point_id, increments already_spawned[k].num, and sets last_respawn_update to curr_time.

Sources of variance between a death and a spawn:

- Per-(level, faction) threshold derived from squad_descr LTX
- Wave count per refill cycle (configurable 1 to 8)
- 60-second tick alignment
- Random smart pick among open-budget eligibles per wave
- Engine random pick among open recipes on the chosen smart
- Engine random pick from the recipe's squad pool
- Engine random NPC count within the squad section's npc_in_squad range
- ZCP substitution if enabled

Architecture:

AlifeBalance is reactive plus periodic with no global scheduler. The death callback increments a counter keyed by (level, faction), and the 60-second real-time tick reads those counters, applies waves, and resets fired counters. Runtime is xlibs, a reverse-engineered API wrapping the X-Ray engine source.

Performance:

- O(1) per death
- O(L * F) per 60-second tick over death pairs
- O(S * R * Q) on first death for a (level, faction) pair (S smarts on level, R recipes per smart, Q sections per recipe), then O(1) on every cache hit
- 3 luabind per wave (CTime read, sub, write back)
- No persistence for AlifeBalance state, counters reset on game load

Compatibility:

- Vanilla and ZCP read the same field AlifeBalance writes, and the advanced cooldown applies under both gates.
- Redone Collection and GAMMA NPC Spawns ship pure LTX configuration, while AlifeBalance writes runtime state on a different layer.
- Night Mutants uses a parallel SIMBOARD:create_squad path, and the spawned squads still get respawn_point_id set.
- Nocturnal Mutants spawns entities directly outside smart terrains and is ignored.
- GAMMA Dynamic Despawner enforces an online cap on the despawning side, and despawn does not fire squad_on_npc_death, so it never triggers a false wave.
- AlifeGuard, AlifePlus, Warfare, and Guards Spawner have no overlap with AlifeBalance's intervention surface.

MCM:

- Smart Pacing: enable toggle, waves to refill (1 to 8, default 4), minimum hours per wave (1 to 6, default 1).
- Development: log level, show map markers, show status button, reset counters button.
- Map markers are green PDA spots with a 5-minute linger flagging every advanced smart, and right-clicking a marker teleports to that smart or shows its tracked wave stats.

Requirements:

- Anomaly 1.5.3
- xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001), version 1.5.1 or later
- MCM

Install (MO2):

1. Install xlibs
2. Install AlifeBalance
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):

Disable or remove in MO2.

Credits:

Altogolik for support, ideas, and source materials.

Usage and License:

- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in LICENSE file and on GitHub.
