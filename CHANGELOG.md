# Changelog

## seedlod.5: current patch

- Added fast early-exit height sampling for vanilla-style noise datapacks.
- Added cached datapack biome profiles and placed-feature hints.
- Added common vanilla-block surface inference for custom biomes.
- Replaced chunk-quadrant tree placement with world-space local-minimum distribution.
- Redesigned round, conifer, and flat-canopy tree silhouettes.
- Added the fast datapack terrain sampling setting.

## seedlod.4

- Replaced hard cliff snapping with continuous slope-aware interpolation.

## seedlod.3

- Added adaptive sample quality.
- Added optional terrain smoothing.
- Added deterministic biome-aware vegetation proxies.

## seedlod.2

- Fixed black daylight rendering by lighting synthetic air.
- Closed terrain undersides with neighbor-aware skirts.
- Removed the transition gap.
- Added prediction markers for safe regeneration.

## seedlod.1

- Initial experimental seed-backed distant terrain sampler.
