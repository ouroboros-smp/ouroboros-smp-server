# Phase 0 — Walking Skeleton, in Depth

Companion to `minestom-backend-plan.md`. Everything below is grounded in the actual source of
this Minestom tree (MC 26.1.2, protocol 775, data version 4790, Java 25), `mehen-proxy`,
`mehen`, and `mehen-e2e` as of 2026-07-06.

**Objective:** a `ouroboros-backend` application that boots behind Velocity with modern
forwarding, serves an imported Anvil world, shuts down cleanly, ships as a container, and is
reachable through the existing proxy without touching `mehen-proxy` code.

**Exit criteria (all testable):**
1. `docker compose up` brings up MariaDB, Redis, Velocity (+ mehen-proxy), NanoLimbo, and the
   Minestom backend.
2. A linked, eligible player connects through Velocity, runs `/server minestom-backend`, lands
   in the imported world, and walks around with correct terrain, lighting, and biomes.
3. A wrong forwarding secret produces the expected "Invalid proxy response!" kick (proves the
   forwarding handshake is actually validated, not bypassed).
4. `SIGTERM` on the container saves chunks and disconnects players gracefully.
5. CI builds the app and runs a headless boot smoke test.

---

## 1. Decisions to lock first

**D1 — New repo `ouroboros-backend`, Minestom from Maven Central.** Gradle (Kotlin DSL), Java
25 toolchain (Minestom compiles with `--release 25`; this is separate from the Java 21 plugin
toolchains — different processes, no conflict). Depend on `net.minestom:minestom:<release>`
pinned to a release matching MC 26.1.2 / protocol 775. Do not build against this fork yet;
it exists as the escape hatch (JitPack is configured here if we ever carry patches).

**D2 — Server name.** Register the backend in `velocity.toml` as a *new* entry (working name
`minestom-backend`) alongside `primordial-folia` and `link-limbo`. mehen-proxy's `AccessGate`
routes ELIGIBLE players to `folia-server` (default `primordial-folia`) — we don't change that
in Phase 0. Players reach Minestom via `/server minestom-backend` (requires the Velocity
`velocity.command.server` permission). Cutover later is a one-line config change: point
mehen-proxy's `folia-server` key at the Minestom entry.

**D3 — Auth mode is config-driven.** `Auth.Velocity(secret)` in anything proxied;
`Auth.Offline()` for bare local dev runs. Never expose the backend port publicly — with
Velocity forwarding, anyone who can reach the port and speak the handshake gets kicked, but
the port should still be network-isolated (compose-internal only, no host mapping).

**D4 — Adventure gamemode in Phase 0.** No gameplay systems exist yet; spawning players in
ADVENTURE avoids uncontrolled block edits in the imported world. Survival mode arrives with
Phase 2 mechanics.

**D5 — Backend compression off.** Velocity already compresses (its `compression-threshold =
256`); double compression wastes CPU. Call `MinecraftServer.setCompressionThreshold(0)`
*before* `start()` (it throws once the server is live).

---

## 2. The bootstrap, API-exact

What the current Minestom actually exposes (the demo's commented `VelocityProxy.enable(...)`
is stale API — the real path is the `Auth` sealed interface):

```java
public final class Backend {
    public static void main(String[] args) {
        var config = BackendConfig.load(Path.of("config.yml"));   // §3

        MinecraftServer.setCompressionThreshold(0);                // D5, before start
        MinecraftServer server = MinecraftServer.init(
                config.velocitySecret() != null
                        ? new Auth.Velocity(config.velocitySecret())  // Velocity(String) wraps HmacSHA256 key
                        : new Auth.Offline());
        MinecraftServer.setBrandName("ouroboros");

        // World: current AnvilLoader(Path, Key) expects <path>/dimensions/<ns>/<value>/region.
        // A Paper world dir is <world>/region — see §4 for the import step that restructures it.
        var world = MinecraftServer.getInstanceManager().createInstanceContainer(
                DimensionType.OVERWORLD,
                new AnvilLoader(config.worldPath(), DimensionType.OVERWORLD.key()));
        world.setChunkSupplier(LightingChunk::new);                // without this: no light

        var spawn = config.spawnPos();
        MinecraftServer.getGlobalEventHandler()
                .addListener(AsyncPlayerConfigurationEvent.class, e -> {
                    e.setSpawningInstance(world);                  // MANDATORY or the join hangs
                    e.getPlayer().setRespawnPoint(spawn);
                })
                .addListener(PlayerSpawnEvent.class, e -> {
                    if (e.isFirstSpawn()) e.getPlayer().setGameMode(GameMode.ADVENTURE);
                });

        MinecraftServer.getSchedulerManager().buildShutdownTask(() -> {
            world.saveChunksToStorage();                           // AnvilLoader write path is real
            world.saveInstance();
        });
        // SIGTERM/SIGINT already trigger a clean stop: ServerFlag.SHUTDOWN_ON_SIGNAL defaults true.

        server.start(config.bindHost(), config.bindPort());
    }
}
```

