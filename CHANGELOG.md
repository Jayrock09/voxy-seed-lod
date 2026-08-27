# Rewrite changelog

## rewrite.2

- Fixed the first in-game failure, `Asking for biomes before we have biomes`.
- Changed disposable terrain generation to finish each status across the required region before advancing to the next status.
- Added an extra biome halo for surface rules, carvers, and decoration that query neighboring chunks.
- Added a five-second failed-region retry delay to prevent worker and chat log storms.
- Bumped the prediction schema so output from the broken pipeline is treated as stale.

## rewrite.1

- Replaced sampled terrain reconstruction with disposable real Minecraft world generation.
- Added real noise terrain, aquifers, surface rules, carvers, structure placement, and biome decoration.
- Added real vanilla sky-light and block-light propagation over a temporary halo.
- Removed proxy trees, biome-name surface guesses, smoothing, selective fake features, presets, sample-stride controls, and automatic quality targeting.
- Kept distance-based N-sized Voxy output with complete aligned hierarchy publication.
- Added 8 by 8 distant generation batches to amortize structure and feature halo work.
- Protected real Voxy sections, including empty sections, from later predicted writes.
- Added generator fingerprint invalidation and a real-generation diagnostics HUD.
- Reduced the settings page to enable, distance, high-detail radius, threads, and diagnostics.
- Added scheduler, fingerprint, negative-coordinate, concurrent hierarchy, and saved hierarchy repair tests.
- Verified the patch by applying it to a clean checkout of Voxy commit `337b919d6638cce3d65264efb10b0d20cd060010` and running the full Gradle build.

This is an experimental architecture build. In-game correctness and performance across vanilla and datapack generators still require field testing.
