# AlifeBalance Architecture

AlifeBalance accelerates respawn at smart terrains whose recipes produce a faction the player has been killing. It counts deaths per (level, faction), and every 60 wall-seconds advances the `last_respawn_update` timestamp of eligible smarts backward, bringing the engine's own cooldown gate closer to expiry. The engine still owns the spawn itself, the recipe choice, the squad section, and the NPC count. AlifeBalance writes exactly one field: `last_respawn_update`.

Two MCM knobs: `advances` (1-8, default 4) sets how many advances drive one smart from full cooldown to the floor; `Min Minutes` (10-360, default 120) sets the floor on remaining cooldown that AlifeBalance never pushes below. With defaults, four advances of accumulated combat push one smart through, and the engine ages the final two game-hours on its own clock before firing `try_respawn`.

Built on xlibs. `_ab_deps` asserts the minimum xlibs version on load.

Part of a three-mod alife family: **AlifePlus** extends A-Life with new behaviors, **AlifeBalance** tunes existing rates and counts (this mod), **AlifeGuard** keeps alife state clean.

![Layered position](img/layers.png)

---

## The cooldown clock

Every smart terrain holds two engine fields that together define when it can spawn next:

- `respawn_idle` — cooldown duration in game-seconds. Set once from LTX (`smart_terrain.script:231`), per-smart. Vanilla LTX default 86400 (24 game hours); the engine code fallback for unset entries is 43200 (12h); ZCP under GAMMA is 21600 (6h); Redone varies. AlifeBalance only reads this field.
- `last_respawn_update` — timestamp of the smart's last spawn. The engine resets it to `curr_time` after every `SIMBOARD:create_squad`. AlifeBalance writes this field; one subtract per advance, in `_advance_smart` at `ab_smart_balance.script:291-305`.

The engine's cooldown gate inside `smart_terrain.script:try_respawn` (delegated to `smr_pop.smart_can_respawn` under ZCP):

```
elapsed         = curr_time - last_respawn_update
gate_open       = elapsed > respawn_idle
hours_remaining = respawn_idle - elapsed       (debug overlay reports this)
```

AlifeBalance brings the gate closer by aging `last_respawn_update` backward:

```
smart.last_respawn_update :sub( (respawn_idle - Min Minutes * 60) / advances seconds )
```

The cooldown duration (`respawn_idle`) never changes; only the timestamp does. The engine's `respawn_idle` setting remains the ground truth for cycle length.

Worked example at vanilla LTX `respawn_idle=86400`, MCM `advances=4`, `Min Minutes=120`:

```
respawn_idle         = 86400 s   = 24.0 game-hours
usable               = 79200 s   = 22.0 game-hours      (respawn_idle - Min Minutes*60)
per-advance subtract = 19800 s   =  5.5 game-hours      (usable / advances)
advances to floor    = 4
floor remaining      =  7200 s   =  2.0 game-hours      (engine ages this naturally)
```

Per-advance subtract scales with the resolved cycle length, so on shorter-cooldown modpacks each advance is a smaller game-time move while the kill cost stays the same:

| Scenario     | resolved idle | usable | per-advance | floor |
|--------------|--------------:|-------:|------------:|------:|
| Vanilla LTX  |           24h |    22h |        5.5h |    2h |
| ZCP @ GAMMA  |            6h |     4h |          1h |    2h |

`_resolve_idle` (`ab_smart_balance.script`) reads `smr_amain_mcm.get_config("respawn_idle")` when ZCP is loaded and enabled; falls through to `smart.respawn_idle` otherwise. ZCP's `smr_pop.smart_can_respawn` gates against the same cvar, so AB and ZCP measure against the same baseline.

The picker's skip conditions in `_can_advance` (called per eligible smart per tick):

- **Degenerate config**: `usable <= 0` (when `Min Minutes * 60 >= respawn_idle`). Vanilla pace owns this smart.
- **Fresh smart**: `last_respawn_update == nil`. Engine cooldown gate is already open; no acceleration needed.
- **`advances >= 2`**: skip if a full advance push would overshoot the floor (`max_push < advance_seconds`). No clamping; the next tick can push again after natural ageing.
- **`advances == 1`**: skip if `max_push < Min Minutes * 60` (smart is within one `Min Minutes` of the floor, default 7200 game-seconds). Precision-drift guard, and avoids wasting a kill on a smart already close to firing naturally.

