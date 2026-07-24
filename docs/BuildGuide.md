# Build Guide

> Advanced build, configuration, debugging, and profiling guide for the WWE '13 Recompilation project.

This document goes beyond the basics in [SETUP.md](../SETUP.md) and covers the full lifecycle of the build system: configuration options, incremental builds, debugging techniques, profiling with Tracy, and release builds.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Project Configuration](#project-configuration)
- [Building](#building)
- [Running the Application](#running-the-application)
- [Debugging](#debugging)
- [Profiling with Tracy](#profiling-with-tracy)
- [Running the Codegen Tool](#running-the-codegen-tool)
- [Incremental Builds](#incremental-builds)
- [Release Builds](#release-builds)
- [Build Options Reference](#build-options-reference)
- [Cleaning the Build](#cleaning-the-build)
- [Troubleshooting Build Issues](#troubleshooting-build-issues)

---

## Prerequisites

Before following this guide, complete all steps in [SETUP.md](../SETUP.md). Verify:

```cmd
clang --version    :: Must be 18+
cmake --version    :: Must be 3.25+
ninja --version    :: Any recent version
```

---

## Project Configuration

### First-Time Configuration

From the repository root, run:

```cmd
cmake --preset win-amd64
```

This uses the preset defined in `CMakePresets.json` which sets:
- Generator: Ninja Multi-Config
- Compiler: `clang` / `clang++`
- Flags: `-march=x86-64-v3` (requires AVX2 — Sandy Bridge+ CPU)
- C++ Standard: C++23
- Configurations: Debug, Release, RelWithDebInfo

Build files are generated in `out/build/win-amd64/`.

### Available Presets

| Preset | Platform | Notes |
|--------|---------|-------|
| `win-amd64` | Windows x64 | Default for this project |
| `win-arm64` | Windows ARM64 | For ARM64 Windows devices |
| `linux-amd64` | Linux x64 | Requires GTK3 and Vulkan SDK |
| `linux-arm64` | Linux ARM64 | Raspberry Pi 4+, Apple Silicon (Rosetta) |

### Configuration Options

Pass CMake cache variables with `-D`:

```cmd
cmake --preset win-amd64 -DREXGLUE_USE_VULKAN=ON -DREXGLUE_USE_D3D12=OFF
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `REXGLUE_USE_D3D12` | BOOL | ON (Win) | Enable Direct3D 12 backend |
| `REXGLUE_USE_VULKAN` | BOOL | OFF (Win) | Enable Vulkan backend |
| `REXGLUE_ENABLE_TRACY` | BOOL | ON | Tracy profiler integration |
| `REXGLUE_ENABLE_PERF_COUNTERS` | BOOL | ON | Lightweight performance counters |
| `REXGLUE_ENABLE_SANITIZERS` | BOOL | OFF | UBSan (undefined behavior sanitizer) |
| `REXGLUE_ENABLE_FIDELITYFX` | BOOL | OFF | AMD FidelityFX (experimental) |
| `REXGLUE_BUILD_TESTS` | BOOL | OFF | PPC instruction tests |

> **Note on `-march=x86-64-v3`:** This requires a CPU with AVX2 support (Intel Haswell 2013+ or AMD Ryzen 2017+). If your CPU is older, edit `CMakePresets.json` and change `x86-64-v3` to `x86-64`.

---

## Building

### Build Debug (development)

```cmd
cmake --build out/build/win-amd64 --config Debug
```

- No optimizations (`-O0`)
- Full debug symbols
- Tracy profiler enabled (compiles profiling zones)
- Perf counter overlay enabled
- Output: `out/win-amd64/Debug/`

### Build Release

```cmd
cmake --build out/build/win-amd64 --config Release
```

- Full optimizations (`-O3`, `-DNDEBUG`)
- No debug symbols
- Tracy and perf counters compiled out
- Output: `out/win-amd64/Release/`

### Build RelWithDebInfo (profiling)

```cmd
cmake --build out/build/win-amd64 --config RelWithDebInfo
```

- Moderate optimizations (`-O2`)
- Full debug symbols (`.pdb` on Windows)
- Tracy and perf counters enabled
- Output: `out/win-amd64/RelWithDebInfo/`

This configuration is best for profiling — you get real-world performance with symbol information for meaningful stack traces.

### Parallel Builds

Ninja automatically uses all available CPU cores. To limit parallelism (useful on machines with limited RAM):

```cmd
cmake --build out/build/win-amd64 --config Debug -- -j4
```

### Building a Specific Target

```cmd
cmake --build out/build/win-amd64 --config Debug --target wwe13
```

---

## Running the Application

After building, the executable and required DLLs are in the output directory:

```cmd
cd out\win-amd64\Debug
wwe13.exe --game_data_root "C:\path\to\wwe13\game_data"
```

### Command-Line Arguments

ReXGlue applications accept these standard arguments:

| Argument | Description |
|----------|-------------|
| `--game_data_root <path>` | Path to the extracted game data directory |
| `--user_data_root <path>` | Path for save data (defaults to `%APPDATA%\wwe13\`) |
| `--update_data_root <path>` | Path to title update data (optional) |
| `--cache_path <path>` | Path for shader cache (defaults to `%APPDATA%\wwe13\cache\`) |

### Game Data Layout

The game data directory should contain the extracted XEX and associated files:

```
game_data/
├── default.xex         ← The main game executable
├── default.xex.xzx     ← Optional: decryption headers
└── (other game files)
```

---

## Debugging

### Attaching Visual Studio Debugger

1. Open Visual Studio 2022
2. Go to **Debug → Attach to Process**
3. Find `wwe13.exe` in the process list
4. Click **Attach**

For source-level debugging, configure the PDB path in Visual Studio's **Tools → Options → Debugging → Symbols**.

### Debugging with Source in Visual Studio

1. Open the repository folder in Visual Studio via **File → Open → Folder**
2. Visual Studio will detect the `CMakePresets.json` and offer CMake targets
3. Set `wwe13` as the startup project
4. Press **F5** to build Debug and run with the debugger attached

### Debug Overlay

In Debug and RelWithDebInfo builds, the `ReXApp` base class includes a debug overlay. Press the configured key binding (default varies — check `OnKeyDown` in the app class) to toggle:
- **Log view** — scrollable output of all `REXCPU_*` and `REXKRNL_*` log messages
- **Performance overlay** — frame time, GPU time, perf counter values

### Logging

Log output goes to:
1. The debug overlay (in-app)
2. The console window
3. A log file (if `--log_file` is configured)

Log macros by subsystem:

| Macro | Subsystem | Level |
|-------|----------|-------|
| `REXCPU_DEBUG(...)` | PPC core | Debug |
| `REXCPU_WARN(...)` | PPC core | Warning |
| `REXKRNL_DEBUG(...)` | Kernel | Debug |
| `REXKRNL_WARN(...)` | Kernel | Warning |
| `REXGFX_DEBUG(...)` | Graphics | Debug |
| `REXGFX_WARN(...)` | Graphics | Warning |

---

## Profiling with Tracy

Tracy is an open-source profiler integrated into the ReXGlue SDK. It is compiled out in Release builds but active in Debug and RelWithDebInfo.

### Setup

1. Download the Tracy profiler GUI from [github.com/wolfpld/tracy](https://github.com/wolfpld/tracy/releases)
2. Run the profiler GUI: `Tracy.exe`
3. Run your application in `RelWithDebInfo` configuration
4. In the Tracy GUI, click **Connect** — it will detect the running application automatically

### What to Look For

| Metric | What it tells you |
|--------|------------------|
| Frame time spike | Expensive function call in that frame |
| Lock contention | Threads waiting on mutexes or kernel objects |
| Long stub calls | Stubs that should be real implementations |
| Repeated calls to same function | Possible hot path to optimize |

### Enabling Guest Function Profiling

By default, Tracy zones are placed on hook functions. To profile individual recompiled guest functions, define `REXGLUE_PROFILE_GUEST_FUNCTIONS` in your CMake configuration:

```cmd
cmake --preset win-amd64 -DCMAKE_CXX_FLAGS="-DREXGLUE_PROFILE_GUEST_FUNCTIONS"
```

> **Warning:** This adds a Tracy zone to every recompiled function. For a game with 100,000 functions, this will significantly impact performance and produce enormous profiler data. Use only when targeting a specific subsystem.

---

## Running the Codegen Tool

The codegen tool (`rexcodegen.exe` in the SDK output) is run once to generate the C++ source from the XEX binary.

### Prerequisites

- The WWE '13 XEX binary placed at a known path (e.g., `game_data/default.xex`)
- A `manifest.toml` in the project root (see [docs/ReXGlue.md](ReXGlue.md) for format)

### Running

```cmd
rexglue-sdk\out\win-amd64\Debug\rexcodegen.exe --config manifest.toml
```

Or if the SDK is installed:

```cmd
rexcodegen --config manifest.toml
```

### Expected Output

The tool will:
1. Load and decrypt the XEX
2. Print analysis progress (function count, scan results)
3. Write generated files to the output directory specified in `manifest.toml`

This can take several minutes for a large binary.

### Regenerating After Manifest Changes

If you change config flags in `manifest.toml` (e.g., enabling `ctr_as_local`), re-run the codegen tool. The generated files in `src/generated/` will be overwritten. Your hooks in `src/hooks/` are unaffected.

---

## Incremental Builds

Ninja tracks file dependencies and only recompiles changed files.

After modifying a hook file:

```cmd
cmake --build out/build/win-amd64 --config Debug
```

Ninja will recompile only the changed `.cpp` files and re-link. For a project with many generated translation units, incremental builds after a single hook change should take seconds.

After regenerating source with the codegen tool, Ninja will detect that generated files changed and recompile all affected units. This can take several minutes.

---

## Release Builds

When preparing a release:

1. Build in Release configuration:
   ```cmd
   cmake --build out/build/win-amd64 --config Release
   ```

2. The output directory `out/win-amd64/Release/` contains everything needed:
   - `wwe13.exe`
   - `rexruntime.dll`
   - Any other required DLLs (staged automatically by `rexglue_configure_target`)

3. Strip debugging symbols (already absent in Release builds).

4. Test the Release build explicitly — Release has different code paths (NDEBUG, different optimizations) and can expose latent bugs not visible in Debug.

> **Never ship Debug builds.** They are significantly larger and slower, and include profiling and logging overhead.

---

## Build Options Reference

### Compiler Flags

The SDK sets these globally for all targets:

| Flag | Effect |
|------|--------|
| `-fno-strict-aliasing` | Prevents aggressive alias-based optimizations that break type-punning |
| `-ffp-model=strict` | IEEE 754 strict floating-point (prevents dangerous FP optimizations) |
| `-fno-char8_t` | Disables C++20 `char8_t` (avoids UTF-8 literal incompatibilities) |
| `-march=x86-64-v3` | Requires AVX2 instruction set |
| `-mcmodel=large` | (Linux only) Allows >2GB code+data range |
| `-msse4.1` | (Non-MSVC, AMD64) Enables SSE 4.1 for simde |

### Debug-Only Flags

| Flag | Effect |
|------|--------|
| `-g` | Generate debug information |
| `-O0` | No optimizations |

### Release-Only Flags

| Flag | Effect |
|------|--------|
| `-O3` | Maximum optimization |
| `-DNDEBUG` | Disables assertions and debug-only code |

---

## Cleaning the Build

### Soft Clean (CMake targets only)

```cmd
cmake --build out/build/win-amd64 --config Debug --target clean
```

This removes compiled objects and binaries but keeps the CMake cache.

### Hard Clean (full reconfiguration)

```cmd
rmdir /s /q out\build\win-amd64
cmake --preset win-amd64
cmake --build out/build/win-amd64 --config Debug
```

A hard clean is necessary when:
- CMakeLists.txt is significantly changed
- The SDK submodule is updated to a new version
- Build errors persist after minor source changes

---

## Troubleshooting Build Issues

### "error: ReXGlue requires Clang 18 or newer"

The CMake preset sets `CMAKE_CXX_COMPILER=clang++`. If Clang is not on your PATH, CMake falls back to MSVC. Verify:

```cmd
clang++ --version
```

If not found, add `C:\Program Files\LLVM\bin` to PATH and reconfigure.

### "ninja: command not found" during cmake --build

Ninja is not on PATH. Install via `winget install Ninja-build.Ninja` and open a new terminal.

### LNK2001: unresolved external symbol

A hook or stub is declared but not defined, or a `DECLARE_REX_FUNC` does not have a matching `DEFINE_REX_FUNC` or `REX_HOOK`/`REX_STUB`. Check for typos in function names.

### "FATAL_ERROR: At least one graphics backend must be enabled"

Both `REXGLUE_USE_D3D12` and `REXGLUE_USE_VULKAN` are `OFF`. Ensure at least one is enabled. On Windows, `REXGLUE_USE_D3D12` should be `ON` by default.

### Extremely slow first build

The first build compiles the entire SDK and all generated translation units. For a large game, this can take 15–30 minutes. Use `-j4` to reduce parallelism if the machine runs out of RAM.

For more help, see [docs/Troubleshooting.md](Troubleshooting.md).
