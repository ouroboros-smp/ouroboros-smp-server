# Engine Efficiencies, Multithreading, and New Simulations

Companion to `minestom-backend-plan.md` and `phase-0-walking-skeleton.md`. Answers three
questions against the actual source of this tree (MC 26.1.2, Java 25): what engine
efficiencies are available, whether the tick can be multi-threaded, and what it takes to build
Vintage Story-style regional weather and Timberborn-style volumetric water.

---

## 1. Multithreading: the engine is already parallel — it ships with parallelism off

### How a tick actually runs

`TickSchedulerThread` drives `ServerProcessImpl.TickerImpl.tick()` at 20 TPS. Each tick, in
order:

1. **Serial prologue** (on the tick-scheduler thread): global `SchedulerManager` tasks,
   `ConnectionManager.tick` (logins, keep-alives), then every instance's instance-level tick
   sequentially (time/clocks, weather interpolation, world border, `InstanceTickEvent`).
2. **Parallel phase**: `ThreadDispatcher.updateAndAwait` — every loaded chunk is a
   *partition* pinned to one `TickThread`; each thread ticks its chunks and all entities in
   them, then all threads join on a `CountDownLatch` (a hard per-tick barrier).
3. **Serial epilogue**: thread rebalancing (time-boxed), click-callback cleanup, end-of-tick
   scheduler tasks, and the single-threaded packet flush (`PacketViewableUtils.flush`).

The pool size for phase 2 is **`minestom.dispatcher-threads` — default 1**. That is the whole
reason Minestom "ticks on a single thread" today. Setting `-Dminestom.dispatcher-threads=N`
genuinely parallelizes chunk + entity ticking, with correctness machinery that already exists:

- **Acquirable API** (`net.minestom.server.thread.Acquirable`, experimental): each entity is
  pinned to its chunk's tick thread; cross-thread access goes through
  `acquirable.sync(...)`/`lock()`, which locks the owning thread at its cooperative yield
  point. Ownership violations throw when assertions are on or
  `-Dminestom.acquirable-strict=true`.
- jcstress tests cover dispatcher serialization; JMH covers acquire contention.

### What we should do

**Adopt N > 1 early (Phase 0/1), with discipline baked in from the first commit:**
- Run dev/CI with `-ea` and `-Dminestom.acquirable-strict=true` so ownership bugs surface
  immediately.
- Treat every event listener and `BlockHandler` as multi-thread code: events fired from
  entity/chunk ticks run on the owning `TickThread`, not on one global thread. This is a
  coding standard, not a framework feature — cheap to adopt now, brutal to retrofit.
- Cross-entity interactions (e.g. patrol golems targeting players in another chunk) use
  `Acquirable`, mirroring the discipline the Folia plugins already practice with region
  schedulers.