The picker sorts eligible smarts by actor-to-smart distance descending. Advances pile on the farthest qualifying smart until it hits the floor, then move to the next-farthest. This is a soft preference: if only near-actor smarts are eligible, they still receive advances, and the engine's own `respawn_radius` gate inside `smart_terrain.script:try_respawn` will defer the spawn until the actor moves out.

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
      smarts, threshold = ab_smart_recipe.get_eligible_and_threshold(level_id, faction)
      if #smarts == 0:                              [NOELIG], no action
      else if count < threshold:                    [BELOW],  no action
      else:
        with_budget = filter smarts by _can_advance AND ab_smart_recipe.evaluate_budget_for_faction
        if #with_budget == 0:                       [DEFER],  counter holds
        else:
          sort with_budget by actor->smart distance descending
          while count >= threshold and #with_budget > 0:
            smart = with_budget[1]                  (farthest from actor)
            smart.last_respawn_update:sub(params.delta)
            count -= threshold
            _stats.advances += 1                    [ADVANCE]
            if not _can_advance(smart):
              remove smart from with_budget
          by_faction[faction] = count               (leftover < threshold carries to next tick)
          if advances_this_tick > 1: log [BURST]

ENGINE (its own alife tick at the advanced smart)
  - se_smart_terrain:update -> try_respawn
  - cooldown gate (smart_terrain.script:try_respawn): diff > respawn_idle ? proceed : skip
  - per-recipe budget gate (smart_terrain.script:try_respawn, max_respawn_count > already_spawned[k].num)
  - picks recipe at random from open ones
  - picks squad section at random from recipe.squads
  - SIMBOARD:create_squad(smart, section)
  - sets squad.respawn_point_id = smart.id
  - increments already_spawned[k].num
  - writes last_respawn_update = curr_time
