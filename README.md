# Titan Souls – 64-bit (arm64) Android Compatibility Layer

> ⚠️ **Experimental and unstable.** The game may crash, freeze, or fail to reach the main menu. This is a work in progress, not a finished port.

> 📌 **Personal use only.** The APK produced by this project contains the original game. **Do not upload or share the built APK.** See [Legal notice](#legal-and-copyright-notice).

---

## Table of contents

1. [What is this?](#what-is-this)
2. [How it works](#how-it-works)
3. [Current status](#current-status)
4. [Known issues](#known-issues)
5. [Repository contents](#repository-contents)
6. [Building](#building)
7. [Installing](#installing)
8. [Debugging and reporting bugs](#debugging-and-reporting-bugs)
9. [Legal and copyright notice](#legal-and-copyright-notice)
10. [Third-party components](#third-party-components)

---

## What is this?

The Android build of *Titan Souls* ships only **32-bit ARMv7** native libraries
(`armeabi-v7a`). Many recent phones and tablets have dropped 32-bit support, so
the original APK will not install or run on them.

This project tries to run the game's **unmodified 32-bit native code** on
64-bit-only devices by **emulating ARM32 at runtime** rather than recompiling
it. No game source code is used or required, because none is publicly available.

## How it works

1. **ARM32 JIT.** [dynarmic](https://github.com/lioncash/dynarmic) translates
   the game's ARMv7 machine code into ARM64 code while the game runs. The
   original 32-bit library is treated as *data* and is shipped inside the APK
   as `assets/guest/libTestSuite.so`.
2. **Custom loader.** A 64-bit native library (also named `libTestSuite.so`, so
   the game's Java side loads it as usual) loads the guest library into
   emulated memory and redirects the game's imports (libc, pthread, math
   helpers such as `__aeabi_idiv`, etc.) to 64-bit host implementations.
3. **JNI emulation.** The game's native code expects an Android Java VM. The
   loader provides a minimal `JavaVM` / `JNIEnv` that answers the calls the
   game makes (class lookups, method IDs, strings, ...).
4. **Host bridges.** The loader also bridges the game's OpenGL ES, zlib and
   Android calls to the device's own 64-bit libraries, and restores the game's
   `.obb` expansion file from a backup location if it is missing after a
   reinstall.
5. **Silent audio stand-ins.** FMOD Ex (the game's closed-source, 32-bit-only
   audio engine) is replaced inside the emulator by stubs that always report
   success, so the game runs **without sound** for now. The bundled
   `libfmodex.so` and `libAmazonGamesJni.so` are empty placeholders that only
   let the Java side's `System.loadLibrary(...)` calls succeed on arm64.
6. **APK repackaging.** [Apktool](https://github.com/iBotPeaches/Apktool)
   unpacks the original APK, the 32-bit library is moved to the guest assets,
   the arm64 libraries are added, the NVIDIA Shield requirement is removed from
   the manifest, and the result is aligned and signed with a debug key.
7. **Diagnostics.** A watchdog prints guest thread states, system-call counts
   and crash details on screen (lines starting with `[watch]`, `[crash]`,
   `[status]`, `[logcat]`). This is what you see in the red `FATAL` screens.

```
 Original APK (yours)
   ├── lib/armeabi-v7a/libTestSuite.so ──► assets/guest/libTestSuite.so  (guest code, emulated)
   └── Java / assets / resources        ──► unchanged
 This project (built from source)
   └── lib/arm64-v8a/
         ├── libTestSuite.so   (dynarmic loader + JNI + imports)
         ├── libfmodex.so          (empty placeholder)
         └── libAmazonGamesJni.so  (empty placeholder)
```

## Current status

- [x] Game boots and creates its worker threads
- [x] Guest code runs (thread pool, gamepad checker, etc.)
- [ ] C++ exception unwinding (needs `.ARM.exidx` lookup support)
- [ ] Stable JNI emulation (some calls appear to return NULL, causing crashes)
- [ ] Reaches the title screen reliably
- [ ] Audio (FMOD is currently silent stubs, so the game has no sound)
- [ ] Gameplay and saving verified on real devices

## Known issues

| Symptom (on-screen `FATAL` message) | Probable cause |
|---|---|
| `a C++ exception was thrown but exception unwinding is not implemented` | The unwinder cannot find the exception tables (`.ARM.exidx`) of the loaded guest library. A lookup such as `__gnu_Unwind_Find_exidx` / `dl_iterate_phdr` must point to the guest's tables. |
| `segmentation fault: read of 0x20` / `0x58` | NULL object plus a small field offset. Likely an unimplemented or NULL-returning JNI call (e.g. `FindClass`, `GetMethodID`, `CallStaticObjectMethod`). |
| `segmentation fault` at addresses that look like ASCII text (e.g. `0x726576c4`) | Possible memory corruption or a thread-synchronisation race in the emulation layer. |
| `the game called abort()` | The game detected an error and aborted; check the lines above it. Note that some `abort()` / `throw` calls are currently "soft-stubbed" (logged as `[cxx] soft-stub ...` and ignored), which can hide the real problem and cause a crash later. |
| Crashes change between runs | Suggests a race or memory problem rather than one fixed bug. |

The guest library keeps its symbol names, so crash addresses can be resolved to
functions: take the guest `pc` / `lr`, convert to an offset in the original
`libTestSuite.so` and use `llvm-symbolizer` or `addr2line`.

## Repository contents

| Path | Description |
|---|---|
| `titan64_bundle.zip` | Project sources: `native/` (CMake project with `src/` for the loader, ELF32 loader, HLE of libc/Android/GLES/JNI, FMOD stubs, and `stubs/`, `tests/`), plus a short `README.md` and `FIX-NOTE.txt` change notes |
| `dynarmic-master.zip` | Snapshot of the dynarmic source used for the build |
| `.github/workflows/` | GitHub Actions workflow *Build Titan Souls arm64 APK* |
| `README.md` | This file |

> The repository should **not** contain any `.apk`, `.obb`, or other file taken
> from the game. If you build with GitHub Actions, see
> [Building with GitHub Actions](#building-with-github-actions) for how to do it
> without publishing the game.

## Building

You need **your own legally obtained copy** of the Titan Souls Android APK
(version `1.0.3`). The build repackages that APK; it does not download or
include the game for you.

### Requirements

- Linux (the workflow uses Ubuntu 24.04)
- Android NDK (recent release with the CMake toolchain file)
- CMake and Ninja
- Boost headers (`libboost-dev`)
- JDK 17 or newer
- Android SDK build-tools (`zipalign`, `apksigner`)
- [Apktool 2.9.3](https://github.com/iBotPeaches/Apktool/releases/tag/v2.9.3)

### Manual build

```bash
# 0. Put your APK next to the sources
cp "Titan Souls_1.03.apk" Titan_Souls_1.03.apk

# 1. Unpack sources and Boost headers
unzip -q titan64_bundle.zip -d work
unzip -q dynarmic-master.zip -d work/ext
mkdir -p work/boostinc && cp -r /usr/include/boost work/boostinc/

# 2. Build the arm64-v8a native libraries
cmake -S work/native -B build -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-26 -DANDROID_STL=c++_static \
  -DCMAKE_BUILD_TYPE=Release \
  -DDYNARMIC_DIR="$PWD/work/ext/dynarmic-master" \
  -DBoost_INCLUDE_DIR="$PWD/work/boostinc" -DBoost_NO_BOOST_CMAKE=ON \
  -DCMAKE_FIND_ROOT_PATH_MODE_INCLUDE=BOTH \
  -DCMAKE_CXX_FLAGS="-DFMT_CONSTEVAL="
cmake --build build -j"$(nproc)"
llvm-strip --strip-unneeded build/libTestSuite.so

# 3. Repackage the APK
java -jar apktool.jar d -f -o app Titan_Souls_1.03.apk
mkdir -p app/assets/guest app/lib/arm64-v8a
cp app/lib/armeabi-v7a/libTestSuite.so app/assets/guest/libTestSuite.so   # 32-bit game library -> emulator data
rm -rf app/lib/armeabi-v7a
cp build/libTestSuite.so build/libfmodex.so build/libAmazonGamesJni.so app/lib/arm64-v8a/
sed -i '/com.nvidia.feature.shield/d' app/AndroidManifest.xml             # do not require an NVIDIA Shield
java -jar apktool.jar b -f -o unsigned.apk app

# 4. Align and sign with a debug key
zipalign -p -f 4 unsigned.apk aligned.apk
keytool -genkeypair -keystore debug.keystore -storepass android -keypass android \
  -alias key -keyalg RSA -keysize 2048 -validity 10000 -dname "CN=titan64"
apksigner sign --ks debug.keystore --ks-pass pass:android --key-pass pass:android \
  --out Titan_Souls_arm64.apk aligned.apk
```

The result is `Titan_Souls_arm64.apk`.

### Building with GitHub Actions

The workflow `Build Titan Souls arm64 APK` runs the same steps and uploads
`Titan_Souls_arm64.apk` as a build artifact. Start it from the **Actions** tab
(`workflow_dispatch`).

The workflow currently expects an `.apk` file in the repository root. Because
that file is copyrighted, **do not keep it in a public repository**. Use one of
these approaches instead:

- Run the workflow in a **private** repository you control, or
- Change the workflow to fetch your APK from a private location (for example a
  private release asset or a secret URL) instead of reading it from the repo.

**Never publish the built APK**, and do not attach it to public Releases.

## Installing

1. Uninstall any existing copy of Titan Souls. The rebuilt APK is signed with a
   different (debug) key, so Android will refuse to install it over the
   original.
2. Copy `Titan_Souls_arm64.apk` to your device and install it (you may need to
   allow installs from unknown sources).
3. The game needs its **OBB expansion file** (`main.<version>.com.devolver.titansouls.obb`)
   from your own copy. Place it in
   `Android/obb/com.devolver.titansouls/`. If it is missing after a reinstall,
   the app looks for a backup copy in a few common folders and restores it;
   otherwise it logs where to put the file.
4. Launch the game. Expect occasional crashes; see [Known issues](#known-issues).

Requirements: a 64-bit ARM device (`arm64-v8a`) running Android 8.0 (API 26) or
newer, since the native build targets `android-26`.

## Debugging and reporting bugs

When opening an issue, please include:

- Device model, chipset and Android version
- Whether the original (unmodified) APK installs on your device
- The **full** red `FATAL` line, including `pc` and `lr` (the on-screen text is
  often cut off; a screenshot of the first lines is fine)
- The last lines of the `[logcat]`, `[crash]` and `[status]` output
- Whether the crash happens at the same point on every run

Do **not** attach the game's APK, OBB or extracted files to an issue.

## Legal and copyright notice

- This project is **not affiliated with, endorsed by, or sponsored by Acid Nerve
  or Devolver Digital**.
- *Titan Souls*, including its code, art, music and all related assets, belongs
  to its respective owners. All rights are reserved by them.
- This repository is intended to contain only **original compatibility code**
  (a loader and emulation layer) and build scripts. It is meant to let a
  **legally owned copy** of the game run on hardware its original build does
  not support.
- **The APK produced by the build contains the original game** (its 32-bit
  library and assets are repackaged inside it). It is therefore for **your
  personal use only** and **must not be redistributed**, uploaded to public
  releases, or shared.
- You must supply your own copy of the game, obtained legally. This project
  does not provide it and does not help bypass any purchase or licence check.
- This software is provided **"as is"**, without warranty of any kind. Use it at
  your own risk.
- Laws on reverse engineering and compatibility differ by country. You are
  responsible for making sure your use complies with the law where you live.
  Nothing here is legal advice.
- If you are a rights holder and believe something in this repository infringes
  your rights, please open an issue or contact **NonameName**
  ([@NonameName-wq](https://github.com/NonameName-wq)) on GitHub and it will be
  addressed promptly.

## Third-party components

| Component | Used for | License |
|---|---|---|
| [dynarmic](https://github.com/lioncash/dynarmic) | ARM32 → ARM64 JIT. Source: `lioncash/dynarmic` (snapshot in `dynarmic-master.zip`) | 0BSD. Its `LICENSE.txt` names Copyright © 2017 merryhime; keep that file and notice when redistributing. |
| [Apktool](https://github.com/iBotPeaches/Apktool) | Unpacking / rebuilding the APK (build time only) | Apache-2.0 `[verify]` |
| Boost (headers) | Build dependency of dynarmic | Boost Software License |
| dynarmic's bundled libraries (`biscuit`, `fmt`, `mcl`, `oaknut`, `robin-map`, `xbyak`, `zycore`, `zydis`) | Dependencies of dynarmic | Permissive licenses, each under its own license file in `externals/*/` (see the license list in dynarmic's README). Keep them when redistributing. |
| DejaVu Sans Mono | The on-screen debug font (`font8x16.h` is a bitmap generated from it) | DejaVu / Bitstream Vera license (permissive, requires keeping the notice) |
| Android NDK system libraries (`log`, `android`, `EGL`, `GLESv2`, `z`) | Linked at build time | Provided by the platform / NDK |

This project's own original code is released under the **MIT License** (see
[`LICENSE`](LICENSE)). That license applies **only** to the original code in this repository and **not** to
*Titan Souls* or to any third-party component.