**Known limits (and why they're acceptable):**
- Chunk→thread assignment is **round-robin by a counter, no spatial locality, no runtime
  migration** (`ThreadProvider.counter()`, `RefreshType.NEVER`, hardwired in
  `ServerProcessImpl`). Fine at SMP scale; see fork patch F1 below for the fix.
- The serial prologue/epilogue and per-tick barrier remain: this is *parallel-within-a-tick*,
  not Folia-style independent region schedulers. True region scheduling would require
  removing the global barrier, parallelizing the control plane (connection tick, instance
  tick, packet flush, global scheduler), and replacing the global lock inside
  `AcquirableImpl.enter`. **Recommendation: don't.** For our population, parallel chunk
  ticking plus off-thread simulations covers the need at a fraction of the risk.
- The most valuable concurrency is free and needs no engine change: **simulations run on
  their own worker threads** (double-buffered), synchronizing results into the world once per
  tick. Both simulations below are designed that way.

### Fork patches worth carrying (first real uses of the fork — both upstream candidates)

- **F1 — Injectable `ThreadProvider`.** `ServerProcessImpl` hardwires the round-robin
  counter. Small patch: allow supplying a spatially-aware provider (group neighboring chunks
  on one thread, `RefreshType.ALWAYS` for migration). Better cache locality, fewer
  cross-thread entity interactions.
- **F2 — Auto-batched block updates.** `Instance.setBlock` sends **one `BlockChangePacket`
  per block, immediately**; `MultiBlockChangePacket` exists in the protocol registry but the
  engine never uses it. Patch: aggregate dirty blocks per section per tick and flush as
  multi-block packets. Vital once any simulation writes blocks at volume (see §3), and a
  general win.
- **F3 — Biome-update API** (optional). `ChunkBiomesPacket` (biomes-only chunk update) is
  likewise registered but never sent; a small `Instance`/`Chunk` API to rewrite biome
  palettes and push that packet serves the weather system's biome swaps (§2). Can live
  app-side initially — the packet record is public.

### Other efficiencies (no fork needed)

- **JVM:** Java 25, generational ZGC. GraalVM reachability metadata is maintained upstream if
  native-image ever becomes interesting (skip for now — JIT throughput wins for a long-lived
  server).
- **Tuning flags:** `minestom.chunk-view-distance` (8), `minestom.entity-view-distance` (5),
  `minestom.entity-synchronization-ticks` (20) are the big packet/CPU levers.
- **Benchmark harness:** the tree ships JMH benches for palette get/set/scan and light
  compute — the two proven hot paths. Every fork patch and simulation prototype should land
  with numbers from these.

---

## 2. Vintage Story-style weather: yes, with zero engine changes

Client weather is two floats — rain level and thunder level — carried by
`ChangeGameStatePacket`. `Instance.setWeather` merely lerps those values and **broadcasts to
all instance viewers**; nothing binds the packet to an instance. Send the same packets
per-player and two players in one world stand in different weather.

Architecture:

- **Climate field, server-side:** temperature/humidity noise + altitude + season, sampled
  continuously. Runs off-thread; per-player weather pushed on movement across climate cells
  and as fronts drift. (One global rain state per client — regionality is entirely
  server-computed, recomputed per position.)
- **Seasons on the multi-clock system:** instances now carry multiple named `WorldClock`s
  with per-clock `rate()` and pause — day length and a season calendar layer on `time()` /
  `worldAge` directly. `SetTimePacket` is also per-player-sendable.
- **Zone visuals via runtime biomes:** biomes are runtime-registerable
  (`DynamicRegistry`) with per-biome fog/water/grass/foliage colors at 4-block resolution.
  Climate zones can *look* different, not just rain differently.
- **Texture:** `ParticlePacket`/sound for flurries, drizzle, heat shimmer near the player.
- **Gameplay hooks** (temperature effects, crop response) are app-layer — and pair naturally
  with rooms (a SKYLIT room reacting to real precipitation; an enclosed FINE room sheltering
  from a cold snap).

### Decoupling precipitation from vanilla biomes

Precipitation *type* is decided client-side, per column, from the biome data the client was
sent: the client renders precipitation only while its weather state is "raining", then checks
the column's biome — `has_precipitation`? `temperature` below 0.15 after altitude adjustment?
Cold → snow, warm → rain, no-precipitation → nothing. In vanilla those inputs are baked into
worldgen; here every one of them is server-authoritative data. Minestom's `Biome`
(`world/biome/Biome.java`) is a plain record — `hasPrecipitation`, `temperature`,
`temperatureModifier`, `downfall`, plus visual `effects` — so weather is *not* attached to
what a biome "really is". The wire biome is a costume.

**Snow (or rain) anywhere — the biome-swap presentation layer:**
- Register cold variants of any biome: identical colors (the `Biome.builder(existing)` copy
  constructor preserves effects), `temperature` below the snow threshold. When the climate
  sim rolls a cold front over a region, rewrite affected chunks' biome palettes to the cold
  variant, turn rain on for players there → snow falls over plains. Front passes, swap back.
- **`ChunkBiomesPacket` exists** (the biomes-only update packet) so a swap needs no full
  chunk resend — but, like `MultiBlockChangePacket`, it is registered in the protocol and
  never sent by the engine. The climate system hand-rolls it (public packet record, a few
  lines), or we land it as fork patch **F3: biome-update API** alongside F2.
- The altitude snow line is free: the client's temperature check is height-adjusted, so one
  moderately-cold variant yields rain in the valley and snow on the peak, per column.
- Snow *accumulation* (layers, melting, ice) is server-side block writes — Phase 2 mechanics
  the climate field simply steers.

**Sandstorms and other non-vanilla weather — composite illusions:** the vanilla client
renders exactly two precipitation types, and the resource-pack rain texture is a single
global texture (retexture it and all rain everywhere becomes sand). So a sandstorm is built
from parts: swap to a custom "duststorm" biome whose `effects` carry sand-colored fog and sky
(instant atmosphere change, most of the perceptual work), keep `has_precipitation: false` so
no rain renders through it, then player-local particle curtains, wind sound loops, and
gameplay bite (slowness, visibility, hunger). Convincing, but it reads as "storm effect,"
not vanilla-crisp falling weather — a client limit shared by every server. The one path to
true custom precipitation rendering is an optional client mod (`LocalBlindness` is the org's
precedent); the server-side composite stays the baseline so vanilla clients get the full
system.

**Design rule this imposes:** once wire-biomes become a runtime costume, no gameplay system
may read the wire biome as truth. WildAnimalBalancer keys species off biome ids today; ported
naively, a cold snap over the savanna would confuse the ecology. The climate system therefore
owns two maps — the *ecological* biome (stable, what the world is) and the *presented* biome
(what the client currently sees). Gameplay reads the first; packets carry the second. Cheap
to enforce from day one, miserable to untangle later.

Sizing: a self-contained Phase 2–3 feature. No fork patches strictly required (F3 optional
quality-of-life).

---

## 3. Timberborn-style volumetric water: buildable — a flagship project with two known traps

There is no fluid behavior to extend: `instance/fluid/` is a static metadata registry; no
flow, no spreading, nothing touches waterlogged semantics. Green field.

**Architecture (what the internals reward):**

- **Truth is a side-channel volume field, not block states.** Water volume per column/cell
  per section in our own arrays. The sim is a shallow-water/heightfield model per column
  (Timberborn's own approach) — far cheaper than full 3D and matching what the client can
  render anyway.
- **The CA runs on dedicated worker threads**, double-buffered (checkerboard or red-black
  passes), with dirty-cell queues and settling: still water sleeps at zero cost. Do *not*
  build on `BlockHandler.tick` — it fires every tick per tickable block with no rate control
  and is the wrong granularity for a field simulation. Hook the flush from the instance tick.
- **Rendering is a projection:** materialize volumes to `water[level=0–7]` states — 8 visible
  levels per block is the hard client fidelity cap. Currents beyond that are server gameplay
  (entity velocity push, wheels, gates).
- **Flush path — this is where the two engine traps live:**
  1. *Never* per-block `Instance.setBlock` (one packet per block, no auto-batching). Use
     `ChunkBatch` (takes the chunk write lock once, applies all changes, resends touched
     sections) or hand-rolled `MultiBlockChangePacket`s — or land fork patch F2 and make the
     whole problem disappear engine-wide.
  2. Every block change invalidates **lighting across a 3×3 chunk neighborhood**
     (`LightingChunk.invalidateNeighborsSection`), and water participates in skylight
     diffusion. Wrap bulk sim writes in `setFreezeInvalidation(true)` and release once per
     flush; light resend is already debounced (`SEND_LIGHT_AFTER_BLOCK_PLACEMENT_DELAY`).
- **Hot-path economics are already instrumented:** palettes are SWAR-optimized with bulk ops
  (`setAll`, `replaceAll`, `copyFrom`, `fill`) and dedicated JMH benches; light compute is
  pooled (work-stealing) and lazily computed on packet request. The prototype should publish
  bench numbers for: cells simulated/ms, sections flushed/tick, light-freeze savings.

**Staging:** prototype post-Phase-2 in a dedicated instance using plain `DynamicChunk` (no
lighting) to isolate sim cost; then integrate with lighting freeze + batched flush; then
gameplay (sources, drains, currents, machines). Sequenced after Phase 2 because survival
mechanics and the batching patch (F2) both feed it.

---

## Recommended order

1. **Phase 0/1:** `dispatcher-threads=N`, strict acquirable assertions in CI, thread-safe
   listener standard. (Free.)
2. **Phase 1–2:** fork patches F1 (thread provider) and F2 (batched block updates), each with
   JMH numbers, each proposed upstream.
3. **Phase 2–3:** climate/weather system (no engine changes).
4. **Post-Phase 2:** volumetric water prototype → integration → gameplay.