```

**Threshold** for a (level, faction) pair is the MAX `npc_in_squad` upper bound across all squad sections in eligible smarts' recipes that produce the faction. Cordon stalker = 3, Swamp boar = 15. Read from `squad_descr` LTX on first death, cached for the session. No MCM knob — engine-grounded by construction.

**Eligible smart**: a smart on the target level with at least one section in its `respawn_params` recipes whose `squad_descr faction` equals the target faction. Routing props are not checked; they govern where squads can move, not what recipes spawn.

**Open budget**: at least one matching recipe has `max > already_spawned[k].num` right now, where `max = ab_smart_recipe.pick_value_readonly(recipe.num, actor, smart)`.

*Why a side-effect-free walker*: `xr_logic.pick_section_from_condlist` (the engine's reader) runs `cond[3]` effects (info grants, `xr_effects` functors) when the matched branch carries them. AB calls this once per recipe per eligible smart per 60s tick; the engine calls it once per smart per resolved cycle (orders of magnitude rarer). Any side effect a modpack ever writes into a `spawn_num` condlist would fire at AB's pacing, not the engine's. `ab_smart_recipe.pick_value_readonly` mirrors the engine's condition-check semantics exactly (including the type-5 `math.random` draw) but skips the effects block, so AB cannot amplify side effects regardless of what future modpacks write into `spawn_num`.

---

## Engine gates inside try_respawn

`smart_terrain.script:try_respawn` has eight gates. AlifeBalance affects one.

| Gate                                          | Source line | Our effect |
|-----------------------------------------------|-------------|------------|
| Disabled / on_try_respawn callback            | 1607        | Untouched |
| Peace info                                    | 1613        | Untouched |
| Level filter (`respawn_only_level`)           | 1618        | Untouched |
| Actor distance (`respawn_radius`)             | 1625        | Untouched |
| `respawn_params and already_spawned` exist    | 1631        | Untouched |
| Simulation availability                       | 1636        | Untouched |
| Cooldown timer                                | 1657        | Subtract `(respawn_idle - Min Minutes*60) / advances` from `last_respawn_update` per advance |
| Per-recipe budget                             | 1702        | Untouched |

---

## Population invariant

*Scope*: vanilla `try_respawn`. Under ZCP `smr_handle_spawn` the per-faction form degrades; the system-wide form holds. Both are stated below.

**Per-faction form (vanilla):** for any (level, faction) pair, between consecutive engine spawns at any eligible smart for that pair, the engine refills no more NPCs of that faction than have died on that level.

*Proof*: each advance fires only after the counter reaches `threshold = MAX npc_in_squad` across matching sections, and the picker rejects smarts already at the floor. Total deaths between consecutive spawns at a smart `>= applied_advances * threshold`. The spawn creates at most one squad; if the engine picks a matching section, that squad contains at most `MAX npc_in_squad` NPCs of the target faction; if it picks a non-matching section in a mixed pool, zero. Either way `deaths >= MAX >= refill`. ∎

**System-wide form (vanilla + ZCP):** between consecutive engine spawns at any eligible smart, the engine refills no more NPCs than have died on that level (across all factions).

*Proof*: identical bound on advances per `threshold` deaths; ZCP substitution at `smr_pop.script:288-339` swaps the squad section, but produces at most one squad per spawn with bounded `npc_in_squad`. Refill across all factions on that level ≤ deaths across all factions on that level. ∎

The vanilla cooldown ages from game-time alone. If the player ignores a level, no deaths fire, no advances apply, and the engine spawns at vanilla pace after the resolved cycle of game-time. AlifeBalance accelerates the cooldown; it never delays vanilla.

---

## Design rationale: why advance, not clear

Setting `last_respawn_update = nil` bypasses the cooldown gate entirely — the engine fires on its next alife tick. "Combat happened" becomes "spawn now," with no middle ground; combat intensity stops mattering once threshold is crossed once.

Per-advance subtract keeps the gate engaged. Each advance is a small constant push, and sustained combat produces sustained refill. Burst combat that crosses threshold once produces `1 / advances` of a cycle; the vanilla cooldown ages out the rest naturally if combat stops. The pacing matches death rate without overriding the engine's own gate.

The `Min Minutes` floor guarantees the engine fires the last leg on its own clock. AlifeBalance pushes closer but never to expiry; from the floor the engine's natural ageing closes the gap and `try_respawn` runs from the engine's clock, not ours. The picker drops smarts whose age would cross the floor on the next advance, so the floor is a hard guarantee, not a soft target.

---

## State and callbacks

Owned state, all in-memory, reset on game load:

| Owner      | State                                | Purpose |
|------------|--------------------------------------|---------|
| ab_smart_balance  | `_deaths[level_id][faction]`         | death counter per pair, decremented by threshold per advance |
| ab_smart_recipe  | `_eligible[level_id][faction]`       | cached eligible smarts, recipe-content derived, static for the session |
| ab_smart_recipe  | `_thresholds[level_id][faction]`     | cached MAX `npc_in_squad` across matching sections |
| ab_smart_balance  | `_delta_cache[smart_id]`             | cached `{ delta, advance, max_age }`, invalidated on MCM change or reset |
| ab_smart_balance  | `_smart_stats[smart_id]`             | per-smart advance bookkeeping for the right-click "Show stats" tip (lifetime scope) |
| ab_smart_balance  | `_advance_pending[smart_id]`         | set on advance, consumed on next spawn at the smart; drives `from_us` correlation in `[SPAWN]` log + `spawns_from_us` counter |
| ab_smart_balance  | `_seen_squads[squad.id]`             | dedup set for spawn tracing (DEBUG only) |
| ab_smart_map     | `_markers[smart_id]`                 | linger expiry per marked smart |
| ab_smart_balance  | `_stats`                             | 10 diagnostic counters (deaths, counted, vermin, ticks, advances, spawns, etc.) |

Callbacks: `squad_on_npc_death`, `squad_on_npc_creation`, `on_option_change`, `load_state`, `actor_on_first_update`, `map_spot_menu_add_property`, `map_spot_menu_property_clicked`. One `CreateTimeEvent` timer at 60-second wall interval.

No persistence for AlifeBalance state. Advanced `last_respawn_update` values survive in the engine's own save data via `utils_data.w_CTime / r_CTime`.

Not owned by AlifeBalance: which recipe the engine picks (random over open-budget), which squad section within that recipe (random), NPC count within `npc_in_squad` range (random), ZCP `smr_handle_spawn` substitution, online cap enforcement (GAMMA Dynamic Despawner, AlifeGuard).

---

## Inventory Balance

The goal is to let the player keep vanilla NPC corpse looting enabled. Vanilla looting pays two costs over long sessions: jackpot bodies (one stalker carrying a vendor run of gear), and creep toward the engine's 65535 alife-ID cap as every looted item consumes one ID (vanilla Anomaly warns at 64000 via `alife_on_limit`). Anti-loot addons (NPC Stop Looting Dead Bodies, Weapons Drop on Bodies, BoltBeGone) sidestep both by blocking or rewriting the loot path. Inventory Balance bounds hoarding at the source instead, so anti-loot addons are no longer needed.

Periodic scanner over online stalkers. Walks the online set in small batches, trims one NPC per frame, rescans each NPC at most once per game-day. Scope is random long-lived stalkers (gulag survivors, generic patrols). Companions, story NPCs, traders, and named characters are filtered out at the scheduler via `xcreature.is_unscriptable` and never reach `trim_npc`.

### Why scanner, not death-time hook

The previous shipped approach wrapped `death_manager.keep_item` and only ran when an NPC died. Three failure modes were missed:

1. **Long-lived NPCs never trim**. The population that survives is the population that hoards. Hundreds of random online stalkers (gulag survivors, generic patrols) keep looting corpses they walk over and never die. Death-time only catches NPCs that die; the survivors drive save bloat and steady performance drag indefinitely. (Companions, story NPCs, traders, and named characters are scripted-identity holders filtered at the scheduler entirely, so the hoarding problem is the random long-lived population only.)
2. **Save bloat accumulates between deaths**. Every looted item is a server object persisted in the save. Long sessions accumulate without bound. Continuous trim bounds live state instead of waiting for the death event.
3. **Bursty performance**. N deaths in a firefight = N trims in the same frame as the corpse spawn. xslice spreads the trim cost across frames.

The engine `utils_item.is_overweight(npc)` self-limit at `xr_corpse_detection.script:421` caps live hoarding by weight (around 50kg) but the cap is generous; NPCs accumulate plenty before hitting it.

### Pipeline

```
TICK (every 30 wall-seconds via actor_on_update)
  |
  v
