AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
- Version: 1.0.0 (xlibs 1.4.1)
- Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
- Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
- Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Please use the RESET button in MCM when updating to a new version !

Combat outpaces the Zone.

Smart terrains run a respawn cooldown of twelve game-hours (six on ZCP). In long combat sessions on heavily modded setups, squads die faster than the cooldown can refill them. Active zones drain. Whole regions can hollow out across a play session.

AlifeBalance helps the engine refill drained smart terrains sooner, using the smart terrain's own configured spawn pool, on the engine's own update tick.

No entity creation. No spawn injections. No engine gate bypass.
Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, and Redone. Compatible by design.

Example:

A duty patrol gets wiped at the Bar Zastava during a heavy firefight. The smart's configured population for that recipe is two squads; one is now dead. The vanilla respawn cooldown is twelve game-hours. The area sits at half population for twelve in-game hours. With AlifeBalance, within one to four random game-hours, the smart's cooldown timer is backdated and the engine's next try_respawn passes the cooldown gate. The smart spawns one duty squad from its own configured pool. The area refills.

The intervention is one field write. The engine does everything else.

Structural invariants:

- Event-driven: subscribes to engine callbacks. Idle otherwise.
- Engine-native: every intervention is a single write to a vanilla server-entity field the engine reads on its own update tick.
- No fabrication: no entity creation calls of any kind.
- No bypass: every engine gate still applies. Cooldowns, faction filters, smart-terrain props, modpack overrides, despawner caps.
- Configuration-respect: vanilla, ZCP, Redone, or modpack-configured spawn rules remain authoritative. AlifeBalance helps the engine honor them.

Scope:

AlifeBalance hosts balance features that operate on the engine's spawning side, using the engine's own gates and state. The first feature is Smart Pacing. Additional features will be added under the same contract.

Features:

Smart Pacing

  Faction-aware respawn cadence tied to combat death pressure.

  Death tracking. Stalker factions and specific mutant prop keys (predatory_day, predatory_night, vegetarian, zombied_day, zombied_night, special) each have their own counter. Cowardly-tier mutants (rats, tushkanos, flesh, zombies, karliks) do not count toward death pressure. Protected squads (story, companions, task targets) do not count.

  Producer and consumer are decoupled. The death event handler only increments counters. A real-time 60-second tick scans the counters, picks one faction whose count is at or above the threshold, and runs one intervention. Bursts of deaths cannot produce bursts of interventions.

  Intervention. Among smarts on the dying faction's level, the smart must accept the faction and have a matching respawn recipe with budget available (engine's own condlist gate). If no smart qualifies, the intervention is skipped and the counter waits for the next tick. Among qualifying smarts, weighted random pick favors emptier ones.

  Single field write. AlifeBalance overwrites the chosen smart's cooldown timestamp with current game time minus 24 game-hours. No budget decrement. No respawn parameter injection. No timestamp set to nil.

  The engine handles the spawn. On its next try_respawn, the cooldown gate passes immediately (24 game-hours exceeds vanilla 12h and ZCP 6h cooldowns). The engine picks one recipe at random from the smart's eligible set, picks one squad section at random from that recipe's pool, calls SIMBOARD:create_squad, increments the budget counter, sets the cooldown timestamp to now. One squad spawns.

  Variance between a death and a spawn:
  - Counter accumulation timing (deaths arrive irregularly).
  - Threshold value.
  - Random pick among over-threshold factions when several cross threshold in the same tick.
  - Weighted smart pick.
  - Engine's random pick among eligible recipes on the chosen smart.
  - Engine's random pick from the recipe's squad pool.
  - Engine's random NPC count from the squad section's npc_in_squad range.
  - ZCP substitution if enabled.

Architecture:

Reactive plus periodic. No global scheduler. The death callback only increments a counter. A 60-second real-time tick reads counters and runs at most one intervention. The runtime is xlibs, a reverse-engineered API wrapping the X-Ray engine source.

Performance:
- O(1) per death.
- O(L*F + S*K) per 60-second tick. L levels, F factions, S smarts on level, K recipes per smart. Sub-millisecond.
- O(3) luabind per intervention (xtime.game_time, CTime:sub, field write).
- No persistence. Death counters reset on game load.

Compatibility:
- Vanilla and ZCP read the same field AlifeBalance writes through the same code path.
- Redone Collection and GAMMA NPC Spawns ship pure LTX configuration. AlifeBalance writes runtime state on a different layer.
- Night Mutants uses a parallel SIMBOARD:create_squad path that does not touch the same field.
- Nocturnal Mutants spawns entities directly outside smart terrains.
- GAMMA Dynamic Despawner enforces an online cap on the despawning side.
- AlifeGuard, AlifePlus, Warfare, and Guards Spawner have no overlap with AlifeBalance's intervention surface.

The mod has no base script edits and no engine patches.

MCM:
- Smart Pacing: enable, deaths per nudge.
- Development: log level, diagnostics buttons (show status, reset counters).

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