Facts worth knowing behind that sketch:
- `Auth.Velocity` validates a 32-byte HMAC-SHA256 over the forwarded payload and requires
  forwarding version 1; failure ⇒ kick "Invalid proxy response!" (`LoginListener`). The
  login-plugin exchange times out after `minestom.login-plugin-message-timeout` (5 s default).
- `AsyncPlayerConfigurationEvent` runs off the tick threads — blocking there is safe, and
  `Player#kick` is valid. This is exactly where mehen's ban gate lands in Phase 1, so Phase 0
  should already route joins through one `PlayerJoinPipeline` class we can extend.
- There is **no built-in whitelist, ban list, op system, or enforced permissions** — only the
  cosmetic `Player.setPermissionLevel(0–4)`. Until Phase 1, the proxy's access gate is the
  *only* gate. That is acceptable precisely because the backend port is compose-internal (D3).

## 3. Configuration

Follow the org convention (mehen/mehen-proxy): one `config.yml`, SnakeYAML, no exotic config
framework — with one deviation: **the forwarding secret comes from a file path or env var**,
not inline YAML, so the config can be committed to the ops repo without the secret.

```yaml
bind:
  host: 0.0.0.0
  port: 25565
velocity:
  # one of: secret-file, MINESTOM_VELOCITY_SECRET env; absent => offline (dev only)
  secret-file: /run/secrets/velocity-forwarding.secret
world:
  path: /data/world           # dimensions/minecraft/overworld/region layout, see §4
  spawn: { x: 0.5, y: 65.0, z: 0.5 }
```

