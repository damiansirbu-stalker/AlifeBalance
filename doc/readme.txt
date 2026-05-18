AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
- Version: 1.0.0 (xlibs 1.5.0)
- Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
- Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
- Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Please use the RESET button in MCM when updating to a new version !

Combat outpaces the Zone.

Smart terrains run a respawn cooldown of 12 to 96 game-hours depending on which smart. In long combat sessions on heavily modded setups, squads die faster than the cooldown can refill them. Active zones drain. Whole regions can hollow out across a play session.

AlifeBalance tracks NPC deaths per level and per faction. Once enough of one faction has died on one level, it clears the cooldown on a randomly picked smart on that level whose recipes can produce the dying faction and that has open spawn budget right now. The engine refills from that smart's own configured pool on its next alife tick.

No entity creation. No spawn injections. No engine gate bypass.
Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, and Redone. Compatible by design.

Example:

A bandit firefight thins bandit presence on the Cordon. The death counter for (cordon, bandit) climbs. Once it crosses the per-(level, faction) threshold, AlifeBalance scans the smarts on Cordon whose recipes spawn bandit squads (sections like bandit_sim_squad_novice / advanced / veteran). Among those, it filters to smarts with open spawn budget right now. From the filtered set, it picks one at random, sets its respawn cooldown to "now expired", and resets the counter. The engine's next try_respawn at that smart passes the cooldown gate and spawns one bandit squad from the smart's own pool. The area refills.

If every bandit-spawning smart on Cordon is at budget cap right now, the nudge is deferred and the counter holds. The next tick retries.

The nudge is one field write. The engine does everything else.

Structural invariants:

- Event-driven: subscribes to engine callbacks. Idle otherwise.
- Engine-native: every nudge is a single nil-write to a vanilla server-entity field the engine reads on its own update tick.
- No fabrication: no entity creation calls of any kind.
- No bypass: every engine gate still applies. Cooldowns, faction filters, smart-terrain props, modpack overrides, despawner caps.
- Refill never outpaces death: per (level, faction), between consecutive nudges, the engine refills no more NPCs of that faction than have died on that level.
- Configuration-respect: vanilla, ZCP, Redone, or modpack-configured spawn rules remain authoritative. AlifeBalance helps the engine honor them.

Scope:

AlifeBalance hosts balance features that operate on the engine's spawning side, using the engine's own gates and state. The first feature is Smart Pacing. Additional features will be added under the same contract.

Features:

Smart Pacing

  Faction-aware respawn cadence tied to combat death pressure on a level.

  Death tracking. Twelve stalker factions (stalker, bandit, csky, dolg, freedom, ecolog, killer, monolith, renegade, army, greh, isg), plus zombied, plus mutant faction keys (monster, monster_predatory_day, monster_predatory_night, monster_vegetarian, monster_zombied_day, monster_zombied_night, monster_special) each have their own counter per level. Protected squads (story, companions, task targets) do not count. Cowardly-tier mutants (rats, tushkanos, flesh, zombies, karliks) do not count.

  Per-(level, faction) threshold. The threshold is the maximum upper bound of npc_in_squad across the squad sections in eligible smarts' recipes that produce the dying faction. Derived from engine LTX data. A level with rat-lair smarts producing that faction has a threshold around 20; a stalker-only level around 4-5. No MCM knob.

  Smart eligibility is recipe-content based. A smart is eligible to refill faction F if at least one section across all its respawn_params recipes has its squad_descr faction field equal to F. Same vocabulary as squad.player_id at spawn time and xcreature.community at runtime. Smart-terrain props (the routing-side admission flags) are not used -- they control where a squad can move to, not what a smart's recipes spawn.

  Budget filter. After eligibility, AlifeBalance filters smarts to those with open spawn budget right now (the engine's already_spawned[k].num < max gate). If no eligible smart has open budget, the nudge defers: counter holds, retry next tick. This prevents wasted nudges on smarts whose recipes are already at cap, and lets pressure accumulate until a slot frees naturally (engine decrements when one of its tracked squads dies).

  Producer and consumer are decoupled. The death event handler only increments counters. A real-time 60-second tick walks the counters, picks every (level, faction) at or above threshold, applies the budget filter, runs one nudge per pair where filtering finds an open-budget eligible smart, and resets only those pairs' counters. Bursts of deaths cannot produce bursts of nudges.

  Nudge. AlifeBalance clears the smart's respawn cooldown by setting last_respawn_update to nil. The engine treats nil as "first spawn" and passes the cooldown gate on its next alife tick. No budget decrement. No respawn parameter injection.

  Refill <= death invariant. Between any two consecutive nudges of the same (level, faction), the engine refills no more NPCs of that faction on that level than have died. The threshold is the maximum the next refill could possibly spawn.

  The engine handles the spawn. The engine picks one recipe at random from the smart's eligible set, picks one squad section at random from that recipe's pool, calls SIMBOARD:create_squad, sets the new squad's respawn_point_id, increments the smart's budget counter, sets the cooldown timestamp to now. One squad spawns.

  Variance between a death and a spawn:
  - Per-(level, faction) threshold value (derived from LTX).
  - 60-second tick alignment.
  - Random smart pick among eligible smarts on the level.
  - Engine's random pick among eligible recipes on the chosen smart.
  - Engine's random pick from the recipe's squad pool.
  - Engine's random NPC count from the squad section's npc_in_squad range.
  - ZCP substitution if enabled.

Architecture:

Reactive plus periodic. No global scheduler. The death callback only increments a counter keyed by (level, faction). A 60-second real-time tick reads counters and runs at most one nudge per over-threshold pair. The runtime is xlibs, a reverse-engineered API wrapping the X-Ray engine source.

Performance:
- O(1) per death.
- O(L * F) per 60-second tick over death pairs.
- O(S * K) on first death for a (level, faction) pair (S smarts on level, K squads per recipe), then O(1) cached.
- 1 luabind per nudge (field write).
- No persistence. Counters reset on game load.

Compatibility:
- Vanilla and ZCP read the same field AlifeBalance writes (nil short-circuits both cooldown gates).
- Redone Collection and GAMMA NPC Spawns ship pure LTX configuration. AlifeBalance writes runtime state on a different layer.
- Night Mutants uses a parallel SIMBOARD:create_squad path; spawned squads still get respawn_point_id set.
- Nocturnal Mutants spawns entities directly outside smart terrains, ignored.
- GAMMA Dynamic Despawner enforces an online cap on the despawning side. Despawn does not fire squad_on_npc_death, so it does not trigger nudges.
- AlifeGuard, AlifePlus, Warfare, and Guards Spawner have no overlap with AlifeBalance's intervention surface.

The mod has no base script edits and no engine patches.

MCM:
- Smart Pacing: enable toggle.
- Development: log level, show map markers, diagnostics buttons (show status, reset counters). Map markers (green PDA spots, 5-minute linger) flag every nudged smart; right-click teleports to that smart.

Requirements:
- Anomaly 1.5.3
- xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
- MCM

Install (MO2):
1. Install xlibs
2. Install AlifeBalance
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):
Disable or remove in MO2.

Credits:
Altogolik - support, ideas, source materials

Usage and License:
- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in LICENSE file and on GitHub.
