# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Fabric client mod ("Projectile Trajectory Preview", mod id `ptp`, package `fr.madu59.ptp`) that renders the predicted trajectory and impact zone of projectiles (bow, crossbow, trident, snowball, pearls, eggs, potions, exp bottles, fishing rod). Targets Minecraft 26.3, Java 25, Fabric Loom (`net.fabricmc.fabric-loom`). Versions live in `gradle.properties`.

## Commands

- `./gradlew build` — compile and produce jars in `build/libs/` (this is also what CI runs; there is no test suite or linter).
- `./gradlew runClient` — launch a dev Minecraft client (uses the `run/` directory).

## Architecture

Uses Loom's `splitEnvironmentSourceSets()`: almost all code is in `src/client/java`; `src/main/java` only holds the server-side entrypoint (`Ptp`) and `HandshakeNetworking`. Entrypoints are declared in `src/main/resources/fabric.mod.json` (main, client, modmenu).

Prediction pipeline (client side):

1. **Registration** — `registry/ProjectileRegistry.init()` registers each vanilla item with `ProjectileDataAPI.registerProjectile(item, provider, enabledSupplier)`. The supplier reads a toggle from `SettingsManager`. Adding a new vanilla item means adding a line here plus a toggle option.
2. **Providers** — `projectiles/VanillaProjectiles` holds the provider functions `(ItemStack, Player, List<ProjectileData> out)`. Each computes start position, velocity, offset and physics constants and appends one or more `ProjectileData` (e.g. multishot crossbow adds several). Many items share a provider (snowball/pearl/egg all use `snowballTrajectory`).
3. **API** — `api/projectiles/ProjectileDataAPI` is the public extension point for other mods (map of id → provider, per-id enabled supplier, blacklist). Public methods carry `@since` tags; keep them backward compatible.
4. **Simulation** — `physics/ProjectileData` describes gravity, drag, water drag, and a `PhysicsOrder` (`TrajectoryUtils.ORDER_PDG` / `ORDER_GPD`: the order in which position, drag and gravity are applied per tick differs per projectile type and must match vanilla). `PreviewImpact` is the result.
5. **Rendering/loop** — `PtpClient` hooks `ClientTickEvents.END_CLIENT_TICK` and `LevelRenderEvents.AFTER_SOLID_FEATURES`; `util/RenderUtils` draws the line/outline/highlight, `util/TrajectoryUtils` has aim/view helpers.

Other pieces:
- `config/SettingsManager` + `config/Option` define all settings (persisted to `config/ptp.json`, also editable via `/ptpConfig setting value` and the Mod Menu screen in `config/configscreen/`). New settings need entries in `SettingsManager` and every `assets/ptp/lang/*.json` (en_us is the base).
- `compat/` — optional Vivecraft integration (compile-only dependency; guard with `ModCompat.isVivecraftLoaded()`), as are Mod Menu.
- The mod intentionally does nothing on servers unless also installed server-side (anti-PVP-cheat handshake; the client-side networking hookup in `PtpClient` is currently commented out).

## Release workflow

Pushes to `main` trigger `.github/workflows/release.yml`, which reads `mod_version` from `gradle.properties` and publishes a GitHub release if none exists for that version. Bump `mod_version` and update `CHANGELOG.md` ("What's new:" list) for a release.
