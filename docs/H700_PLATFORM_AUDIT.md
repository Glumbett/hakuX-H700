# H700 Linux port: source and platform audit

**Audited against:** upstream-derived `master` at the time this file was added.  
**Target:** RG40XX H / Allwinner H700 / Mali-G31 MP2 / muOS Banana Andromeda.  
**Status:** source inspection only; no Linux build has been attempted or claimed.

## Findings

### 1. The emulator core is not yet platform-neutral

- `native/core/log.c` already writes to stderr on all platforms and adds Android log output under `__ANDROID__`. This is a good pattern to preserve.
- `hw/xbox/hle/prop.h` defines `hle_prop_get` and `hle_prop_new` only under `__ANDROID__`. Many native files use these debug-property helpers. A Linux implementation or a platform-neutral configuration interface is needed before a Linux build can compile.
- `native/cmake/hle.cmake` links Android libraries (`android`, `log`, `aaudio`) into HLE/APU targets and defines `VK_USE_PLATFORM_ANDROID_KHR` for volk.
- `native/cmake/fex.cmake` links `log` into `nc_fex`; this must be conditional for Linux.
- `native/CMakeLists.txt` links `log` into `hakux_native` and `xboxrun`.
- `native/dev/apu/out.c` is implemented specifically with Android AAudio. Linux audio needs a separate backend (SDL audio or another supported Linux audio API), or an explicit initial no-audio mode that preserves APU timing.

### 2. The current SDL app frontend is Android/JNI-specific

- `native/app/app_main.c` is the SDL entry point for `NativeCoreActivity`, exports JNI methods, includes Android system-property and syscall headers, reads an `ANativeWindow` from SDL Android window properties, and uses Android activity lifecycle assumptions.
- `native/app/app.h` exposes `struct ANativeWindow *` in its presentation interface.
- `native/app/present.c` passes that Android window through the HLE presenter.
- `native/app/vsync.c` uses Android `AChoreographer` and `ALooper`.
- `native/cmake/app.cmake` links the SDL frontend against `android` and `log`, and adds Android-oriented adrenotools hooks when present.

**Implication:** do not try to build the current Android app frontend unchanged on muOS. The clean route is a Linux SDL frontend that shares the existing emulator core and HLE GPU, with Android JNI/window/vsync code kept in the Android target.

### 3. Vulkan presentation needs a Linux surface backend

- The HLE GPU uses Vulkan and its display images are platform-independent offscreen images.
- The present path in `hw/xbox/hle/vk/swapchain.c` is coupled to the Android window/surface path.
- `native/cmake/hle.cmake` currently sets `VK_USE_PLATFORM_ANDROID_KHR` for volk.
- The Android-specific custom driver loader in `native/app/vkdriver.c` uses adrenotools; this is not the Linux MaliSX2 loading mechanism.

**Implication:** preserve the HLE's offscreen display-image logic, but introduce a platform presentation interface for creating a Vulkan surface, swapchain and timing source. For H700, investigate the actual muOS graphics stack and MaliSX2's Linux support before choosing DRM/KMS, SDL Vulkan windowing, or framebuffer output. Do not assume an Android `ANativeWindow` path or Turnip/adrenotools applies.

### 4. Existing command-line runner is useful but not a complete solution

- `native/run/xboxrun.c` boots the native core from a disc image and contains useful diagnostics, capture and timeout handling.
- It includes Android-oriented property and logging assumptions through the Vulkan driver path and comments.
- It is not a standalone muOS GUI, controller frontend, audio backend or proven Linux Vulkan presenter.
- `tools/native/xboxrun.sh` is an Android `/system/bin/sh` helper using `pidof`, `dumpsys` and `/data/local/tmp`; it is not a muOS launcher.

**Implication:** use the runner as a reference and possible headless diagnostic target, not as proof that the Linux app is nearly finished.

## Proposed source layout

Keep platform implementations explicit:

- `native/platform/common/`: shared configuration, paths and logging interfaces where genuinely cross-platform.
- `native/platform/android/`: existing Android property, AAudio, Choreographer, JNI and Android Vulkan-window implementation.
- `native/platform/linux/`: Linux config/property shim, Linux audio backend, Linux timing/presentation integration and muOS paths.
- `native/app/linux/`: SDL3 Linux frontend and controller/event loop.
- `native/cmake/`: platform-selectable CMake targets; Android remains the default supported build until Linux parity is demonstrated.

This is a proposed layout, not a committed code reorganisation.

## Next engineering sequence

1. Enumerate every use of `hle_prop_get`, `hle_prop_new`, `PROP_VALUE_MAX`, Android headers and Android-only link libraries.
2. Decide the Linux config contract (environment variables plus a config file is a reasonable initial approach; no persistent Android-style property service should be assumed).
3. Add platform-selectable CMake dependencies and prove the Android target remains unchanged.
4. Add a minimal Linux compile target using a real muOS-compatible sysroot/toolchain.
5. Only then implement display/surface and controller support.

## Device information still needed

A build must target the actual muOS userspace ABI. Before selecting the final sysroot, record from the RG40XX H:

- `uname -a`
- `getconf GNU_LIBC_VERSION` (if available)
- `ldd --version` (or the device's libc identification)
- `ls -l /dev/dri /dev/fb* /dev/input` (missing paths are informative)
- `vulkaninfo --summary` (if installed) or the available Vulkan loader/driver details
- whether SSH or a muOS terminal is available

Do not run package installation, change system files, or reflash the SD card for this inspection. Read-only command output is sufficient.
