# Ouroboros SMP: Minestom Backend Game Plan

Status: proposal. Based on a survey of every repo in the `ouroboros-smp` org as of 2026-07-06.

## Where we stand

**This repo** is a clean, current fork of upstream Minestom (MC data `26.1.2`, Java 25, stock
modules only — the only fork-local commits are the AGENTS.md/CLAUDE.md attribution rules).
Minestom is consumed as a *library*: there is no plugin/extension system; you write your own
`main()`, wire events/commands/instances, and ship a single application. It provides instances +
Anvil loading, a generator API, an entity AI framework (goal/target selectors, navigation),
potion effects, Brigadier-style commands, a scheduler, resource-pack push, and Velocity modern
forwarding via `Auth.Velocity(secret)`. It ships **no vanilla gameplay**: no natural mob
spawning, no vanilla mob behavior, no death drops/loot tables/XP, no redstone, no
hoppers/pistons, no fire/explosion block damage, no vanilla worldgen, no RCON, and (since the
permission API was removed) no permissions.

**The org's estate**, sorted by role:

| Repo | Role | Minestom impact |
|---|---|---|
| `mehen-proxy` (Velocity) | Access gate + linking limbo | **Zero changes.** Talks to the backend only via named-server routing in `velocity.toml` plus shared MariaDB/Redis. No plugin messaging channels exist. |
| `mehen-bot`, `mehen-e2e` | Discord governance bot; acceptance tests | Zero changes. `mehen-e2e` (MariaDB + Redis + MC protocol) becomes our acceptance harness for the new backend. |
| `LocalBlindness` | Client-side Fabric mod | Zero changes — no server involvement. |
| `mehen` (Folia) | DB-authoritative ban backstop | Moderate port. Store/bus/Flyway are platform-neutral; only the login hook, kick, and command need Minestom equivalents. Vanilla ban-list import must be dropped. |
| `rooms` | Room gameplay engine | **Best positioned.** Hexagonal: `rooms-core` is platform-neutral behind 6 ports; `settings.gradle` already reserves `rooms-minestom`; SQLite schema deliberately reusable. Adapter ≈ ≤2k LOC. |
| `keepgear` (Folia, ~250 LOC) | Keep hotbar/armor/offhand on death | Trivial logic, but it *configures vanilla death behavior* — on Minestom we own the death system, so this becomes a feature of our own death handler, not a port. |
| `wildanimalbalancer` (Folia, ~560 LOC) | Demand-driven passive-animal top-up | The census/target math ports as-is; but it presumes livestock that wander/graze/breed (vanilla AI) and vanilla biomes. Becomes the spawn-engine brain in our own mob layer. |
| `patrol` (Folia, ~3k LOC) | PvP heat → iron golem enforcers | Core (`patrol.core`) is platform-neutral behind `EnforcerSquadPort`; the adapter is the hardest port in the org: golem chase/attack/patrol relies on vanilla AI + pathfinding, which we must assemble from Minestom's goal-selector/navigation framework. |
| `coffer` (Folia, ~1.2k LOC) | Container ownership/locks | `core` access policy ports directly, but most of its guards protect mechanics Minestom doesn't have (hopper transfer, pistons, explosion block damage, fire). Gated on Phase 2/4 mechanics existing. Storage moves from Bukkit PDC to block NBT/tags. |
| `emojibridge` | Resource pack serve/enforce | Repo is an empty scaffold today. Resource-pack push/enforce is native to Minestom (Adventure), but the planned **RCON reload trigger has no Minestom support** — replace with a Redis pub/sub trigger (mehen-bot already speaks Redis). |

**The core insight:** porting the plugins is the *smaller* half of the job. The bigger half is
that an SMP needs the vanilla survival layer Minestom deliberately omits. Several of our plugins
(keepgear, wildanimalbalancer, half of coffer) exist to *tweak* vanilla behaviors — on Minestom
we own those behaviors outright, so those plugins dissolve into configuration of systems we
build once. That's a real architectural win, paid for by having to build those systems.

## Target architecture

One Gradle multi-module application (working name: `ouroboros-backend`), not a constellation of
plugins:

```
ouroboros-backend/
  app/            main(), config, Auth.Velocity, instance bootstrap, wiring
  governance/     mehen port: ban gate, Redis bus, kick, /mehen command
  permissions/    small DB- or file-backed permission service (LuckPerms has no Minestom support)
  vanilla-layer/  death & drops, XP, item pickup, food/regen, combat (evaluate MinestomPvP),
                  block placement rules, container UX
  mobs/           spawn engine (wildanimalbalancer math) + livestock/golem AI from
                  Minestom goal selectors
  packs/          emojibridge: resource pack push/enforce, Redis-triggered reload
```

