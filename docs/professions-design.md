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

Twenty styles from the whiteboard. Armor rating is a **certification**: **N**one / **L**ight /
**M**edium / **H**eavy — wear classes above your cert and the style's bonuses switch off (plus
movement/stamina penalties). Armor class → attribute-modifier packages we define, since armor
values are ours (`ARMOR`, `ARMOR_TOUGHNESS`, `MOVEMENT_SPEED`, `KNOCKBACK_RESISTANCE`).

| Style | Armor | Core mechanics |
|---|---|---|
| Baller | N/L | Throws paper, fire charges, slime balls, snowballs, wind charges, bricks, clay, pearls, potions, poisonous potatoes, eggs |
| Pig Rider | L/M | Law enforcement: billy club + shield, rides pigs, knockout/restrain/transport to jail |
| Pirate (based) | N/L | Fishing-rod displacement, special casts; trident secondary |
| Scout | N/L | Move speed, dodge roll, vault, ping |
| Spearman | N/L | Short spear + shield or long spear alone; reach and flexibility |
| Swordman | N/M | Sword + shield; balance |
| Berserker | L/M | Axes + swords; high damage, counters shields |
| Bowman | N/L | Range, mobility, bow attack speed; fast daggers |
| Crossbowman | M/H | Higher damage, armor pen, rockets; fast daggers |
| Mobile Wall | M/H | Tank; ally buff aura while shield raised; armored-horse trample |
| Mounted Archer | N/L | Draw speed while moving; light mounted spear use |
| Spear Cavalry | L/M | Mounted charges, momentum damage |
| Juggernaut | M/H | Mace + great axe; tanky and slow in heavy |
| Assassin | N/L | Stealth; poisoned daggers; dagger/crossbow sneak attacks |
| Duelist | N/L | Opening-slash ability; single sword; speed and damage |
| Mounted Scout | N/L | Swiftest rider; horse-handling abilities |
| Slayer | N/L | Attack scales inversely with defense and health; great axe |
| Hoplite | L/M | Long spear **with** shield; mobile reach fighter |
| Pioneer | N/L | Scout+; clears exhaustion; harder for mobs to notice |
| Valkyrie | L/M | Mace specialist; high jump |

**Suggested organization** (keeps the flat list, adds a prerequisite skeleton — natural
clusters are visible on the whiteboard): base styles (Scout, Spearman, Swordman, Berserker,
Bowman, Baller, Pirate) branch into advanced ones (Scout → Pioneer / Mounted Scout; Spearman →
Hoplite / Spear Cavalry; Swordman → Duelist / Mobile Wall; Berserker → Slayer / Juggernaut /
Valkyrie; Bowman → Mounted Archer / Crossbowman; Pig Rider and Assassin as gated entries —
Pig Rider via governance standing, Assassin via infamy). Open question for the group.

### Mechanics → engine mapping (all verified available)

- **Reach (Spearman/Hoplite):** `ENTITY_INTERACTION_RANGE` / `BLOCK_INTERACTION_RANGE`
  attributes set per player — real per-player reach; hit validation is server-side anyway
  since the damage pipeline is ours.
- **Stealth (Assassin) / mob-notice (Pioneer):** `Entity.updateViewableRule(Predicate<Player>)`
  — per-viewer visibility, so a stealthed Assassin is *absent from chosen clients*, not
  potion-invisible. Mob perception is a parameter of **our** AI target selectors, so
  Pioneer's "harder for mobs to notice" is a multiplier in our own code — trivial here,
  impossible in vanilla.
- **Mounts (Pig Rider, cavalry, Mounted Wall trample):** full passenger/vehicle API
  (`addPassenger`/`getVehicle`); saddled steering for pig/horse; restrain-and-transport =
  the restrained player as passenger on the pig. Charge/trample damage = mount velocity in
  our damage formula.
- **Knockout/restrain (Pig Rider):** unconscious = custom state we own (immobilize via
  attributes, screen effects, interaction lockout). Jail feeds the existing justice stack —
  Pig Rider is the *player-driven* complement to Patrol's golem constabulary, warrants come
  from Patrol heat, bans/records from Mehen.
- **Dodge roll/vault (Scout), high jump (Valkyrie):** velocity impulses + brief i-frames in
  our damage pipeline; `JUMP_STRENGTH` + `SAFE_FALL_DISTANCE` attributes.
- **Projectiles (Baller, Pirate, Crossbowman rockets):** projectile behavior is fully custom
  (Minestom ships none) — throwing bricks and poisonous potatoes costs the same as snowballs.
  Fishing-rod displacement = our hook entity + velocity pulls.
- **Shield mechanics (Swordman/Mobile Wall/Berserker counter):** the blocking model is ours —
  shield-raise events, damage reduction, axe shield-break, and the Mobile Wall aura (AoE ally
  effects while blocking) are all rules in our combat layer.
- **Conditional stats (Slayer, Bowman/Crossbowman daggers, Mounted Archer):** dynamic
  attribute modifiers recomputed on equip/health/mount-state change — cheap server-side
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
  playable slice: 3–4 base styles (Swordman, Bowman, Scout, Spearman) + Laborer/Blacksmith
  loop to prove the economy.
- **Phase 3+:** mounted styles (need the mount layer), Assassin/Pioneer (need AI perception),
  Pig Rider (needs jail/justice wiring), Enchanter (needs the magic framework).

## Open questions

1. Prerequisite skeleton for styles (cluster proposal above) — or keep all 20 flat-entry?
2. Point budgets: exact numbers for "master one + dabble" in each track.
3. Respec friction: free with cooldown, resource cost, or XP-preserving surrender only?
4. PvP scope: are styles always-on, or arena/war-declared contexts only (interacts with
   Patrol heat)?
5. Pig Rider entry: governance-gated (Mehen standing / Discord role) or open?