_start_cycle()
  - if not enabled_inventory: return
  - if xslice.is_active("ab_inventory_balance_scan"): return
  - now = xtime.game_sec()
  - eligible = [ npc_id for npc in xcreature.online_iter()
                 if IsStalker(npc) and npc:alive()
                    and not xcreature.is_unscriptable(npc)
                    and now - _last_scan_game_sec[npc_id] >= scan_cooldown_h * 3600 ]
  - sort eligible by oldest scan first
  - picks = all eligible (xslice paces the actual trim at npcs_per_frame)
  - if #picks == 0: return
  - cycle_id += 1; open xprofiler
  - xslice.start("ab_inventory_balance_scan", picks, { step = npcs_per_frame, func = _visit, on_done = _on_cycle_done })

FRAME (each frame while queue active)
  |
  v
_visit(npc_id)
  - npc = level.object_by_id(npc_id)
  - if not npc or not npc:alive(): return true  (drain; went offline)
  - r = trim_npc(npc)
  - _last_scan_game_sec[npc_id] = xtime.game_sec()
  - cycle counters += r
  - return true  (drain)

CYCLE END
  |
  v
_on_cycle_done()
  - log [SCAN] cycle summary (cycle_id, visited, released, dt_ms)
```

### Public API

| Function | Purpose | Returns |
|---|---|---|
| `trim_npc(npc, opts)` | Apply inventory policy to one NPC. opts: `{ dry_run }`. Cooldown table NOT touched. | `{ released, released_by_category, dt_ms }` |

Probes, MCM "trim now" buttons, TestZone probes, and console diagnostics call `trim_npc` directly without scheduler involvement.

### Policy

Policy values live in `gamedata/configs/alifebalance/ab_inventory_policy.ltx` (DLTX-overridable). Single uniform block `[ab_inventory_policy]` with single-value `<category> = <max>` rows. Shared LTX shape with `ap_trade_policy.ltx` and `ap_stash_policy.ltx`; loaded once at on_game_start via `xinventory.load_policy` and applied per-NPC via `xinventory.classify` (counts) + `xinventory.iterate_surplus` (release pass with `on_surplus = xinventory.release_item`).

| Category | max | Effect |
|---|---|---|
| ammo_slot_3_t1, ammo_slot_3_t2 | 120 (rounds) | NPC keeps 4 mags of rifle ammo per tier |
| ammo_slot_2_t1, ammo_slot_2_t2 | 48 (rounds) | NPC keeps 3 mags of pistol ammo per tier |
| ammo_not_equipped | 0 | Always release mismatched ammo |
| grenade | 3 | Hand grenade buffer |
| grenade_ammo | 6 | Launcher rounds (vog-25, og-7b, m209) |
| medkit, bandage | 5 | Self-heal supply; matches trade veteran max so trade-acquired bandages survive between visits |
| food | 5 | Spare consumables |
| antirad, stim, pill | 3 | Niche consumables |
| drink | 3 | NPCs rarely benefit |
| weapon | 1 | One spare for trading (equipped pre-filtered) |
| outfit, helmet, device | 1 | One spare each (equipped pre-filtered) |
| artefact, crafting | 3 | Harvested artefacts / tools-parts-upgrades for traders |
| other | 3 | Fallthrough sentinel; small cap to bound unknown items |

`money` (kind=i_money pickups) is intentionally absent from the LTX. Cull's `on_surplus` is `release_item` which destroys; the same LTX value `0, 0` in `ap_trade_policy.ltx` only transfers items to the seller. Cull would destroy money piles. No-rule = keep is the safe default for this consumer.

Ammo categories count in ROUNDS (sum of `ammo_get_count` per stack via `xinventory.classify`); other categories count in items. Untouchables (quest / anim / blacklisted) and equipped items are pre-filtered by `xinventory.get_category`. Three runtime per-item untouchable checks also gate via xinventory: items with `get_object_story_id`, items the actor gave to a companion (`axr_companions.is_assigned_item`), and player-strapped weapons (`se_load_var "strapped_item"`). None of these reach the policy.

Companions, traders, story characters, and named NPCs are filtered at the scheduler via `xcreature.is_unscriptable(obj)` and never reach `trim_npc`; the policy only sees random extras whose identities no script depends on.

### State

| State | Purpose |
|---|---|
| `_last_scan_game_sec[npc_id]` | game-second of last visit. Persists for session only; on game load every NPC is eligible. Stale ids harmless (id allocator never recycles within a save). |
| `_cycle_id`, `_cycle_tid`, `_cycle_visited`, `_cycle_released`, `_cycle_timer` | per-cycle counters. Reset in `_start_cycle`. |
| `_last_update`, `_dbg` | wall-clock gate (os.clock), debug-level mirror. |

No persistence. The cooldown table not saved; on game load every NPC is fresh and the first round of cycles trims everyone, then steady-state takes over.

### Prerequisites and conflicts

| Mod / setting | Interaction |
|---|---|
| Vanilla NPC corpse looting enabled | Required. Inventory Balance bounds vanilla looting at the source. With looting disabled the scanner runs but finds nothing to release. |
| Anti-loot addons (`311- NPC Stop Looting Dead Bodies`, `BoltBeGone`, equivalents) | Anti-loot addons Inventory Balance replaces. Disable. They were created to work around two costs of vanilla looting (jackpot kills, 65k alife-ID cap) by blocking or rewriting the corpse-loot path. Inventory Balance bounds the cause, so the workarounds are no longer needed. |
| Jabbers' "Weapons Drop on bodies" 134 | No conflict. They patch `death_manager.keep_item`; we no longer touch that seam. The scanner releases from online inventories; their wrap fires at death on whatever the scanner left behind. |
| Ish's BoltBeGone in Nitpicker's Modpack 124 | Same as Jabbers'. No conflict. |
| Unscriptable NPCs (`xcreature.is_unscriptable`) | Skipped at scheduler level. Covers story characters (Strider, Magpie, Sidorovich) via engine story_id, companions via `npcx_is_companion` info-portion, named NPCs (traders, medics, mechanics, guides, guards, bodyguards, leaders, quest-givers) via `xdata.unscriptable_npcs`, and members of story squads. These NPCs have scripted identities the rest of the game depends on; trimming their inventories risks breaking quest scripts or destroying trader stock. |
| Untouchables (`xinventory.get_section_category(sec) == "untouchable"`) | Never released regardless of category. Covers quest items (`quest_item=1` or `kind=i_quest`), animation props (`anim_item=1`), and sections in `xr_corpse_detection.ltx [ignore_sections]`. Resolved once per section by xinventory and cached. |

### Performance

| Operation | Cost |
|---|---|
| `_start_cycle`, eligible-set walk | O(N online) with N ~ 200 typical. 1 `xtime.game_sec()` + per-NPC `IsStalker` + `alive()` + `xcreature.is_unscriptable` (weak-keyed cache, 0 luabind on hit) + hash read. |
| `_pick_eligible` sort | O(K log K) where K = eligible count. `table.sort` over an array of tables. |
| Per NPC visit | O(items) inventory walk. 12 `item_in_slot` (one per main slot) + ~2 `parse_list` (ammo_class for slots 2 and 3) for state, plus per-item policy (hashes + clsid + IsItem/IsWeapon/IsOutfit/IsHeadgear/IsArtefact bridges). |
| Per release | 1 `alife_object` + 1 `alife_release`. |
| Per cycle dt | Single-digit ms for typical batches (20 NPCs * 1 per frame = 20 frames). |
| Idle cost | 1 `xslice.is_active` hash lookup per actor_on_update + 1 `os.clock()` compare. |

---

## Files

| File | Purpose |
|------|---------|
| `gamedata/scripts/_ab_deps.script` | Version string, xlibs dependency gate |
| `gamedata/scripts/ab_mcm.script` | MCM defaults, UI definition, button handlers |
| `gamedata/scripts/ab_smart_balance.script` | Death handler, periodic tick burst-advance loop, per-smart CTime delta cache, `_advance_smart` cooldown subtract, public `marker_label` + `show_smart_stats` for ab_smart_map |
| `gamedata/scripts/ab_smart_recipe.script` | Per-(level, faction) eligibility + threshold cache, single-pass budget evaluation. Two entry points called by ab_smart_balance: `get_eligible_and_threshold` and `evaluate_budget_for_faction`. Delegates smart discovery and squad-size lookup to xsmart. |
| `gamedata/scripts/ab_smart_map.script` | PDA marker render-state, right-click menu (teleport, show stats). Calls back into ab_smart_balance for label + stats formatting. |
| `gamedata/scripts/ab_inventory_balance.script` | Online inventory scanner. Public API `trim_npc`, xslice scheduler with per-NPC game-day cooldown. Policy loaded from `ab_inventory_policy.ltx` via `xinventory.load_policy`. |
| `gamedata/configs/alifebalance/ab_inventory_policy.ltx` | Per-category `{min, max}` ceilings (DLTX-overridable). Single block `[ab_inventory_policy]`; shared LTX shape with `ap_trade_policy.ltx` and `ap_stash_policy.ltx`. |
| `gamedata/scripts/ab_test.script` | Console-driven test harness. Kill loop fires fake NPC deaths every 3s, alternating on-level / off-level pools (same vermin filter as ab_smart_balance). Flow tests (`ab_test_<flow>()`) inject inventory and assert `ab_inventory_balance.trim_npc` results against the LTX policy. |
| `gamedata/configs/text/eng/ui_st_mcm_ab.xml` | MCM strings (English) |
| `gamedata/configs/text/rus/ui_st_mcm_ab.xml` | MCM strings (Russian) |
| `gamedata/textures/ab_mcm_banner.dds` | MCM banner (512x50) |

---

## MCM

| Setting              | Tab               | Type   | Default | Range   | Effect |
|----------------------|-------------------|--------|---------|---------|--------|
| `enabled`            | Smart Balance     | check  | true    | -       | Master toggle |
| `advances`           | Smart Balance     | track  | 4       | 1-8     | Per-advance subtract is `(respawn_idle - Min Minutes*60) / advances` |
| `Min Minutes`        | Smart Balance     | track  | 120     | 10-360  | Minimum cooldown remaining after every advance, in game minutes |
| `enabled_inventory`  | Inventory Balance | check  | true    | -       | Scanner enable. When off, no scheduler, no trims. |
| `npcs_per_frame`     | Inventory Balance | track  | 1       | 1-10    | xslice step: NPCs trimmed per frame inside a cycle. |
| `scan_cooldown_h`    | Inventory Balance | track  | 24      | 1-72    | Per-NPC rescan cooldown in game-hours. |
| `log_level`          | Development       | list   | WARN    | -       | ERROR / WARN / INFO / DEBUG |
| `show_markers`       | Development       | check  | false   | -       | Green PDA marker on every advanced smart, 5-min linger, right-click teleport / stats |
| `btn_reset_all`      | Development       | button | -       | -       | Restore all MCM settings to factory defaults. Closes the MCM dialog via On_Cancel (not On_Discard, see ab_mcm.script comment for the SEH-on-some-exes rationale). |

Threshold has no knob — it is engine-grounded, read from `squad_descr` LTX per (level, faction) and cached.

---

## Performance

| Operation | Cost |
|-----------|------|
| Per death | O(1). Protection check, vermin species lookup (cached), faction read, level lookup, counter increment. |
| `ab_smart_recipe.get_eligible_and_threshold`, first call per pair | O(S * R * Q) over smarts on level * recipes per smart * sections per recipe. 2 luabind `ini_sys:r_string_ex` per section. Cached. |
| `ab_smart_recipe.get_eligible_and_threshold`, cache hit | O(1) |
| `ab_smart_recipe.evaluate_budget_for_faction`, per tick per eligible smart | O(R * Q) over recipes * sections. 1 `pick_section_from_condlist` per recipe + 1 `ini_sys:r_string_ex` per section until first faction match. |
| `_get_advance_params`, first call per smart | 2 CTime constructions + 2 `setHMSms` + 1 operator-. ~5 luabind. Cached. |
| `_get_advance_params`, cache hit | O(1) |
| `_can_advance`, per tick per eligible smart | 1 luabind: `xtime.game_time():diffSec(lru)`. |
| Per advance | 3 luabind: `last_respawn_update` read, `:sub` with cached delta, write back. |
| Per tick (60s) | O(L * F) over death pairs * O(E * R * Q) budget evaluation per pair, plus one-time computes per new pair or new smart. |

---

## Compatibility

| Mod | Interaction |
|-----|-------------|
| Vanilla `try_respawn` | Reads `last_respawn_update` inside `smart_terrain.script:try_respawn` (cooldown gate). The advance moves the same field. Budget eval applies `ui_options.get("alife/general/alife_stalker_pop" / "..._mutant_pop")` to match the engine's pop_factor multiplier inside the same function. |
| ZCP (forked `try_respawn`) | Gates against `smr_amain_mcm.get_config("respawn_idle")`, not `smart.respawn_idle`. AB's `_resolve_idle` reads the same cvar so the per-advance subtract sizes against ZCP's actual gate. Budget eval applies `smr_amain_mcm.get_config("stalker_pop_factor" / "monster_pop_factor")` to match `smr_pop.get_*_pop_factor` at `smr_pop.script:372-389`. The `Min Minutes` floor holds against the resolved cycle. |
| ZCP `smr_handle_spawn` | Substitutes the squad section in flight (zombie 90% under Survival; random mutant for monster squads; faction reshuffle for stalkers per `smr_pop.script:288-339`). Replacement still gets `respawn_point_id` and `respawn_point_prop_section` set, so engine budget accounting stays intact. The (level, faction) population invariant degrades to "deaths bound system-wide refill, not per-faction refill" under ZCP: AB credits faction A's kills, ZCP may produce faction B's squad. Total spawn-rate cap holds; per-faction accounting does not. |
| Redone Collection | Pure LTX configuration. AlifeBalance writes runtime state. No conflict. |
| GAMMA NPC Spawns | Pure LTX. No conflict. |
| Night Mutants | Parallel `SIMBOARD:create_squad` path. Spawned squads still get `respawn_point_id` set. No conflict. |
| Nocturnal Mutants | Raw `alife_create`. Bypasses smart terrains entirely. Independent. |
| GAMMA Dynamic Despawner | Enforces online cap on the despawning side. Despawn does not fire `squad_on_npc_death`. No false advances. |
| AlifeGuard | Squad-aware despawner. Same: despawn is not death. |
| AlifePlus | Reactive A-Life framework. Different fields and patterns. No overlap. |
| Warfare | `faction_controlled` smarts (16 of ~490) have an injected `respawn_params` entry whose `.squads` field is the vanilla `squads_by_faction[faction]` list. `xsmart.section_faction` reads those vanilla sections normally. Works. |
