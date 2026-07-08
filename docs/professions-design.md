# Professions & Combat Classes

Companion to `minestom-backend-plan.md` and `engine-improvements.md`. Two **separate**
progression systems — economic professions (SWG pre-CU style) and combat classes (fighting
styles) — sharing one substrate. Engine claims verified against this tree (MC 26.1.2).

## Design principle: two tracks, one economy

SWG pre-CU put combat and crafting in a single 250-skill-point budget. At MMO population that
created hybrids and interdependence; at SMP population (~dozens) it would force everyone into
half-fighter compromises and hollow out the crafting economy. So:

- **Economic professions** and **combat classes** have separate point pools, separate XP
  streams, and separate trees. Every player carries one of each.
- They interlock *only through the economy and the world*: Blacksmiths make the weapons the
  styles certify in, Alchemists brew the Assassin's poisons and the Doctor's wound kits,
  Entertainers clear battle fatigue, Builders raise the rooms all of it happens in, and combat
  consequences flow into the existing justice stack (Patrol heat, Mehen governance).
- Nothing good drops from mobs. We own loot outright (Minestom ships none), so the SWG stance
  — crafters make everything, gear decays, resources rotate in quality — is a founding rule,
  not a retrofit.

## Shared substrate (built once, used by both)

| Piece | Mechanism |
|---|---|
| Identity & persistence | MariaDB alongside `governance` (Mehen `links` already binds MC UUID ↔ Discord); skill/XP tables keyed by canonical UUID |
| Skill graphs | Data-driven (YAML → registry): boxes with prerequisites, XP type, point cost, granted attribute modifiers / certifications / abilities |
| UI | **Server-driven dialogs** (`net.minestom.server.dialog`, `ShowDialogPacket`) — real UI screens on vanilla clients for tree browsing, respec confirmation, crafting experimentation |
| Stat effects | **Attribute system**, per-player server-set: `BLOCK_BREAK_SPEED`, `MINING_EFFICIENCY`, `ENTITY_INTERACTION_RANGE`, `JUMP_STRENGTH`, `LUCK`, `ATTACK_*`, `MOVEMENT_SPEED`, `KNOCKBACK_RESISTANCE`, `SAFE_FALL_DISTANCE`, `SCALE`, … — skill boxes grant mechanically real, client-enforced bonuses |
| Abilities | One shared framework (cooldowns, costs, triggers) used by combat actives and Enchanter magic; triggers from item use / sneak-combos / hotbar items / dialogs (no custom keybinds on vanilla clients) |
| Custom items | Daggers, billy clubs, short/long spears, great axes don't exist in vanilla — item components + custom models via the resource pack **emojibridge already serves**. Mace and wind charge are vanilla 1.21 items |
| Discord | Milestones published on Redis → mehen-bot grants roles ("Master Weaponsmith", "Juggernaut") — one pub/sub message on the existing bus |

---

## System 1 — Economic professions

Seven base professions, three specializations each. Use-based XP only: you get better at
mining by mining, at performing by holding an audience.

| Base | Specializations |
|---|---|
| Enchanter | Warlock, Wizard, Druid |
| Alchemist | Doctor, Medic, Witch |
| Pleb/Peasant | Farmer, Rancher, Angler |
| Entertainer | Host, Bard, Performer |
| Laborer | Forester, Miner, Forager |
| Builder | Mason, Carpenter, Inventor |
| Blacksmith | Toolsmith, Armorsmith, Weaponsmith |

**Skill points:** budget tuned so a player can master one base + one specialization and
dabble in a second base. Mastery is scarce identity, not a grind checkpoint. Respec =
surrender boxes (dialog-confirmed), refunding points but not XP.

**System hooks (each profession rides infrastructure already planned):**

- **Pleb** — Farmer: crop growth reads the climate field (season/temperature/precipitation);
  Rancher: mob layer + breeding AI, ecology map shared with WildAnimalBalancer; Angler: the
  volumetric water sim + our loot system (fishing tables are ours).
- **Laborer** — gathering XP; the SWG resource-quality system lands here: resource nodes
  whose stats shift by region and season, driven by the same climate field. Skill grants via
  `MINING_EFFICIENCY`, `BLOCK_BREAK_SPEED`, `LUCK`.
- **Builder** — hooks **rooms quality scoring**: crafted furnishings legitimately raise room
  tier; Mason/Carpenter certify structural blocks; Inventor points at water-sim machinery
  (wheels, gates, pumps).
