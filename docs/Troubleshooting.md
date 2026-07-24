# Troubleshooting

> A handbook of common problems, error messages, and solutions for the WWE '13 Recompilation project.

Each section covers a category of problems. For every issue, the root cause is explained before the solution — understanding why something fails is more useful than just knowing the fix.

---

## Table of Contents

- [LLVM / Clang Issues](#llvm--clang-issues)
- [CMake Issues](#cmake-issues)
- [Ninja Issues](#ninja-issues)
- [Visual Studio Issues](#visual-studio-issues)
- [Windows SDK Issues](#windows-sdk-issues)
- [Git and Submodule Issues](#git-and-submodule-issues)
- [ReXGlue SDK Issues](#rexglue-sdk-issues)
- [Compilation Errors](#compilation-errors)
- [Linker Errors](#linker-errors)
- [Runtime Crashes](#runtime-crashes)

---

## LLVM / Clang Issues

### `clang` is not recognized as an internal or external command

**Cause:** Clang was not added to the system `PATH` during installation, or the terminal was not restarted after installation.

**Fix:**
1. Verify Clang is installed: check `C:\Program Files\LLVM\bin\clang.exe` exists.
2. Add it to PATH:
   - Press `Win + S`, search **"Edit the system environment variables"**
   - Click **Environment Variables**
   - Under **System variables**, find `Path`, click **Edit**
   - Click **New** and add `C:\Program Files\LLVM\bin`
   - Click **OK** on all dialogs
3. Open a new terminal and retry.

---

### `clang version 16` or older version reported

**Cause:** An older LLVM installation is on the PATH, and the newer version was not installed or is in a different location.

**Fix:**
1. Check all Clang installations: `where clang`
2. The first result is the one being used.
3. If the correct version is installed but not first on PATH, reorder PATH entries so the newer LLVM bin directory comes first.
4. Alternatively, re-run the LLVM installer and select **"Add to PATH"** to overwrite the PATH entry.

---

### CMake cannot find `clang++`

**Cause:** `clang` exists on PATH but `clang++` does not. This can happen if only the C frontend was installed.

**Fix:** The standard LLVM installer includes both `clang` and `clang++`. Reinstall LLVM from [github.com/llvm/llvm-project/releases](https://github.com/llvm/llvm-project/releases) using the standard Windows installer.

---

## CMake Issues

### `cmake: command not found`

**Cause:** CMake is not on PATH.

**Fix:** Reinstall CMake from [cmake.org/download](https://cmake.org/download/) and select **"Add CMake to the system PATH"** during installation. Open a new terminal after installation.

---

### `CMake Error: CMAKE_VERSION is less than required`

**Cause:** CMake version is older than 3.25, which is required for CMakePresets.json v6.

**Fix:** Download and install the latest CMake from [cmake.org/download](https://cmake.org/download/). The installation will replace the existing CMake.

---

### `FATAL_ERROR: ReXGlue requires Clang compiler`

**Cause:** CMake found a different compiler (MSVC or GCC) instead of Clang. This happens when the preset is not used, or when CMake's compiler search finds MSVC first.

**Fix:** Always use the preset: `cmake --preset win-amd64`. The preset explicitly sets `CMAKE_C_COMPILER=clang` and `CMAKE_CXX_COMPILER=clang++`. Do not pass `-DCMAKE_CXX_COMPILER=cl` or similar overrides.

If the preset is being used but the error still occurs:
1. Delete the build directory: `rmdir /s /q out\build\win-amd64`
2. Verify `clang++.exe` exists at `C:\Program Files\LLVM\bin\clang++.exe`
3. Re-run `cmake --preset win-amd64`

---

### `FATAL_ERROR: ReXGlue requires Clang 18 or newer (found 17.x)`

**Cause:** Clang is installed but version is below the minimum required (18.0).

**Fix:** Download and install Clang 18 or newer from [github.com/llvm/llvm-project/releases](https://github.com/llvm/llvm-project/releases). Uninstall the old version first, or ensure the new version's bin directory appears before the old one in PATH.

---

### `CMake Error: Generator "Ninja Multi-Config" not found`

**Cause:** Ninja is not installed or not on PATH. CMake cannot use the Ninja Multi-Config generator without Ninja being available.

**Fix:** Install Ninja: `winget install Ninja-build.Ninja`. Open a new terminal and verify: `ninja --version`.

---

### `CMake Error: CMAKE_SIZEOF_VOID_P is 4, ReXGlue only supports 64-bit`

**Cause:** CMake is targeting a 32-bit architecture. This happens if a 32-bit compiler is found or if a 32-bit CMake toolchain file is being used.

**Fix:** Ensure you are using a 64-bit compiler. With the presets, this should be automatic. Verify `clang++ --version` reports `Target: x86_64-pc-windows-msvc` (not `i686`).

---

### `FATAL_ERROR: At least one graphics backend must be enabled`

**Cause:** Both `REXGLUE_USE_D3D12` and `REXGLUE_USE_VULKAN` are `OFF`.

**Fix:** On Windows, `REXGLUE_USE_D3D12` defaults to `ON`. If you explicitly set it to `OFF`, ensure `REXGLUE_USE_VULKAN=ON` is also set. Never disable both.

---

## Ninja Issues

### `ninja: command not found`

**Cause:** Ninja is not installed or not on PATH.

**Fix:**
- Via winget: `winget install Ninja-build.Ninja` — open a new terminal after installation.
- Manual: Download `ninja-win.zip` from [github.com/ninja-build/ninja/releases](https://github.com/ninja-build/ninja/releases), extract `ninja.exe` to a folder on PATH.

---

### `ninja: error: build.ninja: Permission denied`

**Cause:** The `build.ninja` file is locked by another process (another CMake configure run, or antivirus software scanning the file).

**Fix:** Close other terminals or processes that may be accessing the build directory. If antivirus is the cause, add the build directory to the antivirus exclusion list.

---

### `FAILED: ... error: too many errors emitted`

**Cause:** A compilation error triggered a cascade of subsequent errors, flooding the output. Ninja stops after a certain number of errors.

**Fix:** Look at the first error in the output — that is the root cause. Subsequent errors are usually consequences of the first. Fix the first error and rebuild.

---

## Visual Studio Issues

### Visual Studio 2022 is not finding the Windows SDK

**Cause:** The Windows SDK was not installed with the Visual Studio C++ workload, or it was installed but not detected.

**Fix:**
1. Open the Visual Studio Installer
2. Click **Modify** on your Visual Studio 2022 installation
3. Under **Individual components**, search for "Windows SDK" and install the latest version
4. Restart Visual Studio and retry

---

### Visual Studio cannot open the `.sln` file

**Note:** This project uses CMake, not `.sln` files. Open the repository folder via **File → Open → Folder** instead of looking for a `.sln` file.

---

## Windows SDK Issues

### Linker error: `cannot open include file: 'windows.h'`

**Cause:** The Windows SDK is not installed or the include path is not configured.

**Fix:** Ensure the "Desktop development with C++" workload is installed in Visual Studio 2022. CMake uses the Windows SDK headers automatically when MSVC linker is available.

---

### `d3d12.h: No such file or directory`

**Cause:** The DirectX headers are not installed. They come with the Windows 10/11 SDK.

**Fix:** Install the Windows 11 SDK (10.0.22621.0 or later) via the Visual Studio Installer → Individual components.

---

## Git and Submodule Issues

### `fatal: not a git repository`

**Cause:** The repository was downloaded as a ZIP rather than cloned, or the `.git` directory is missing.

**Fix:** Clone the repository properly:
```cmd
git clone https://github.com/OWNER/wwe13-recompilation.git
cd wwe13-recompilation
git submodule update --init --recursive
```

---

### `rexglue-sdk/` directory is empty

**Cause:** The repository was cloned without `--recurse-submodules`, and the submodule was not initialized afterward.

**Fix:**
```cmd
git submodule update --init --recursive
```

This downloads the ReXGlue SDK into `rexglue-sdk/`.

---

### `error: Server does not allow request for unadvertised object`

**Cause:** The submodule commit referenced in `.gitmodules` was not fetched.

**Fix:**
```cmd
git submodule update --init --recursive --force
```

---

### Git pre-commit hook fails

**Cause:** The ReXGlue SDK includes a pre-commit hook (`scripts/git/hook-pre-commit.ps1`) that runs checks before each commit.

**Fix:** Read the error output from the hook — it will explain what check failed. Common causes are clang-format violations or missing CHANGELOG entries. Fix the underlying issue rather than bypassing the hook.

---

## ReXGlue SDK Issues

### Codegen tool produces no output

**Cause:** The `manifest.toml` path or XEX path is incorrect, or the tool failed silently.

**Fix:**
1. Run with verbose output: `rexcodegen --config manifest.toml --verbose`
2. Check that the path to the XEX binary in `manifest.toml` is correct and the file exists
3. Ensure the output directory exists and is writable

---

### Codegen tool reports "No v* tag reachable from HEAD"

**Cause:** The ReXGlue SDK submodule has no git tags (common with a fresh clone that didn't fetch tags).

**Fix:** This is a warning, not an error. The version string will use a fallback. To suppress it, fetch tags:
```cmd
cd rexglue-sdk
git fetch --tags
```

---

### `rexruntime.dll` not found when running the application

**Cause:** The `rexglue_configure_target()` CMake function stages DLLs next to the executable automatically, but only when the target is built. If you copied the executable without the DLL, it won't start.

**Fix:** Always run from the build output directory (`out/win-amd64/Debug/`), or ensure `rexruntime.dll` is copied alongside the executable.

---

## Compilation Errors

### `error: use of undeclared identifier 'ctx'`

**Cause:** A hook function is defined outside of a `REX_HOOK`, `REX_HOOK_RAW`, or `REX_FUNC` macro. The `ctx` and `base` parameters are only available inside these macros.

**Fix:** Wrap the code in the correct macro:
```cpp
// Wrong
void MyHook() {
    u32 value = ctx.r3.u32;  // ctx not in scope
}

// Correct
REX_HOOK_RAW(sub_82012ABC) {
    u32 value = ctx.r3.u32;  // ctx is in scope
}
```

---

### `error: 'REX_LOAD_U32' was not declared`

**Cause:** The generated `wwe13_init.h` header was not included in the translation unit.

**Fix:** Add `#include "generated/wwe13_init.h"` at the top of the file. All files that use generated macros must include this header.

---

### `error: C++23 feature not available`

**Cause:** The file is being compiled with an older C++ standard. This can happen if a file sets its own standard or if the CMake configuration was altered.

**Fix:** Ensure `CMAKE_CXX_STANDARD=23` is set. With the ReXGlue preset, this is automatic. Check that no file-level pragmas or per-target CMake settings are overriding the standard.

---

### `error: static assertion failed: ReXGlue only supports 64-bit`

**Cause:** The code is being compiled for a 32-bit target.

**Fix:** Ensure the compiler is targeting x86_64. With the CMake presets, this is handled automatically. Verify with `clang++ --version` that the target is `x86_64`.

---

## Linker Errors

### `LNK2001: unresolved external symbol sub_82012ABC`

**Cause:** A function is declared with `DECLARE_REX_FUNC(sub_82012ABC)` in the init header but has no corresponding definition anywhere. Either the generated `.cpp` file is not included in the build, or a `REX_HOOK` / `REX_STUB` for that function is missing.

**Fix:**
1. Check `generated/rexglue.cmake` — the function's `.cpp` file should be listed there
2. Verify the `.cpp` file is included in the build via `include(generated/rexglue.cmake)` in `CMakeLists.txt`
3. If the function is a kernel import stub, add `REX_STUB(sub_82012ABC)` in a stubs file

---

### `LNK2019: unresolved external symbol __imp__sub_82012ABC`

**Cause:** `DECLARE_REX_FUNC` generates an `extern "C"` declaration for both `name` and `__imp__name`. The `__imp__` variant is used for import thunks. If the generated code references `__imp__name` but no `DEFINE_REX_FUNC` exists, the linker fails.

**Fix:** Ensure every `DECLARE_REX_FUNC` has a corresponding `DEFINE_REX_FUNC` (in generated code) or a `REX_HOOK` / `REX_STUB` (in hook/stub code).

---

### `LNK1248: image size exceeds maximum`

**Cause:** On Windows, the default executable size limit is 4 GB. A very large game with many generated functions can exceed this.

**Fix:** This is a known issue for large recompilation projects. Solutions:
1. Split the generated code into a DLL module using `rexglue_configure_module_target()`
2. Enable `non_volatile_as_local` and other context optimization flags in `manifest.toml` to reduce code size

---

## Runtime Crashes

### Application crashes immediately with `access violation at 0x00000000`

**Cause:** A null pointer dereference. In guest code, this usually means a guest address of `0` was used without a null check. The `REX_LOAD_U32(0)` would read from `base + 0`, which in the guest memory layout is the null guard region.

**Fix:** Check the stack trace in the debugger. If inside generated code, look for a `REX_LOAD_U32` or `REX_STORE_U32` call where the address argument evaluates to `0`. This usually means an uninitialized pointer or a failed allocation that returned null.

---

### Application crashes with "Unimplemented PPC instruction"

**Cause:** The codegen tool encountered a PPC instruction it does not have a translation for, and inserted `REX_UNIMPLEMENTED(addr, opcode)` which throws `std::runtime_error`.

**Fix:**
1. The error message includes the address and opcode of the unimplemented instruction
2. Check the ReXGlue Discord or GitHub Issues for whether this instruction is planned
3. Report it as a GitHub Issue on the ReXGlue SDK repository with the opcode details

---

### Application exits immediately with "invalid function called"

**Cause:** An indirect function call resolved to an address not in `PPCFuncMappings[]`. The game called a function through a pointer, but the codegen tool did not discover that address during analysis.

**Fix:**
1. Note the address in the error message
2. Check if it falls within the code section of the XEX (expected range: `0x82000000+`)
3. This is a codegen analysis gap — report it with the function address to the ReXGlue repository
4. As a workaround, the address can be manually added to the function mapping table

---

### "STUB" warnings flooding the log, nothing renders

**Cause:** Critical kernel functions are still stubs. A common case is that the graphics initialization function (which calls into Xenos register setup) is returning without doing anything.

**Fix:** Sort the stub log by frequency. The most-called stubs are the most important. Implement the highest-frequency stubs first, starting with thread management and memory allocation.
