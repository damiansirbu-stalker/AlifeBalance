AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
- Version: 1.0.0 (xlibs 1.4.1)
- Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
- Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
- Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Please use the RESET button in MCM when updating to a new version !

Combat outpaces the Zone.

Smart terrains run a respawn cooldown of 12 to 96 game-hours depending on which smart. In long combat sessions on heavily modded setups, squads die faster than the cooldown can refill them. Active zones drain. Whole regions can hollow out across a play session.

AlifeBalance helps the engine refill drained smart terrains sooner, using the smart's own configured spawn pool, on the engine's own update tick.

No entity creation. No spawn injections. No engine gate bypass.
Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, and Redone. Compatible by design.

Example:

A duty patrol gets wiped at the Bar Zastava during a heavy firefight. The squad's origin smart now has open budget. AlifeBalance counts the dead NPCs against that smart's threshold; once the threshold is met, the smart's respawn cooldown is cleared. The engine's next try_respawn passes the cooldown gate and spawns one duty squad from the smart's own configured pool. The area refills.

The nudge is one field write. The engine does everything else.

Structural invariants:

- Event-driven: subscribes to engine callbacks. Idle otherwise.
- Engine-native: every nudge is a single nil-write to a vanilla server-entity field the engine reads on its own update tick.
- No fabrication: no entity creation calls of any kind.
- No bypass: every engine gate still applies. Cooldowns, faction filters, smart-terrain props, modpack overrides, despawner caps.
- Refill never outpaces death: per origin smart, between consecutive nudges, the engine refills no more NPCs than have died from squads that smart spawned.
- Configuration-respect: vanilla, ZCP, Redone, or modpack-configured spawn rules remain authoritative. AlifeBalance helps the engine honor them.

Scope:

AlifeBalance hosts balance features that operate on the engine's spawning side, using the engine's own gates and state. The first feature is Smart Pacing. Additional features will be added under the same contract.

Features:

Smart Pacing

  Origin-smart-aware respawn cadence tied to combat death pressure.

  Death tracking. Every NPC death from a squad with a known origin smart counts toward that smart's threshold. Squads without a tracked origin (scripted, save-loaded with broken link, Warfare-managed) are skipped.

  Per-smart threshold. The threshold is the maximum upper bound of npc_in_squad across every squad section the smart can spawn. Derived from engine LTX data. A rat-lair smart has a threshold around 20; a stalker-only smart has a threshold around 4-5. No MCM knob.

  Producer and consumer are decoupled. The death event handler only increments counters. A real-time 60-second tick walks the counters, picks every smart whose count has reached its threshold, runs one nudge per such smart, and resets that smart's count. Bursts of deaths cannot produce bursts of nudges.

  Nudge. AlifeBalance clears the smart's respawn cooldown by setting last_respawn_update to nil. The engine treats nil as "first spawn" and passes the cooldown gate on its next alife tick. No budget decrement. No respawn parameter injection.

  Refill <= death invariant. Between any two consecutive nudges of the same smart, the engine refills no more NPCs than have died. The threshold is the maximum the next refill could possibly spawn, so we never spawn more than was lost.

  The engine handles the spawn. The engine picks one recipe at random from the smart's eligible set, picks one squad section at random from that recipe's pool, calls SIMBOARD:create_squad, sets the new squad's respawn_point_id, increments the smart's budget counter, sets the cooldown timestamp to now. One squad spawns.

  Variance between a death and a spawn:
  - Threshold value (derived per-smart).
  - 60-second tick alignment.
  - Engine's random pick among eligible recipes on the chosen smart.
  - Engine's random pick from the recipe's squad pool.
  - Engine's random NPC count from the squad section's npc_in_squad range.
  - ZCP substitution if enabled.

Architecture:

Reactive plus periodic. No global scheduler. The death callback only increments a counter keyed by squad origin smart. A 60-second real-time tick reads counters and runs at most one nudge per over-threshold smart. The runtime is xlibs, a reverse-engineered API wrapping the X-Ray engine source.

Performance:
- O(1) per death.
- O(N) per 60-second tick, N = smarts with recent deaths.
- O(K) on first death from a smart (K = total squad sections in its recipes), then O(1) cached.
- 1 luabind per nudge (field write).
- No persistence. Counters reset on game load.

Compatibility:
- Vanilla and ZCP read the same field AlifeBalance writes (nil short-circuits both cooldown gates).
- Redone Collection and GAMMA NPC Spawns ship pure LTX configuration. AlifeBalance writes runtime state on a different layer.
- Night Mutants uses a parallel SIMBOARD:create_squad path; spawned squads still get respawn_point_id set, so their deaths feed the same counter.
- Nocturnal Mutants spawns entities directly outside smart terrains. Those squads have no respawn_point_id and are ignored.
- GAMMA Dynamic Despawner enforces an online cap on the despawning side. Despawn does not fire squad_on_npc_death, so it does not trigger nudges.
- AlifeGuard, AlifePlus, Warfare, and Guards Spawner have no overlap with AlifeBalance's intervention surface.

The mod has no base script edits and no engine patches.

MCM:
- Smart Pacing: enable toggle.
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