- **Entertainer** — the cantina loop: battle fatigue (see combat) is cleared **only by an
  entertainer performing in a declared TAVERN/THEATER room**. Rooms is the social-hub engine;
  performance XP scales with live audience.
- **Alchemist** — possible because we own the health model: wounds (max-health damage pools)
  healable only by Doctor/Medic; Witch handles the stranger brews (Assassin poisons,
  Enchanter reagents).
- **Blacksmith** — experimentation crafting: continuous stat outcomes from resource quality +
  skill + rolls; output carries real attribute modifiers on item components. **Item decay**
  keeps demand permanent.
- **Enchanter** — the one net-new system (magic on the shared ability framework); Druid gets
  climate-sim hooks (weather sense, small local nudges).

---

## System 2 — Combat classes (fighting styles)

Structure (from the design discussions): **five fundamental styles**, each branching into
**three specialties**. A specialty **retains its fundamental's bonuses** — a Lancer is still a
Spearman underneath. Armor rating is a **certification**: **N**one / **L**ight / **M**edium /
**H**eavy — wear classes above your cert and the style's bonuses switch off (plus
movement/stamina penalties). Armor class → attribute-modifier packages we define, since armor
values are ours (`ARMOR`, `ARMOR_TOUGHNESS`, `MOVEMENT_SPEED`, `KNOCKBACK_RESISTANCE`).

| Fundamental | Armor | Core kit | Specialties |
|---|---|---|---|
| **Scout** | N/L | Move speed, dodge roll, vault; marks targets with glowing via spyglass (toggleable) | Outrider, Assassin, Pioneer |
| **Spearman** | N/L | Short spear + shield, or two-handed long spear; reach | Lancer, Pirate, Hoplite |
| **Swordsman** | N/M | Sword and board, 50/50 attack/defense | Pig Rider, Mobile Wall, Duelist |
| **Archer** | N/L | Bow focus; increased mobility while using one | Ranger, Baller, Crossbowman |
| **Berserker** | L/M | Axe + shield; big damage, quick turtle, lash out again | Valkyrie, Slayer, Juggernaut |

| Specialty | Of | Armor | Core mechanics |
|---|---|---|---|
| Outrider | Scout | N/L | The fastest mounted style; top speed TBD |
| Assassin | Scout | N/L | Stealth, sneak attacks, tainted daggers; city infiltration |
| Pioneer | Scout | N/L | Solo exploration and independence: clears own exhaustion, harder for mobs to notice, affinity with wolves/tamables |
| Lancer | Spearman | L/M | Continuous mounted spear charges |
| Pirate | Spearman | N/L | Array of fishing-rod casts to hook/displace enemies; trident primary |
| Hoplite | Spearman | L/M | Long spear **with** shield; durable, mobile; trades damage for survivability |
| Pig Rider | Swordsman | L/M | Law enforcement: billy club + shield; knockout, load onto pig, transport to prison |
| Mobile Wall | Swordsman | M/H | Shield specialist; buffs nearby allies while shield raised; horseback trample |
| Duelist | Swordsman | N/L | Opening-cut ability; single sword, no shield; faster, higher damage |
| Ranger | Archer | N/L | Mounted/speedy archer — the faster they move, the better the bow |
| Baller | Archer | N/L | Throws anything (paper, fire charges, slime balls, snowballs, wind charges, bricks, clay, pearls, potions, poisonous potatoes, eggs); shield-blocking an entity deflects it back with increased velocity |
| Crossbowman | Archer | M/H | More armor, more damage, armor penetration; vanilla rate of fire; rockets |
| Valkyrie | Berserker | L/M | Mace specialist; very jumpy and fally |
| Slayer | Berserker | N/L | Less armor and less health → more damage; greataxe |
| Juggernaut | Berserker | M/H | Mace and/or greataxe; tankiest style in full Netherite but slower; reduced fall damage to complement the mace |

**Multiclassing rules** (from the discussions):
- Multiclassing is fully allowed; the point budget means a multiclasser **forgoes the final
  levels** of their styles — mastery is the price of breadth. (Canonical example: Slayer with
  Scout levels up to the dodge roll = the glass-cannon build.)
- **One specialty per fundamental**: a Pirate Baller is legal (Spearman + Archer trees); a
  Pirate Lancer is not (both Spearman specialties).
