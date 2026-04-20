<!--
  Keywords (for GitHub/Google indexing):
  android, android-ndk, aosp, a_native_window_creator.h, anativewindow,
  libgui, libutils, surfaceflinger, surface-composer, dlsym, dlopen,
  itanium-abi, c++ name mangling, symbol patching, mirrorSurface,
  android 15, android 16, imgui overlay, native window, private ndk symbols
-->

# android-native-window-symbol-patcher

> Keep **`a_native_window_creator.h`** alive across Android version upgrades.
> A maintained header + a small Python tool that find and patch private
> **`libgui.so` / `libutils.so`** symbols (`dlsym` targets) when Google changes
> their C++ mangling between releases — so one build keeps working on
> **Android 11 → 12 → 13 → 14 → 15 → 16**.

<p>
  <a href="TUTORIAL.md"><img alt="Tutorial" src="https://img.shields.io/badge/docs-TUTORIAL.md-2ea44f?style=flat-square"></a>
  <a href="JOURNAL.md"><img alt="Journal" src="https://img.shields.io/badge/docs-JOURNAL.md-8a5cf6?style=flat-square"></a>
  <img alt="Android" src="https://img.shields.io/badge/Android-11%E2%80%9316-3DDC84?style=flat-square&logo=android&logoColor=white">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.8%2B-3776AB?style=flat-square&logo=python&logoColor=white">
  <img alt="C%2B%2B" src="https://img.shields.io/badge/C%2B%2B-17-00599C?style=flat-square&logo=cplusplus&logoColor=white">
</p>