Feature repos keep their platform-neutral cores (`rooms-core`, `patrol.core`, coffer `core`,
mehen store/bus) and grow Minestom adapter modules (`rooms-minestom`, etc.), published via
JitPack and consumed by `app`. The Folia plugins keep shipping to prod untouched during the
transition — same cores, two adapters.

**Fork policy:** don't carry patches in this fork yet. Depend on upstream
`net.minestom:minestom` from Maven Central and keep this fork as the escape hatch (JitPack is
already configured) for the day we need to patch internals. Fork maintenance against Minestom's
protocol-update cadence is real ongoing cost; take it only when forced.

**Contracts that must not change:** the MariaDB schema (`bans`, `links`, `players`,
`login_history`, `link_codes`) and Redis channels (`minecraft.ban`, `discord.ban`,
`member.left`, `link.completed`, `evade.suspected`). The proxy and bot are blind to the backend
implementation as long as these hold.

## Phases

### Phase 0 — Walking skeleton (S)
Stand up `ouroboros-backend` with `main()`, Velocity modern forwarding, an Anvil-loaded world,
graceful shutdown, Dockerfile (Java 25), CI. Register it in `velocity.toml` alongside the Folia
server and connect through the proxy. Exit criteria: a linked player can `/server` into it and
walk around the imported world.

### Phase 1 — Governance parity (M)
Nobody plays on a backend the governance system can't police.
- Port `mehen`: ban gate on Minestom's async login/configuration event; direct
  `Player#kick(Component)` (the Folia entity-scheduler indirection collapses); Redis
  subscriptions unchanged; `/mehen` on Minestom's command API; drop the vanilla ban import.
- Build the `permissions/` service (mehen + patrol + coffer all check permission nodes today).
- Build `packs/` (emojibridge, Minestom-native from day one): Adventure resource-pack push,
  status-event enforcement, reload via a new Redis channel instead of RCON.
- Exit criteria: `mehen-e2e` passes against the Minestom backend.

### Phase 2 — Survival baseline (L–XL, the big rock)
Define the **supported-mechanics list** explicitly and build it:
- Death & drops: our own death handler with keepgear semantics built in (kept slots, dropped
  remainder as item entities, XP orbs, keep-level rules).
- Combat/food/regen/potions: evaluate the community MinestomPvP library before writing our own.
- World: import the existing Anvil world; pre-generate expansion chunks with vanilla tooling
  and enforce a world border rather than reimplementing vanilla worldgen.
- Mobs: spawn engine driven by wildanimalbalancer's target/budget math; livestock AI
  (wander/graze/breed) and hostile basics from Minestom goal selectors.
- Explicitly deferred (Phase 4 or never): redstone, hoppers/pistons, brewing, enchanting,
  villagers. Write the list down; player expectations are the risk here.

### Phase 3 — Feature ports (M each, parallelizable once Phase 2 lands)
1. **rooms-minestom** first — cleanest seam, module already reserved. Implement the 6 ports
   (`Scheduler` becomes trivial without region threading), instance↔`WorldId` mapping, block
   change/move events, potion sink, and a `TEXT_DISPLAY` entity for the floating label. Reuse
   the SQLite stores.
2. **patrol** — port the Folia adapter behind `EnforcerSquadPort`: golem spawn, chase/attack
   via goal selectors + Navigator, boss-bar meter, Nether penal instance. Core unchanged.
3. **coffer** — access policy + placement/break/open guards over our own container layer;
   hopper/piston/explosion guards land with whichever of those mechanics we actually build.

### Phase 4 — Cutover & extras
Run both backends behind Velocity, migrate players deliberately, retire Folia when the
supported-mechanics list is accepted. Then decide, deliberately, whether any deferred vanilla
mechanics are worth building.

## Risks
- **Vanilla-parity expectations.** The fatal failure mode is players discovering missing
  mechanics one by one. The Phase 2 supported-mechanics list is the mitigation — publish it.
- **Protocol churn.** Minestom tracks new MC versions on its own schedule; budget for update
  work each MC release, more if we start carrying fork patches.
- **Patrol AI quality.** Golem hunt behavior is the most gameplay-sensitive reimplementation;
  prototype it early in Phase 2's mob work, not at port time.
- **emojibridge is unwritten.** Building it Minestom-native (Phase 1) avoids porting it later
  and dodges the RCON gap entirely — but confirm mehen-bot's trigger side moves to Redis.
