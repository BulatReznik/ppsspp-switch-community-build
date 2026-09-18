# PPSSPP Switch FlatOut experiment — chat handoff

## Purpose

This branch is an experimental Nintendo Switch build based on the stable
community port `v0.6.5`, updated to upstream PPSSPP after PR #21715. The only
runtime question left is whether the persistent slowdown in FlatOut: Head On
while driving through water is fixed on real Switch hardware.

This is not a production release and must not replace the stable branch until
the A/B test is completed.

## Original request

- Keep the stable Switch port as the base; do not use the Vulkan preview.
- Bring over the complete upstream viewport/culling/depth rework from PR #21715,
  not only the old `MAX_CULL_CHECK_COUNT` change.
- Preserve required Switch-specific patches and avoid unrelated refactoring.
- Use a pinned upstream commit for reproducibility.
- Produce a working `.nro` and an SD-card ZIP if the toolchain is available.
- Do not push to the original repository or open a PR without permission.
- Prepare a concrete FlatOut A/B test against Switch community build `v0.6.5`.

## Repository findings

- The Switch project is a full fork of PPSSPP, not a PPSSPP submodule or
  subtree. Third-party dependencies are Git submodules.
- Stable Switch `v0.6.5` is based on upstream PPSSPP commit
  `fa50bb1976065c4f8b1b47af227d367fe9771555` (PPSSPP 1.20.4).
- The experimental branch is `experimental/flatout-post-21715`.
- The pinned post-PR upstream commit is
  `8df41f8fb99e7dbe0aab04f9f77cb62e0eb3639f`.
- FlatOut-specific upstream commit
  `5ad24adc93116415bcc99432bbd3e0b5c6c1228a` is an ancestor of that commit.

## Local history

1. `27df0e3989` — merge the complete upstream viewport/culling rework (#21715).
2. `52a82a6470` — mark the build as `0.6.5-exp21715` and require the pinned
   upstream SHA in the release script.
3. `90f4bf1107` — fix two post-merge Switch compilation errors:
   - return `false` from `IsRenderDocLoaded()` on unsupported platforms;
   - bring in upstream's missing `gettimeofday()` initialization in
     `time_to_unix_utc()`.

The upstream merge conflicts were resolved in the CMake configuration,
`GPU/Common/DepthRaster`, `GPU/GLES/ShaderManagerGLES`, and FFmpeg integration.
The stable Switch FFmpeg revision was intentionally kept at
`82049cca2e4c1516ed00a77b502a21f91b7843f4`.

## Switch-specific patches

The existing patches for these submodules apply and compile successfully:

- `ext/aemu_postoffice`
- `ext/glslang`
- `ext/lua`

The build script is idempotent: it applies a patch when absent and accepts it
when already applied.

## Build environment and result

The Windows host already had Docker Desktop. Instead of installing a global
devkitPro/MSYS2 toolchain, the build used the official reproducible image:

```text
devkitpro/devkita64:latest
digest sha256:1fc388c3a0d34bd2045a6dadcb1020e069d5f876a187fd705de14b4440c00282
```

The image contains devkitA64, libnx, SDL2, libpng, CMake, Ninja, `nacptool`, and
`elf2nro`. The complete build linked `PPSSPPSDL.elf` successfully and generated:

```text
build-switch-v0.6.5-exp21715/PPSSPP.nro
dist/v0.6.5-exp21715/package/switch/ppsspp/PPSSPP.nro
dist/v0.6.5-exp21715/PPSSPP-Switch-0.6.5-exp21715.zip
```

Verification results:

- NRO size: `25,853,736` bytes.
- Packaged assets: `190` files.
- ZIP files: `191` (`PPSSPP.nro` plus assets).
- Embedded version: `0.6.5-exp21715`.
- Embedded runtime path: `/switch/ppsspp/`.
- Experimental ZIP SHA-256:
  `c0b77c8c46b65efaef84130bd740fb4782f1a79348350863201c0b76d54e1d0b`.
- Both baseline and experimental archives pass `ZipFile.testzip()`.

The stable baseline downloaded from GitHub release `v0.6.5` is:

```text
dist/baseline-v0.6.5/PPSSPP-Switch-0.6.5.zip
SHA-256 78ae639f28e371260f52dccb05dfd820bc81c8086457a251674e0185922c6297
```

## Rebuild on another Windows machine

Install Docker Desktop, start it, clone the fork recursively, check out the
experimental branch, and run from PowerShell:

```powershell
docker pull devkitpro/devkita64:latest
docker run --rm -e JOBS=2 -v "${PWD}:/workspace" -w /workspace devkitpro/devkita64:latest bash -c "git config --global --add safe.directory /workspace; exec bash ./scripts/build-switch-release.sh"
```

The first build downloads submodules and compiles the pinned FFmpeg, so it is
substantially slower than an incremental rebuild.

On a Windows bind mount, Linux symlinks and executable-bit differences can make
some submodules appear modified. The build also intentionally applies the three
Switch submodule patches listed above. Do not commit those submodule worktree
changes as new submodule SHAs.

## FlatOut A/B test

1. Back up `/switch/ppsspp/`, especially `PSP/SAVEDATA` and `PSP/SYSTEM`.
2. Test stable `PPSSPP-Switch-0.6.5.zip` first.
3. Fully exit PPSSPP, then replace `PPSSPP.nro` and `assets` with the
   experimental package. Keep the same `PSP` directory and configuration.
4. Launch through title takeover (hold `R` while starting a regular Switch game
   until hbmenu appears), not restricted applet mode.
5. Keep backend, resolution, frameskip, CPU clock, cheats, texture scaling, and
   all performance settings identical.
6. Enable `Show FPS Counter`, `Show Speed`, and the `Draw Frametimes Graph`
   debug overlay.
7. Use the same car, track, camera, save state, and water section. Do one warm-up
   pass and record at least two subsequent passes for each build.

The main metric is sustained emulation speed, not only displayed FPS. A shader
compilation hitch is brief and usually disappears on the second pass. The
reported FlatOut bug is a repeatable, sustained slowdown whenever water is in
the scene.

If the slowdown remains, repeat once with file logging enabled and the `G3D`
and `FrameBuf` channels at `Info`. Logging should be disabled for the primary
performance comparison because it can add overhead. Collect the video with
overlays, settings screenshots, and `PSP/DUMP/log.txt`.

## Remaining work

- Run the physical Switch A/B test. A successful build alone does not prove the
  FlatOut issue is fixed.
- If confirmed, decide separately whether to prepare a production branch or PR.
- No push or PR to `SirSamael/ppsspp-switch-community-build` was performed while
  creating the experimental build.