Tuning stays on JVM `-D` flags (Minestom's own mechanism, `ServerFlag`): the ones that matter
for an SMP are `minestom.chunk-view-distance` (default 8), `minestom.entity-view-distance`
(5), `minestom.tps` (20), `minestom.keep-alive-kick` (15000). Put them in the Dockerfile's
`JAVA_TOOL_OPTIONS` so ops can override per environment.

## 4. World import — the step with teeth

The current `AnvilLoader(Path, Key)` reads regions from `<path>/dimensions/<namespace>/<value>/region`
(the 26.1+ layout). A Paper world is `<world>/region`. There is a deprecated legacy
constructor that reads the old layout, but it's `forRemoval` — don't build on it. Instead the
import script (one-time, checked into `ouroboros-backend/tools/`):

1. Copy the Paper world dir (server offline or from backup — region files mid-write are torn).
2. Restructure: `region/` → `dimensions/minecraft/overworld/region/`; keep `level.dat` at root.
3. Strip what Minestom will never read (see below): `entities/`, `poi/`, `DIM-1/`, `DIM1/`,
   `playerdata/`, `advancements/`, `stats/`, `datapacks/`.

What loads and what doesn't (verified against `AnvilLoader` in this tree):
- **Loads:** block states, biomes, per-section sky/block light, heightmaps, and **block
  entities** (chests/signs get their NBT attached; behavior needs `BlockHandler`s we don't
  register until Phase 2 — in Phase 0 a chest is scenery).
- **Does not load:** **entities** (no `entities/` region reading at all — no mobs, item
  frames, armor stands, villagers), no POI. Unknown biomes fall back silently to plains.
  Chunks whose status isn't empty/`minecraft:full` are skipped with a warning — pre-generated
  but never-finalized frontier chunks will be holes; run a Paper pre-generation pass first if
  the border area matters.
- Player data does not carry over (Minestom keeps no `playerdata/`); Phase 0 players spawn at
  the configured spawn with empty inventories. Fine for a skeleton; inventory migration is a
  Phase 4 cutover task, not a Phase 0 one.

The nether/end question stays out of Phase 0: one overworld instance only.

## 5. Velocity & fleet wiring

`mehen-e2e` already carries the topology template (`config/velocity/velocity.toml`):

```toml
[servers]
primordial-folia = "folia:25565"
link-limbo       = "nanolimbo:25566"
minestom-backend = "minestom:25565"     # Phase 0 adds this line
```

Required but missing from that template today (flag to whoever owns it):
`player-info-forwarding-mode = "modern"` and `forwarding-secret-file` — without them the
template runs legacy/no forwarding and every Minestom join dies with "Invalid proxy
response!". The same secret file feeds Velocity and the backend's `secret-file` (§3).

Nothing else in the fleet changes: mehen-proxy's routing, NanoLimbo, the `governance` MariaDB
(3306, user `mehen`), Redis (6379), and every Redis channel are untouched. The backend doesn't
even connect to MariaDB/Redis until Phase 1 — Phase 0 has zero infra dependencies beyond the
proxy.

Compose sketch (extends the e2e file, which already scaffolds `velocity` and `nanolimbo`
service blocks as comments):

```yaml
  minestom:
    build: ../ouroboros-backend
    expose: ["25565"]            # compose-internal only — never ports:
    volumes:
      - world-data:/data/world
    secrets: [velocity-forwarding]
    stop_grace_period: 30s       # room for chunk save on SIGTERM
```

## 6. Packaging & CI

**Dockerfile:** two-stage; build with a Gradle + Temurin 25 image, run on a JRE 25 base.
Entrypoint `java $JAVA_TOOL_OPTIONS -jar backend.jar`; use ZGC (`-XX:+UseZGC` — generational
is default in 25) and `-Xmx` sized to the host. Healthcheck: the server-list ping works out of
the box, so a tiny status-ping probe (or even a TCP connect check) against 25565 is enough.

**CI (GitHub Actions), three jobs:**
1. `build` — `./gradlew build` on Temurin 25.
2. `smoke` — headless boot test: `MinecraftServer.init(new Auth.Offline())`, start on an
   ephemeral port, assert a status ping answers, `stopCleanly()`. Runs in plain JUnit, no
   container needed.
3. `integration` — use `net.minestom:testing` (`Env`) for join-flow tests: fake player
   connects, lands in the configured instance, spawn point applied. This is the harness
   Phase 1's ban-gate tests will extend.

**e2e:** add the `minestom` service to `mehen-e2e`'s compose and a first protocol-level test
(its README already plans real MC protocol tests) that handshakes through Velocity to the
backend. That test is the automated form of exit criterion 2.

## 7. Work breakdown

| # | Task | Size | Depends on |
|---|---|---|---|
| 0.1 | Create `ouroboros-backend` repo: Gradle, Java 25, pinned Minestom, CI `build` job | S | — |
| 0.2 | `main()` bootstrap: Auth wiring, instance, join pipeline, shutdown save | S | 0.1 |
| 0.3 | `config.yml` loader + secret-file/env resolution | S | 0.1 |
| 0.4 | World import script + document the layout restructure | S | — |
| 0.5 | Dockerfile + compose service + secret plumbing | S | 0.2, 0.3 |
| 0.6 | Velocity: add server entry; fix e2e `velocity.toml` template (modern forwarding) | XS | — |
| 0.7 | Smoke + `testing`-based join tests in CI | M | 0.2 |
| 0.8 | e2e: minestom service + first through-the-proxy protocol test | M | 0.5, 0.6 |

Rough total: one focused week. 0.7/0.8 are where the time goes and where the value compounds —
every later phase lands on that harness.

## 8. Phase 0 non-goals (so nobody scope-creeps them in)

No MariaDB/Redis connection, no ban gate (Phase 1). No permissions service (Phase 1). No
survival mechanics, block handlers, or mob AI (Phase 2). No nether/end instances. No replacing
NanoLimbo (a Minestom-based limbo is a candidate *after* the skeleton proves the toolchain,
not before). No fork patches — upstream artifact only.
