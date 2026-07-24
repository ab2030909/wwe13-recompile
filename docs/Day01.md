# Day 01 — Environment Setup

**Date:** July 25, 2026
**Phase:** 1 — Environment Setup
**Status:** ✅ Complete

---

## Summary

Today was the first day of the WWE '13 Recompilation project. The goal was to get a complete development environment running, understand what the project involves at a high level, and create the initial documentation structure.

Starting from zero: no prior experience with reverse engineering, Xbox 360 architecture, or recompilation. This journal is an honest record of what was set up, what was learned, and what questions remain.

---

## What Was Set Up

### Tools Installed

All tools were installed and verified working:

| Tool | Version | Notes |
|------|---------|-------|
| Visual Studio 2022 | 17.x | Desktop C++ workload, Windows 11 SDK |
| LLVM / Clang | 18.x | Added to system PATH |
| CMake | 3.29.x | Added to system PATH during install |
| Ninja | 1.11.x | Installed via winget |
| Python | 3.12.x | Added to system PATH during install |
| Git | 2.45.x | Configured with user name and email |
| Ghidra | Latest | JDK 21 (Eclipse Temurin) installed |
| ReXGlue SDK | 0.8.x | Cloned as submodule |

### Verification Commands

Every tool was confirmed working:

```cmd
clang --version
# clang version 18.1.8

cmake --version
# cmake version 3.29.x

ninja --version
# 1.11.1

python --version
# Python 3.12.x

git --version
# git version 2.45.x.windows.1
```

### ReXGlue SDK Build

Configured and built the ReXGlue SDK:

```cmd
cmake --preset win-amd64 -S rexglue-sdk
cmake --build rexglue-sdk/out/build/win-amd64 --config Debug
```

Build completed successfully. Output at `rexglue-sdk/out/win-amd64/Debug/`.

---

## What Was Learned

### What is Static Recompilation?

Static recompilation is fundamentally different from emulation. An emulator simulates the original hardware and interprets or JIT-compiles game code at runtime. A static recompiler reads the original binary and translates it into source code for a different platform — ahead of time. The result is compiled and runs natively.

The analogy: imagine you have a book written in French. An emulator reads it aloud, translating word by word as it goes. A static recompiler translates the whole book into English first, then you read the English version directly. The English version is faster to read.

For WWE '13:
- The original binary is PowerPC machine code for the Xbox 360
- ReXGlue translates it into C++23
- That C++ is compiled with Clang 18+ for x64 Windows or Linux
- The result runs natively — no Xbox 360 hardware needed

### What is ReXGlue?

ReXGlue is the toolkit that makes this possible. It has two main parts:

1. **Codegen tool (`rexglue` CLI):** Reads the XEX binary, analyzes its functions, and outputs a C++ project. Every function in the original binary becomes a C++ function like `void sub_82012345(PPCContext& ctx, uint8_t* base)`.

2. **Runtime SDK:** A library that replaces the Xbox 360's operating system. It provides a memory model, kernel object implementations (threads, mutexes, events), file system, graphics (via D3D12 or Vulkan), audio, and input.

### What is a PPCContext?

This is one of the most important things to understand. Every recompiled function receives two arguments:
- `PPCContext& ctx` — a structure representing the Xbox 360 CPU's register state
- `uint8_t* base` — a pointer to the start of the 4GB virtual guest memory space

The `PPCContext` contains the PowerPC CPU's state:
- 32 general-purpose registers: `r0`–`r31` (integer arithmetic)
- 32 floating-point registers: `f0`–`f31` (floating-point arithmetic)
- 128 vector registers: `v0`–`v127` (SIMD / Altivec operations)
- Special registers: CR (condition), XER (overflow), LR (link/return address), CTR (count), FPSCR

When a function reads an argument, it reads from `ctx.r3`, `ctx.r4`, etc. When it writes a return value, it writes to `ctx.r3`. This mirrors the Xbox 360 PowerPC calling convention.

### What is a XEX?

XEX stands for Xbox EXecutable. It is the executable format used by the Xbox 360, similar to how Windows uses `.exe` (PE format) and Linux uses ELF. A XEX file contains:
- Headers describing the binary (entry point, import/export tables, etc.)
- Compressed code and data sections
- A list of kernel imports — the OS functions the game needs

