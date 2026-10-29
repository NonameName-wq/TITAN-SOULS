# Titan Souls – 64-bit Android Compatibility Layer (Experimental)

> ⚠️ **Experimental and unstable.** The game may crash, freeze, or fail to reach the main menu. This is a work in progress.

## What is this?

The Android build of *Titan Souls* ships only 32-bit ARMv7 native libraries
(`armeabi-v7a`). Many modern phones have dropped 32-bit support, so the
original APK will not install or run on them.

This project tries to run those unmodified 32-bit libraries on 64-bit-only
devices by **emulating the ARM32 code at runtime** instead of recompiling it.
No game source code is used or required.

## How it works

1. **ARM32 JIT.** [dynarmic](https://github.com/merryhime/dynarmic) translates
   the game's ARMv7 machine code into ARM64 code while the game runs.
2. **Custom loader.** The game's `.so` libraries are loaded into memory
   manually, and the game's imports (libc, pthread, math helpers such as
   `__aeabi_idiv`, etc.) are redirected to 64-bit host implementations.
3. **JNI emulation.** The game expects to run inside an Android Java VM.
   A minimal fake `JavaVM` / `JNIEnv` answers the calls the game makes
   (class lookups, method calls, strings).
4. **Diagnostics.** A watchdog reports guest threads, system call counts and
   crash details, which is why you may see `[watch]` and `[crash]` lines.

*[Adjust these points to match your actual code.]*

## Current status

- [x] Game boots and creates its worker threads
- [ ] C++ exception unwinding (needs `.ARM.exidx` lookup support)
- [ ] Stable JNI emulation (some calls currently return NULL → crash)
- [ ] Reaches the title screen and is playable
- [ ] Audio (the game uses FMOD Ex, 32-bit only) `[status?]`

## Known issues

| Error | Likely cause |
|---|---|
| `a C++ exception was thrown but exception unwinding is not implemented` | Unwinder cannot find the exception tables of the loaded library |
| `segmentation fault: read of 0x58 / 0x20` | NULL object plus small offset, probably an unimplemented or NULL JNI result |
| Crashes with changing addresses between runs | Possible memory corruption or a thread-synchronisation race |

## Usage

1. Obtain your **own legally acquired** copy of *Titan Souls* for Android
   (version `[1.0.3]`).
2. Download `titan64_bundle.zip` `[from Releases / this repo]`.
3. `[Install steps here.]`

When reporting a bug, please include your device model, Android version, and
the **full** `FATAL` line (including `pc` and `lr`) from the on-screen log.

## Building

`[Describe how to build, e.g. the CI workflow in .github/workflows, NDK version, CMake commands.]`

## Legal and copyright notice

- **This project is not affiliated with, endorsed by, or sponsored by
  Acid Nerve or Devolver Digital.**
- *Titan Souls*, its code, art, music and all related assets are the property
  of their respective owners. All rights reserved by them.
- **This repository does not contain, and must not be used to distribute,
  any part of the game**: no APK, OBB, native libraries, assets, or extracted
  game files. You must supply your own copy, obtained legally.
- The project contains only original compatibility code (a loader and an
  emulation layer) written to let a legally owned copy run on hardware
  that its original build does not support.
- No game code is modified or redistributed. `[Confirm this is true for your bundle.]`
- This software is provided **"as is"**, without warranty of any kind.
- If you are a rights holder and believe something here infringes your
  rights, please open an issue or contact `[your contact]` and it will be
  addressed promptly.

### Third-party components

| Component | Use | License |
|---|---|---|
| [dynarmic](https://github.com/merryhime/dynarmic) | ARM32 JIT | `[check the license in the repo]` |
| `[other libraries]` | | |

This project's own code is released under `[license, e.g. MIT]`. That license
applies only to the original code in this repository and **not** to
*Titan Souls* or any third-party component.