> **NOTE:** <span style="color:red">This project was written by my friend to solve this problem on a real shipping app. The complete open-source source is [on GitHub](https://github.com/mohamad-aljeiawi/cping/tree/main/cping-memory-pubg).</span>

---

## Why this exists

If you ship an Android app that builds its own native overlay window (ImGui,
game menus, debug HUDs, screen mirrors, recorders…) you are almost certainly
calling **private C++ functions** inside `libgui.so` and `libutils.so` via
`dlsym()`. These functions are **not stable NDK APIs**: every Android release
Google can rename them, add a parameter, or move them into another namespace.

When that happens your app crashes on boot on the newest Android version with:

```
libc++abi: terminating due to uncaught exception of type St13runtime_error
  what(): Failed to resolve symbol:
  _ZN7android21SurfaceComposerClient13mirrorSurfaceEPNS_14SurfaceControlE
Aborted
```

This repo solves that problem in two complementary pieces:

- **[`a_native_window_creator.h`](a_native_window_creator.h)** — a
  production, drop-in header that already knows about Android 11…16 deltas,
  uses version-gated symbol descriptors, and keeps older devices working
  after every patch.
- **[`symbol_patcher.py`](symbol_patcher.py)** — a zero-dependency Python
  tool that, given the failing mangled name, scans the device's system
  libraries, picks the best replacement mangling, and rewrites the header
  with the correct 5-step patch (typedef + slot + invoker + descriptor split
  + call-site version gate).

---

## Quick start

### Requirements

- Android Platform-Tools (`adb` in `PATH`).
- Android NDK (only `llvm-nm` and `llvm-cxxfilt` are used).
- Python 3.8+ (no third-party packages).
- A rooted device **or** emulator for `adb pull /system/lib64/*.so`.

Set `NDK_PATH` once so the tool can find `llvm-nm`:

```powershell
# Windows (PowerShell)
$env:NDK_PATH = "C:\Android\Sdk\ndk\27.0.11902837"
```

```bash
# Linux / macOS
export NDK_PATH="$HOME/Android/Sdk/ndk/27.0.11902837"
```

### 1. Drop the header into your project

Copy `a_native_window_creator.h` into your `jni/include/native/` (or
wherever your existing overlay header lives) and `#include` it as usual.
It already supports Android 11 through Android 16 out of the box.

### 2. You hit a new Android version and it crashes — capture the symbol

```powershell
adb logcat -c
adb shell "su -c '/data/local/tmp/your_binary'"
adb logcat -d | Select-String "failed to resolve"
# -> _ZN7android21SurfaceComposerClient13mirrorSurfaceEPNS_14SurfaceControlE
```

### 3. Let the patcher find and apply the fix

```powershell
# Dry-run: scan device, suggest the new mangling, print a unified diff.
python symbol_patcher.py `
    --header a_native_window_creator.h `
    --old-symbol _ZN7android21SurfaceComposerClient13mirrorSurfaceEPNS_14SurfaceControlE `
    --scan-libs

# Apply it in place (auto-creates a timestamped .bak next to the header):
python symbol_patcher.py `
    --header a_native_window_creator.h `
    --old-symbol _ZN7android21SurfaceComposerClient13mirrorSurfaceEPNS_14SurfaceControlE `
    --scan-libs --apply --min-new-version 15
```

### 4. Already know the new mangling? Skip the device scan

```bash
python symbol_patcher.py \
    --header a_native_window_creator.h \
    --old-symbol _ZN7android21SurfaceComposerClient13mirrorSurfaceEPNS_14SurfaceControlE \
    --new-symbol _ZN7android21SurfaceComposerClient13mirrorSurfaceEPNS_14SurfaceControlES2_ \
    --apply --min-new-version 15
```

That's it. Rebuild, push, run — your app now works on Android 15/16 **and**
still works on every previous version you supported.

---

## Using `a_native_window_creator.h` in `main.cpp`

The header exposes a single, easy-to-use class, **`android::ANativeWindowCreator`**,
with four static entry points:

| Method | What it does |
|---|---|
| `GetDisplayInfo()` | Returns the current display size + rotation (`theta`, `width`, `height`). |
| `Create(options)` | Creates a compositor-backed `ANativeWindow*` you can draw on (EGL, Vulkan, ImGui, raw `ANativeWindow_lock`). |
| `Destroy(window)` | Releases the window and its `SurfaceControl`. |
| `ProcessMirrorDisplay()` | Call once per frame on Android 14+ so your overlay survives screen-recording / casting. |

### Minimal example — raw framebuffer fill

```cpp
// main.cpp
// Compile with the Android NDK. Requires linking against -landroid (for
// ANativeWindow_*), plus dlopen/dlsym (already in libc).

#include <android/native_window.h>
#include <android/native_window_jni.h>
#include <chrono>
#include <cstdint>
#include <cstdio>
#include <thread>

#include "a_native_window_creator.h"

using android::ANativeWindowCreator;

int main() {
    // 1. Query the display so our overlay fills the screen.
    auto display = ANativeWindowCreator::GetDisplayInfo();
    std::printf("display: %dx%d (theta=%d)\n",
                display.width, display.height, display.theta);

    // 2. Create a transparent overlay window.
    ANativeWindow* window = ANativeWindowCreator::Create({
        .name           = "MyOverlay",
        .width          = display.width,
        .height         = display.height,
        .skipScreenshot = true,   // hide from screenshots / screen recorders
    });
    if (!window) {
        std::fprintf(stderr, "ANativeWindowCreator::Create failed\n");
        return 1;
    }

    // 3. Configure the buffer format (RGBA8888).
    ANativeWindow_setBuffersGeometry(
        window, display.width, display.height, WINDOW_FORMAT_RGBA_8888);

    // 4. Render loop — paint a solid translucent red, 60 fps.
    for (int frame = 0; frame < 600; ++frame) {
        ANativeWindow_Buffer buf{};
        if (ANativeWindow_lock(window, &buf, nullptr) == 0) {
            auto* pixels = static_cast<uint32_t*>(buf.bits);
            const uint32_t color = 0x80'00'00'FFu; // ABGR: 50% alpha red
            for (int32_t y = 0; y < buf.height; ++y) {
                for (int32_t x = 0; x < buf.width; ++x) {
                    pixels[y * buf.stride + x] = color;
                }
            }
            ANativeWindow_unlockAndPost(window);
        }

        // Keep the overlay visible under screen-mirror on Android 14+.
        ANativeWindowCreator::ProcessMirrorDisplay();

        std::this_thread::sleep_for(std::chrono::milliseconds(16));
    }

    // 5. Clean up.
    ANativeWindowCreator::Destroy(window);
    return 0;
}
```

### ImGui / overlay skeleton

If you already have an ImGui (or any other renderer) integration, the only
thing you need from this header is the `ANativeWindow*`. Pipe it into your
EGL / Vulkan surface creator exactly as you would with any other Android
native window:

```cpp
#include "a_native_window_creator.h"

int main() {
    auto display = android::ANativeWindowCreator::GetDisplayInfo();

    ANativeWindow* window = android::ANativeWindowCreator::Create({
        .name           = "AImGui",
        .width          = display.width,
        .height         = display.height,
        .skipScreenshot = true,
    });

    // Hand `window` to your existing init code:
    //   EGLSurface surface = eglCreateWindowSurface(display, cfg, window, nullptr);
    //   ImGui_ImplAndroid_Init(window);
    //   ...

    while (running) {
        // your frame:
        //   ImGui_ImplOpenGL3_NewFrame();
        //   ImGui_ImplAndroid_NewFrame();
        //   ImGui::NewFrame();
        //   ...  build UI  ...
        //   ImGui::Render();
        //   eglSwapBuffers(display, surface);

        android::ANativeWindowCreator::ProcessMirrorDisplay(); // Android 14+
    }

    android::ANativeWindowCreator::Destroy(window);
}
```

### Building (typical `Android.mk`)

```make
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE      := my_overlay
LOCAL_SRC_FILES   := main.cpp
LOCAL_C_INCLUDES  := $(LOCAL_PATH)/include/native
LOCAL_CPPFLAGS    := -std=c++17 -fexceptions
LOCAL_LDLIBS      := -landroid -llog -ldl
include $(BUILD_EXECUTABLE)
```

Running on the device:

```powershell
adb push libs/arm64-v8a/my_overlay /data/local/tmp/
adb shell "su -c 'chmod 755 /data/local/tmp/my_overlay && /data/local/tmp/my_overlay'"
```

> **Why root?** The header talks directly to `SurfaceFlinger` via private
> `libgui` symbols. That path requires `system`/`graphics` group access,
> which is only available to root processes or apps signed with a platform
> key. Non-rooted shipping apps can still use this header by packaging the
> binary as a privileged helper or by wrapping it in an accessibility / VPN
> service that the user grants explicitly.

---

## What the patcher does, in one picture

```
  logcat: "failed to resolve _ZN...mirrorSurface...E"
                      │
                      ▼
            ┌──────────────────────┐
            │   symbol_patcher.py  │
            ├──────────────────────┤
            │ 1. adb pull libgui.so│
            │ 2. llvm-nm -D  →  T  │
            │ 3. rank candidates   │
            │ 4. rewrite header:   │
            │    • new typedef     │
            │    • new Api slot    │
            │    • new invoker     │
            │    • split descriptor│
            │    • gate call site  │
            └──────────────────────┘
                      │
                      ▼
           a_native_window_creator.h
                 (patched)
```

Full theory and the manual version of each step are in
**[`TUTORIAL.md`](TUTORIAL.md)**.

---

## Documentation

| File | What it is | Who it's for |
|---|---|---|
| **[`TUTORIAL.md`](TUTORIAL.md)** | Step-by-step guide: how the resolver works, reading Itanium mangled names, the 5-step patch pattern, version-gating rules, cheat sheets. Written for both beginners and experienced reverse engineers. | Anyone patching private Android symbols |
| **[`JOURNAL.md`](JOURNAL.md)** | Real, chronological session log that took `cping-memory-pubg` from "crashing on Android 16" to "clean build + overlay + reusable tooling". Every command, every wrong guess, every correction. | Readers who learn best from case studies |
| **[`a_native_window_creator.h`](a_native_window_creator.h)** | The patched, shipping header. Drop it into your project. | C++ / NDK developers |
| **[`symbol_patcher.py`](symbol_patcher.py)** | Automated patcher (5-step pattern). Run `--help` for full CLI. | Anyone maintaining an overlay across Android upgrades |

---

## Features

- **One header, every Android version** — 11, 12, 13, 14, 15, 16.
- **Version-gated symbol descriptors** — old mangling for old devices, new
  mangling for new devices, enforced at compile time + runtime.
- **Automated patching** — no hand-editing mangled names, no hand-splitting
  descriptor ranges, no hand-writing `reinterpret_cast` branches.
- **Safe by default** — dry-run prints a unified diff; `--apply` creates a
  timestamped `.bak` before writing.
- **No third-party Python deps** — stdlib only; works on Windows, macOS, Linux.
- **Teaches, does not hide** — the tutorial explains *why* each patch step
  exists, so you can fix an unusual case by hand when the tool can't.

---

## Full CLI

```text
python symbol_patcher.py --help
```

Common flags:

| Flag | Meaning |
|---|---|
| `--header PATH` | Header to patch (usually `a_native_window_creator.h`). |
| `--old-symbol NAME` | Mangled name `dlsym()` failed on (from logcat). |
| `--new-symbol NAME` | Optional: skip device scan and use this mangling. |
| `--scan-libs` | Pull `libgui.so` / `libutils.so` (+ related) and rank replacements. |
| `--scan-libs=full` | Walk every `*.so` under `/system/lib64` and `/system_ext/lib64`. |
| `--min-new-version N` | First Android release using the new mangling (default: 15). |
| `--apply` | Write changes. Without it, only a diff is printed. |
| `--adb PATH` | Override the `adb` binary (default: whichever is in `PATH`). |

---

## Compatibility

| Android version | Status |
|---|---|
| 11, 12, 12L, 13, 14 | Supported — original mangling, no changes needed. |
| 15 | Supported — `SurfaceComposerClient::mirrorSurface` gained a `parent` arg. Patched. |
| 16 | Supported — watch-list entry for `LayerMetadata` namespace move documented in `TUTORIAL.md` §7. |
| future | Patch with `symbol_patcher.py` + three lines of logcat. |

---

## SEO-friendly keywords (for search)

This project answers, among others, these real-world queries:

- "Android 15 `dlsym` returns null `libgui`"
- "Android 16 `SurfaceComposerClient::mirrorSurface` crashes"
- "`libc++abi: terminating due to uncaught exception of type St13runtime_error` Android"
- "private NDK symbol patching"
- "Itanium C++ name mangling Android"
- "`a_native_window_creator.h` Android 16 fix"
- "native window creator overlay not working on new Android"

---

## Contributing

Pull requests that add rows to the **Known Android deltas** table in
`TUTORIAL.md` §7 (old tail → new tail + behavioural change) are the most
valuable contribution — they directly widen the set of crashes
`symbol_patcher.py` can auto-resolve.

---

## License

See [`LICENSE`](LICENSE) if present. The original header carries its own
attribution preserved inside the file.