When ReXGlue analyzes a XEX, it decrypts and decompresses these sections, then walks the code to identify all function boundaries.

---

## Project Structure Created

The initial documentation structure was created:

```
wwe13-recompilation/
├── rexglue-sdk/      ← ReXGlue SDK (submodule)
├── docs/
│   ├── Day01.md      ← This file
│   ├── Architecture.md
│   ├── ReXGlue.md
│   ├── Xbox360.md
│   ├── Research.md
│   ├── BuildGuide.md
│   ├── Troubleshooting.md
│   └── Glossary.md
├── README.md
├── ROADMAP.md
├── SETUP.md
├── DEVELOPMENT.md
├── CONTRIBUTING.md
├── CHANGELOG.md
├── LICENSE
└── .gitignore
```

---

## Key Discoveries

### ReXGlue Requires Clang 18+

This is enforced at the CMake configure stage. If any other compiler is used, CMake exits with:
```
FATAL_ERROR: ReXGlue requires Clang compiler.
```
The reason is C++23 support. ReXGlue uses features like `std::byteswap`, `std::endian`, and C++23 concepts that are only fully available in recent Clang versions.

### The SDK Uses C++23 Everywhere

The `CMakePresets.json` sets `CMAKE_CXX_STANDARD: 23`. This is unusual for a library — most projects target C++17 for maximum compatibility. ReXGlue trades compatibility for expressiveness: C++23 templates and concepts make the hook API type-safe in ways that were not possible in older standards.

### Large Code Model on Linux

On Linux x86_64, ReXGlue compiles with `-mcmodel=large`. This is necessary because recompiled executables can be over 35MB of code. The default x86-64 code model assumes code and data are within a 2GB range, which a 35MB code binary can violate.

### Memory Layout

The Xbox 360's memory space is mapped at host address `base` (which points to a 4GB allocation). Guest addresses are always 32-bit values (`u32`). To access guest memory from a host address:
```cpp
uint8_t* host_ptr = base + guest_address;
```
The `REX_LOAD_U32(addr)` macro does this plus a big-endian byte swap, because the Xbox 360 CPU is big-endian while x64 is little-endian.

---

## Open Questions

These questions were raised during Day 1. They will be answered as the project progresses.

1. **How does the codegen tool identify function boundaries?** Does it use call graph analysis, pattern matching, or both?

2. **What is the `PPCFuncMappings[]` table?** How is it used by the runtime to resolve indirect function calls?

3. **How does the hook system work at link time?** When `REX_HOOK(sub_82012345, MyNativeFunc)` is used, how does the linker connect the hook to the generated code?

4. **What kernel imports does WWE '13 use?** Are they mostly xboxkrnl.exe or xam.xex? How many are there?

5. **What graphics technique does WWE '13 use?** Deferred rendering? Forward rendering? What vertex format?

6. **How does XMA audio work?** What is the relationship between the XMA codec and XAudio2?

7. **How does the VFS work?** How are game data paths mapped from `game:` / `d:` URIs to host filesystem paths?

---

## Next Steps

- [ ] Read all public headers in `rexglue-sdk/include/rex/` carefully
- [ ] Load the WWE '13 XEX binary in Ghidra and run auto-analysis
- [ ] Document the kernel imports found in the binary
- [ ] Understand the `ReXApp` lifecycle in detail — what each virtual hook does
- [ ] Create the `manifest.toml` for WWE '13
- [ ] Run the codegen tool on the XEX binary for the first time

---

## References

- [ReXGlue SDK](https://github.com/rexglue/rexglue-sdk)
- [ReXGlue Discord](https://discord.gg/CNTxwSNZfT)
- [XenonRecomp](https://github.com/hedge-dev/XenonRecomp) — the original static recompiler for Xbox 360
- [Project Xenia](https://github.com/xenia-project/xenia) — Xbox 360 emulator (ReXGlue's kernel layer is based on Xenia)
- [docs/ReXGlue.md](ReXGlue.md) — detailed SDK reference
- [docs/Xbox360.md](Xbox360.md) — Xbox 360 hardware reference
- [docs/Glossary.md](Glossary.md) — definitions of all key terms
