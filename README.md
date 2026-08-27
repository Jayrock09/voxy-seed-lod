> [!WARNING]
> **AI USE DISCLOSURE:** Generative AI was used heavily to design, implement, debug, document, and iterate on this experimental mod. Some of it may therefore be genuine AI slop. I would be more than happy for an organic life form to audit it, clean it up, or remake the idea without the slop. Human-written fixes and clean-room reimplementations are extremely welcome.

# Voxy Seed LOD Rewrite

This branch is a clean experimental rewrite of Seed LOD for [Voxy](https://github.com/MCRcortex/voxy).

Its goal is deliberately narrow:

> Generate the real seed-backed terrain, vegetation, structures, and lighting that Voxy needs to render a distant horizon, while avoiding the gameplay and persistence cost of loading ordinary Minecraft chunks.

The old hand-written terrain painter, proxy trees, biome-name guesses, smoothing system, sample-stride cockpit, and selective fake-feature fallbacks are not part of this rewrite.

## Important upstream notice

Voxy is created by MCRcortex and is marked **All Rights Reserved / Do not redistribute**. This repository does not contain Voxy source code or a compiled Voxy JAR. It contains an original patch, documentation, and instructions for applying the experiment to an authorized local checkout.

This project is unofficial, unsupported by MCRcortex, and not affiliated with Mojang or Microsoft. Do not report rewrite problems to upstream Voxy.

## What the rewrite actually generates

Temporary `ProtoChunk` regions run the active integrated server's real generation code:

1. Structure starts across Minecraft's required structure halo.
2. Biome assignment from the active biome source.
3. Noise terrain and aquifers.
4. Surface rules.
5. Carvers.
6. Structure references and structure block placement.
7. Full biome decoration, including real trees and visible datapack features.
8. Minecraft's real synchronous sky-light and block-light propagation.
9. Voxy extraction and mipping.
10. Immediate disposal of the temporary region.

This means vanilla and compatible datapack content uses its actual generator rules. The rewrite does not infer an oak from a biome name or replace a structure with a silhouette.

## What it intentionally skips

The temporary chunks are never exposed through the live server chunk map. They receive no tickets and are never loaded as gameplay chunks.

The rewrite also skips:

- Chunk saves and region-file persistence.
- Network chunk packets and client chunk objects.
- Entity creation, mob spawning, and entity ticking.
- Points of interest.
- Scheduled block and fluid ticks.
- Spawn-stage generation.
- Block-entity runtime behavior and loot initialization.
- Advancement, map, and other gameplay bookkeeping.

Visible block states placed by structures and features are extracted. Runtime data behind chests, signs, mobs, and similar gameplay objects is discarded.

## N-sized output

N-sized generation remains, but it is now an honest output optimization.

- The high-detail radius publishes one-block Voxy input.
- The next distance bands publish Voxy level 2, level 3, and level 4 cells.
- Every coarse work unit is complete and aligned before publication.
- Coarse parents remain valid while finer children replace them.
- Real loaded chunks are authoritative, including completely empty real sections.

The distant path uses 8 by 8 disposable generation batches instead of repeatedly creating 4 by 4 regions. This amortizes structure-halo, feature-border, lighting, and setup work. Near the player, 4 by 4 batches keep refinement responsive.

N-sized extraction reduces conversion, Voxy storage, hierarchy updates, meshing, and rendered geometry. It does not pretend that Minecraft can produce exact structures and vegetation without executing the generation stages that decide them.

## Performance expectations

The intended advantage over ordinary chunk loading comes from everything the rewrite does not do: no chunk-map integration, disk save, networking, entity systems, POI work, persistent light storage, client chunks, or retained full-resolution faraway chunks.

There is an unavoidable tradeoff. Exact terrain, structures, decoration, and lighting cost much more CPU than the older approximate seed sampler. At very large distances, Minecraft still has to decide every real block before the rewrite can safely call the result real. N-sized output helps substantially after generation, but it cannot make full decoration free.

The branch currently has compile-time and automated-test validation. It does not yet claim a measured speedup on every CPU or datapack. Vanilla, Tectonic, Terralith, and other generator benchmarks are required before making numerical claims.

## Settings

The rewrite intentionally exposes only a small Seed LOD page:

- **Enabled:** Turns disposable real generation on or off.
- **Distance:** 32 to 1024 chunks.
- **High-detail radius:** 32 to 256 chunks before N-sized output steps down.
- **Threads:** Background disposable world-generation workers.
- **Debug HUD and map:** Shows throughput, stage timings, queue state, hierarchy repair, real replacement, and estimated temporary memory.

Use `/voxy seedlod stats` for the same diagnostic information in chat.

## Safety and cache behavior

Predicted mapping IDs carry a private marker. New predictions may refresh old predictions, but they cannot overwrite sections known to come from real chunks. When a real chunk arrives, it replaces predicted Voxy data and repairs the parent hierarchy.

A generator fingerprint covers the seed, prediction schema, generator type, biome source, height, sea level, and serialized generator settings. A mismatch causes predictions to be refreshed instead of silently presenting terrain from an old datapack setup.

## Current validation

The rewrite branch currently passes:

- Full Gradle build for Minecraft 26.2.
- Generator-fingerprint tests.
- Distance-to-N-size scheduling tests.
- Coarse-parent and fine-child hierarchy tests.
- Negative-coordinate hierarchy tests.
- Concurrent sibling publication tests.
- Saved hierarchy repair tests.

The build succeeding does not replace in-game validation. Structure-border behavior, complicated datapack decoration, lighting boundaries, memory peaks, and real throughput still need field testing.

## Apply the rewrite patch

Start from Voxy commit `337b919d6638cce3d65264efb10b0d20cd060010`, create your own local branch, then apply:

```bash
git am /path/to/voxy-seed-lod-rewrite-mc26.2.patch
```

The rewrite patch is in [`patches/voxy-seed-lod-rewrite-mc26.2.patch`](patches/voxy-seed-lod-rewrite-mc26.2.patch).

Build with the JDK and Gradle configuration required by upstream Voxy:

```bash
./gradlew build
```

Windows:

```powershell
.\gradlew.bat build
```

## Design contract

The full enhanced implementation prompt and architecture contract are in [`REWRITE_SPEC.md`](REWRITE_SPEC.md).

The short version is this: let Minecraft generate the visible world content for real, discard everything that exists only for gameplay, and never silently fall back to fake terrain or fake vegetation.
