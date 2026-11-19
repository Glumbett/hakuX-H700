# H700 Linux / muOS port plan

**Target:** Anbernic RG40XX H (Allwinner H700, ARM64, Mali-G31 MP2)  
**OS target:** muOS Banana Andromeda  
**Working branch:** `h700-linux-port`  
**Upstream:** [rfandango/hakuX](https://github.com/rfandango/hakuX)

This document tracks the Linux port work. It is a plan, not a claim that a Linux build or a working emulator currently exists.

## Current source baseline

The current source tree is still Android-oriented:

- `native/CMakeLists.txt` builds the native core and an `xboxrun` executable. The core link line currently includes Android's `log` library.
- `native/cmake/app.cmake` builds `hakuxcore` around SDL3, but links Android's `android` and `log` libraries.
- `native/app/vkdriver.c` uses Android property access and optional adrenotools hooks intended for custom Android Vulkan driver loading.
- `native/app/vklayer.c` includes Android system-property headers.
- `native/run/xboxrun.c` is a command-line test frontend, but its comments and logging/property assumptions are Android/adb-specific.
- The FEX fork is a Git submodule at `external/fex`; additional native dependencies are described in `android/README.md` and configured through `NC_DEPS`.

The existing SDL3 frontend and command-line runner are useful starting points. Neither should be treated as a ready-made muOS application.

## Porting principles

1. Keep the Android build working. Linux-specific code must be selected by CMake/platform guards rather than replacing Android behaviour.
2. Keep all experimental work on `h700-linux-port`; do not commit port changes to `master`.
3. Separate platform services (logging, properties/configuration, paths, Vulkan loader/driver selection, input and display) from emulator-core logic.
4. Do not bundle copyrighted Xbox BIOS, firmware, keys, or game images. Users must provide their own legally obtained game dumps and any required data.
5. Do not put an untested binary on the user's SD card. The first device test must be reversible and must not require reflashing muOS.

## Work plan and acceptance criteria

### Phase 0 — Confirm the target runtime and dependency graph

- Record the exact muOS userspace ABI: libc family/version, dynamic loader, kernel, available graphics nodes, Vulkan loader and SDL3 availability.
- Confirm the H700 display path (DRM/KMS versus framebuffer) and controller/input device interfaces.
- Inspect the FEX submodule revision and all native dependency revisions.
- Identify which MaliSX2 Linux interfaces are usable by this H700 userspace. Do not assume Android driver-loading mechanisms apply to Linux.

**Exit condition:** a documented target sysroot/toolchain and a dependency list with Android-only components clearly identified.

### Phase 1 — Platform abstraction and Linux build configuration

- Add a Linux ARM64 CMake configuration without changing Android defaults.
- Replace Android-only logging/property calls behind a small platform interface.
- Disable Android-only adrenotools integration on Linux.
- Build the emulator core and command-line runner against the target Linux sysroot.
- Keep SDL3 frontend work separate until the core/runner configuration is understood.

**Exit condition:** CMake configures and the ARM64 Linux targets compile and link against the chosen sysroot. This does not yet prove GPU rendering or gameplay.

### Phase 2 — Graphics, display and input

- Select and validate the Linux Vulkan loader/driver path on Mali-G31.
- Evaluate MaliSX2 where compatible; retain a clear fallback/diagnostic path if it is not.
- Implement or adapt SDL3 presentation for the RG40XX H display orientation and resolution.
- Map the handheld controls through SDL/game-controller APIs.
- Validate audio and suspend/exit behaviour.

**Exit condition:** a test build can create a Vulkan device, present frames and receive controller input on the target handheld.

### Phase 3 — muOS packaging

- Create a muOS-compatible application directory and launch script.
- Keep configuration, saves, shader/pipeline caches and logs in user-writable locations.
- Add dependency checks and useful failure messages.
- Provide a clean uninstall/rollback path.

**Exit condition:** the application launches and exits cleanly from muOS without modifying system files or requiring an SD-card reflash.

### Phase 4 — Device validation and release

- Test boot, menu, controller mapping, rendering, audio, save/load, game exit and repeated launches.
- Record logs, known issues, performance and thermal behaviour.
- Publish a build only after it has been tested on the RG40XX H.

**Exit condition:** reproducible device test results and a documented installation/rollback procedure.

## Immediate next investigation

Before writing platform code, inspect the exact muOS Banana Andromeda runtime and the current native dependency graph. In particular, establish the available libc/toolchain and graphics interfaces on the RG40XX H. These determine whether a portable ARM64 build is possible and which SDL3/Vulkan route is appropriate.