- Awkward pairings largely self-police through **conflicting armor certifications** (a
  Juggernaut Assassin's M/H and N/L kits switch each other's bonuses off) plus targeted
  restrictions where needed.
- The economic professions use the same shape (pick a base, specialize after some levels,
  one specialization per base) so both systems read identically to players.

### Mechanics → engine mapping (all verified available)

- **Reach (Spearman/Hoplite):** `ENTITY_INTERACTION_RANGE` / `BLOCK_INTERACTION_RANGE`
  attributes set per player — real per-player reach; hit validation is server-side anyway
  since the damage pipeline is ours.
- **Stealth (Assassin) / mob-notice (Pioneer):** `Entity.updateViewableRule(Predicate<Player>)`
  — per-viewer visibility, so a stealthed Assassin is *absent from chosen clients*, not
  potion-invisible. Mob perception is a parameter of **our** AI target selectors, so
  Pioneer's "harder for mobs to notice" is a multiplier in our own code — trivial here,
  impossible in vanilla.
- **Spyglass mark (Scout):** spyglass use is a detectable item-use state + look-ray we cast
  server-side; the mark applies the glowing flag via entity metadata, which we can send
  **per-viewer** (targeted metadata packets), so marks can be ally-only and toggleable.
  Pioneer extension from the discussions: tamed wolves (our AI) accept the marked entity as
  a pack target — one line in a target selector we own.
- **Mounts (Outrider, Lancer, Pig Rider, Mobile Wall trample, Ranger):** full
  passenger/vehicle API (`addPassenger`/`getVehicle`); saddled steering for pig/horse;
  restrain-and-transport = the restrained player as passenger on the pig. Charge/trample
  damage = mount velocity in our damage formula; Ranger's "faster you go, better the bow"
  and Lancer's continuous charges are the same velocity input feeding draw time and damage.
- **Knockout/restrain (Pig Rider):** unconscious = custom state we own (immobilize via
  attributes, screen effects, interaction lockout). Jail feeds the existing justice stack —
  Pig Rider is the *player-driven* complement to Patrol's golem constabulary, warrants come
  from Patrol heat, bans/records from Mehen.
- **Dodge roll/vault (Scout), high jump (Valkyrie), Juggernaut fall reduction:** velocity
  impulses + brief i-frames in our damage pipeline; `JUMP_STRENGTH`, `SAFE_FALL_DISTANCE`,
  and `FALL_DAMAGE_MULTIPLIER` attributes all exist for exactly this.
- **Projectiles (Baller, Pirate, Crossbowman rockets):** projectile behavior is fully custom
  (Minestom ships none) — throwing bricks and poisonous potatoes costs the same as snowballs.
  Fishing-rod displacement = our hook entity + velocity pulls.
- **Shield mechanics (Swordsman/Mobile Wall/Berserker/Baller):** the blocking model is ours —
  shield-raise events, damage reduction, axe shield-break, the Mobile Wall aura (AoE ally
  effects while blocking), and the Baller's deflect (blocked entity re-launched with
  amplified velocity — the block event hands us the projectile, we flip and scale its
  vector) are all rules in our combat layer.
- **Conditional stats (Slayer, Archer/Crossbowman daggers, Ranger):** dynamic attribute
  modifiers recomputed on equip/health/mount-state/velocity change — cheap server-side
  listeners feeding the attribute API.
- **Exhaustion (Pioneer), battle fatigue:** our stamina/hunger layer (Phase 2 owns food) —
  fatigue accrues in combat, cleared by Entertainers (the interdependence bridge).
- **Armor-pen (Crossbowman), damage formulas:** the whole damage pipeline is ours; armor pen
  is a term in a formula we write, not a fight with vanilla mechanics.

### Progression

Combat XP per weapon-family, earned by use (landing hits, surviving engagements, mounted
time); style tiers (Novice → I–IV → Mastery) unlock abilities and certifications. Separate
point pool sized so one style is masterable with a second at mid-tier — commitment matters,
but a Juggernaut can still ride a horse badly.

---

## Interlocks (the whole point)

- Blacksmith **Weaponsmith/Armorsmith** make every certified weapon and armor class; decay
  keeps them employed. Toolsmith supplies the Laborers.
- Alchemist line supplies Assassin poisons, combat wound treatment, buffs before a siege.
- Entertainers clear battle fatigue in Builder-built, rooms-scored taverns.
- Pig Rider + Patrol + Mehen: one justice pipeline from heat → warrant → arrest → jail →
  record.
