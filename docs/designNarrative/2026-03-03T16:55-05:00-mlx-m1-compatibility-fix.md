# Design Narrative: MLX M1 Compatibility Fix

**Date:** 2026-03-03
**Participants:** User + Claude
**Scope:** Build and runtime failures after pulling latest exo codebase on M1 Max hardware

---

## Context

After pulling the newest exo codebase, running `uv run exo` produced a cascade of build and runtime failures. All of them traced back to a recent change in how MLX (Apple's ML framework) is sourced: the project had switched from pre-built PyPI wheels to a custom git fork that builds MLX from source. This fork fixes GPU lock issues during RDMA operations, but it introduced hard dependencies on build tooling and GPU features that not all Apple Silicon machines have.

What followed was a chain of five related issues, each one uncovered after resolving the previous.

---

## Issue 1: Merge Conflict in placement_utils.py

**What happened:** `src/exo/master/placement_utils.py` had an unresolved merge conflict. The upstream version introduced a `ring: bool` parameter that controls network interface priority ordering -- thunderbolt-first for ring topology, ethernet-first for RDMA. The stashed (local) version had a single priority dictionary without that distinction.

**Decision:** We kept the upstream version's dual-priority structure since it properly uses the function's `ring` parameter. However, the user specifically wanted thunderbolt set to priority 0 in both branches (ring and non-ring), not just the ring case. The rationale is that on this particular cluster, thunderbolt is always the preferred interconnect regardless of topology.

---

## Issue 2: Metal Compiler Not Found

**What happened:** The build failed with `xcrun: error: unable to find utility "metal"`. The Metal shader compiler wasn't available.

**Root cause:** Commit `f2be9292` (Feb 17, 2026) switched the project from PyPI MLX wheels (pre-built, no compiler needed) to a custom git fork (`rltakashige/mlx-jaccl-fix-small-recv`) that builds from source. Building from source requires the Metal compiler, which lives inside the full Xcode.app -- it's not included in the Command Line Tools.

**Decision:** Install full Xcode.app and switch the developer directory with `xcode-select`. Even after that, we discovered that the Metal Toolchain component needed a separate download via `xcodebuild -downloadComponent MetalToolchain` (a 704MB download). The same step was also needed on st2, the remote machine in the cluster.

This was a sign of things to come -- the git fork was pulling in substantial toolchain requirements that the PyPI wheels had previously abstracted away.

---

## Issue 3: Runtime bfloat16_t Metal Shader JIT Error

**What happened:** After a successful build, launching a model failed with `unknown type name 'bfloat16_t'` during Metal kernel JIT compilation.

**Root cause:** The custom MLX fork JIT-compiles some Metal kernels at runtime. The `bfloat16_t` type requires Apple GPU family 9 (M3 and newer), but the machine runs an M1 Max (Apple GPU family 7). The standard PyPI MLX distribution ships pre-compiled `.metallib` files that work across all supported hardware generations. The git fork doesn't include those pre-compiled artifacts, so it falls back to JIT compilation, which then fails on older GPUs.

**Decision:** Switch back to PyPI MLX by commenting out the git fork override in `pyproject.toml`. This was the pivotal decision in the session -- we were trading away the GPU lock fixes from the fork in exchange for broad hardware compatibility.

---

## Issue 4: Symbol Mismatch / Missing libmlx.dylib

**What happened:** After switching to PyPI MLX in `pyproject.toml`, model launch failed with `Symbol not found`. The `.so` files in the venv were still the ones compiled from the git fork, and they were trying to load `libmlx.dylib` from a stale Homebrew MLX 0.28.0 installation.

We removed Homebrew MLX (`brew uninstall mlx`), but that just changed the error to `Library not loaded: @rpath/libmlx.dylib` -- the old `.so` had baked-in rpaths pointing to the now-deleted library.

**Decision:** Full venv rebuild. We ran `rm -rf .venv && uv sync` to get a clean environment with PyPI MLX and its bundled `libmlx.dylib`. No stale artifacts, no rpath confusion. This resolved the issue cleanly.

---

## Issue 5: Multi-node Deployment

**What happened:** The same fix needed to be applied on st2, the remote Mac in the cluster.

**Decision:** Rather than manually editing files on each machine, we created a git branch (`fix/use-pypi-mlx-for-m1-compat`) and pushed it so st2 could simply pull the changes. On st2, we checked out the branch and rebuilt the venv with the same `rm -rf .venv && uv sync` procedure.

---

## Files Changed

| File | Change |
|------|--------|
| `pyproject.toml` | Commented out the git fork MLX source override, reverting to PyPI MLX |
| `uv.lock` | Regenerated to resolve PyPI MLX 0.30.6 + mlx-metal 0.30.6 |
| `src/exo/master/placement_utils.py` | Resolved merge conflict; set thunderbolt to priority 0 in both ring and non-ring topologies |

---

## Design Implications

This is a meaningful compatibility decision, not just a quick fix. Here is what we are trading:

**What we lose:** The custom MLX fork (`rltakashige/mlx-jaccl-fix-small-recv`) was introduced specifically to fix GPU lock issues that occur during RDMA operations. By reverting to PyPI MLX, those fixes are no longer present on M1/M2 machines.

**Why that is acceptable for now:**
- Single-node and basic multi-node operation works fine without the fork's patches.
- The GPU lock fixes primarily matter for RDMA and tensor-parallel workloads, which are not the common case on M1/M2 hardware.
- A proper long-term fix would be for the fork to pre-compile `.metallib` files during its build process, the same way upstream MLX does. That would give us the GPU lock fixes without the JIT compilation failures on older hardware.
- M3+ machines could still use the fork if needed, since they have the required GPU features. This could be handled via per-machine configuration in the future.

**Open question:** If the RDMA GPU lock fix is critical for production clusters, we may need to either (a) contribute the fix upstream to MLX proper, (b) modify the fork's build to produce `.metallib` files, or (c) maintain separate dependency configurations for M1/M2 vs M3+ machines.