- Climate & water sims feed Farmer/Angler/Forager supply lines that provision everything
  above.

## Phasing

- **Phase 2 dependency:** the combat layer (damage pipeline, blocking, projectiles, stamina)
  is Phase 2's survival baseline — combat classes are *rules layered on it*, so build that
  layer class-aware from the start (damage events carry style context).
- **Phase 3:** shared substrate (graphs, XP, persistence, dialogs, abilities) + first
  playable slice: the five fundamentals (Scout, Spearman, Swordsman, Archer, Berserker) +
  Laborer/Blacksmith loop to prove the economy. Specialties follow once their supporting
  systems exist.
- **Phase 3+:** mounted specialties (need the mount layer), Assassin/Pioneer (need AI
  perception + tamables), Pig Rider (needs jail/justice wiring), Enchanter (needs the magic
  framework).

## Decisions (2026-07-08 grill)

These resolve the design's five open questions (point budgets, respec, PvP scope, Pig Rider
entry, Outrider/Scout tuning). Recorded with the reasoning so the trade-offs aren't
re-litigated.

**1. Point budgets and specialty gating.**
Both buckets use one shared shape: *master one + dabble one*. The budget funds one base/style
fully to Mastery plus a second to roughly tier II-III, no further. Progression runs a six-tier
ladder (Novice, I, II, III, IV, Mastery) in both tracks. The specialty branch unlocks at
**tier II**: the fundamental's core kit is delivered and felt by then, and four tiers of
headroom remain for the specialty to develop. Consequence: a main tree reaches a fully
developed specialty, a dabble second tree only tastes one, and a three-tree multiclasser
Masters nothing (breadth costs mastery).

**2. Respec friction and the XP model.**
Respec stays surrender-only (dialog-confirmed): reclaim the point immediately, forfeit the XP
spent into the surrendered box. No cooldown, no material cost; the XP re-grind is the friction
and it self-scales with box depth. Anti-hoard rule: while a bucket is fully allocated (maxed),
that bucket stops gaining XP, so a finished character cannot bank an XP reserve to instantly
refund a respec. Maxing is per-bucket (max combat, combat XP halts while profession XP keeps
flowing). XP is typed and earned by use in both buckets; whether it lives as a named pool or as
per-box counters is an implementation detail, since the design rules hold either way.

**3. PvP scope: always-on, consequence-gated.**
Styles are live in the open world everywhere, not arena-gated. Aggression is policed by
consequence, not by context, reusing the existing justice stack:

- Open world: PvP live; unprovoked aggression spikes consent-weighted Patrol heat, which drives
  the warrant, arrest, and record pipeline.
- Declared War: a Mehen-mediated conflict between groups that suppresses heat between
  belligerents inside its scope; the sanctioned outlet for large-group fighting (see glossary).
- Duel / arena: opt-in, zero heat, for sport and practice.

See ADR docs/adr/0001-pvp-always-on-consequence-gated.md for the full rationale.

**4. Pig Rider entry: open spec, warrant-gated arrest.**
Anyone can spec Pig Rider. The knockout is ordinary combat aggression (unlawful use generates
heat like any attack), and imprisonment only completes against a target carrying an active
warrant (Patrol heat over threshold). The class self-polices through systems already built; no
governance appointment required. A hook is left for Mehen to grant *expanded* authority
(local-law arrests beyond global heat) to deputized players later.

**5. Outrider speed and Scout's mark.**
Outrider top speed is specified relatively, not as a fixed number: the fastest land movement in
the game, roughly 1.3-1.5x the best non-Outrider mount, hard-capped below the server's reliable
chunk-generation throughput so a rider can never outrun terrain (guards the frontier chunk-hole
hazard). Exact value tuned in playtest. Scout's spyglass mark is ally-only (per-viewer metadata
already supports it), toggleable, and bounded by line-of-sight refresh or duration so it is a
tactical designation, not a permanent wallhack. A public "wanted" glow, if ever needed, comes
from the warrant system, not this mark.

## Glossary additions

**Declared War** - a conflict formally declared through Mehen governance between two groups or
settlements. Within its scope, Patrol heat is suppressed between the belligerents so styles run
fully live without constabulary interference. It is the sanctioned alternative to open-world
aggression, which always carries heat.