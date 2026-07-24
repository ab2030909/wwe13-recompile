# Roadmap

This document describes the long-term development plan for the WWE '13 Recompilation project. Each phase builds on the previous one and includes a detailed checklist of tasks.

Phases are not strictly sequential — research and documentation work continues across all phases. The checklists represent the minimum work needed before moving to the next phase.

---

## Phase 1 — Environment Setup

> **Status: ✅ Complete**

The goal of Phase 1 is to have a fully working development environment where the ReXGlue SDK can be configured, built, and explored.

### Tasks

- [x] Install Visual Studio 2022 with C++ workload and Windows SDK
- [x] Install LLVM / Clang 18+ and verify `clang --version`
- [x] Install CMake 3.25+ and verify `cmake --version`
- [x] Install Ninja and verify `ninja --version`
- [x] Install Python 3.10+ and verify `python --version`
- [x] Install Git and configure user name and email
- [x] Install Ghidra for binary analysis
- [x] Clone the ReXGlue SDK as a submodule at `rexglue-sdk/`
- [x] Configure the ReXGlue SDK project with CMake (`cmake --preset win-amd64`)
- [x] Build the ReXGlue SDK in Debug configuration
- [x] Create initial project documentation structure
- [x] Initialize the Git repository with `.gitignore` and proper structure

---

## Phase 2 — ReXGlue SDK Mastery

> **Status: 🔄 In Progress**

The goal of Phase 2 is to understand the ReXGlue SDK deeply enough to generate a working project from a real XEX binary. This phase is primarily learning and experimentation.

### Tasks

- [ ] Read all ReXGlue SDK public headers in `rexglue-sdk/include/rex/`
- [ ] Understand the `PPCContext` structure (GPRs, FPRs, VMX, CR, XER, FPSCR)
- [ ] Understand `PPCFunc` — the signature of every recompiled function
- [ ] Understand the `REX_HOOK`, `REX_STUB`, and `REX_EXPORT` macros
- [ ] Study the `rex::ReXApp` base class lifecycle (5 init phases)
- [ ] Study `rex::Runtime` — how it owns Memory, VFS, KernelState, Graphics, Audio, Input
- [ ] Study `PPCImageInfo` and how it describes the loaded binary
- [ ] Read the codegen templates in `resources/templates/`
- [ ] Review `init_h.inja` to understand all generated macros (`REX_LOAD_U32`, `REX_STORE_U32`, MMIO, etc.)
- [ ] Run the ReXGlue codegen tool (`rexglue`) with a sample XEX binary
- [ ] Inspect the generated `_init.h`, `_init.cpp`, and function `.cpp` files
- [ ] Build the generated project against the SDK
- [ ] Document the full codegen workflow in [docs/ReXGlue.md](docs/ReXGlue.md)
- [ ] Document the `REX_IMPORT` pattern for calling back into recompiled code
- [ ] Write a simple `REX_HOOK_RAW` example and understand `ctx` / `base` access
- [ ] Study `rexglue_configure_target()` and `rexglue_configure_module_target()` in CMake

---

## Phase 3 — Xbox 360 Architecture Research

> **Status: 📋 Planned**

The goal of Phase 3 is to build enough understanding of the Xbox 360 hardware and WWE '13's structure to guide the reverse engineering and hooking work ahead.

### Tasks

- [ ] Study the IBM Xenon CPU: 3 cores × 2 threads, PowerPC 2.02 ISA, 64-bit
- [ ] Understand the Xbox 360 memory map: RAM at 0x00000000, physical offsets, MMIO at 0x7F000000
- [ ] Understand the XEX2 executable format: header, sections, imports, exports
- [ ] Study the Xbox 360 calling convention: r3–r10 integer args, f1–f13 float args, r3/f1 return
- [ ] Learn about big-endian byte order and how `REX_LOAD_U32` handles the swap
- [ ] Understand what kernel modules WWE '13 imports (xboxkrnl.exe, xam.xex)
- [ ] Research which Altivec/VMX instructions are commonly used in WWE '13
- [ ] Load the WWE '13 XEX in Ghidra and configure the PPC processor
- [ ] Run Ghidra auto-analysis on the binary
- [ ] Export named function list using `scripts/ida/export_named_funcs.py` adapted for Ghidra
- [ ] Document all kernel imports found in the binary
- [ ] Identify the main entry point and startup sequence
- [ ] Identify graphics initialization code (look for Xenos command buffer writes)
- [ ] Identify audio initialization code (look for XMA / XAUDIO imports)
- [ ] Document findings in [docs/Research.md](docs/Research.md)
- [ ] Update [docs/Xbox360.md](docs/Xbox360.md) with WWE '13-specific notes

---

## Phase 4 — Project Generation and Binary Analysis

> **Status: 📋 Planned**

The goal of Phase 4 is to produce a working generated project that compiles successfully, even if the game does not run yet.

### Tasks

- [ ] Create the project `manifest.toml` for WWE '13
- [ ] Configure the XEX binary path and binary config in the manifest
- [ ] Run `rexglue` codegen tool on the WWE '13 XEX binary
- [ ] Verify all generated `.cpp` files compile without errors
- [ ] Set up the WWE '13 application class extending `rex::ReXApp`
- [ ] Configure the CMake project with `rexglue_configure_target()`
- [ ] Implement `REX_STUB` for all unresolved kernel imports
- [ ] Implement `REX_EXPORT_STUB` for all XAM and xboxkrnl exports
- [ ] Configure the VFS path mapping for game data
- [ ] Attempt first launch — capture crash / assertion logs
- [ ] Identify and triage the first critical crash site
- [ ] Document all stubs that need real implementations
- [ ] Set up Tracy profiler integration in Debug builds
- [ ] Add the debug overlay for frame stats via `SetGuestFrameStats()`
- [ ] Write a project generation walkthrough in [docs/BuildGuide.md](docs/BuildGuide.md)

---

## Phase 5 — Rendering

> **Status: 📋 Planned**

The goal of Phase 5 is to reach a state where the game renders visible output — even if only partially correct.

### Tasks

- [ ] Understand the Xenos GPU architecture: unified shader model, command buffer, registers
- [ ] Study `rex::graphics::CommandProcessor` and how it processes the Xenos PM4 packet stream
- [ ] Study `rex::graphics::RegisterFile` and the Xenos register table
- [ ] Configure the D3D12 graphics backend on Windows
- [ ] Configure the Vulkan graphics backend on Linux
- [ ] Implement `OnPreSetup()` hook to configure the graphics backend
- [ ] Identify the first draw call in the startup sequence via GPU trace
- [ ] Enable GPU trace capture (`rex::graphics::trace_writer`) and capture startup frames
- [ ] Replay the trace using `rex::graphics::trace_player` to isolate rendering issues
- [ ] Implement missing shader pipeline stages required for WWE '13
- [ ] Handle WWE '13 vertex buffer formats and texture formats
- [ ] Implement the 720p render target configuration
- [ ] Reach a state where the main menu renders
- [ ] Document the Xenos → D3D12/Vulkan translation layer in [docs/Architecture.md](docs/Architecture.md)
- [ ] Enable AMD FidelityFX integration for upscaling (optional, experimental)

---

## Phase 6 — Audio

> **Status: 📋 Planned**

The goal of Phase 6 is working audio playback, including music, commentary, and sound effects.

### Tasks

- [ ] Study the Xbox 360 XMA audio codec format
- [ ] Understand how `rex::audio::IAudioSystem` is initialized via `REX_AUDIO_BACKEND`
- [ ] Configure the SDL3 audio backend (`rex::audio::sdl::SDLAudioSystem`)
- [ ] Stub all XAudio2 kernel imports initially
- [ ] Implement XMA2 decoding for background music tracks
- [ ] Implement the game's audio streaming system (identify streaming buffers via Ghidra)
- [ ] Implement voice/commentary audio (compressed, streamed)
- [ ] Implement sound effects audio (short, uncompressed)
- [ ] Test audio synchronization with gameplay
- [ ] Document audio system implementation in [docs/Architecture.md](docs/Architecture.md)

---

## Phase 7 — Input

> **Status: 📋 Planned**

The goal of Phase 7 is working controller and keyboard input.

### Tasks

- [ ] Study the Xbox 360 XInput API used by WWE '13
- [ ] Configure the SDL3 input backend (`rex::input::sdl::SDLInputSystem`)
- [ ] Configure the XInput backend for Windows (`rex::input::xinput`)
- [ ] Stub all XINPUT kernel imports
- [ ] Map physical controller buttons to the game's input enumeration
- [ ] Implement `REX_HOOK` for `XInputGetState` and `XInputSetState`
- [ ] Test menu navigation with controller input
- [ ] Test gameplay input — movement, grapples, strikes
- [ ] Add keyboard / mouse input as an alternative (mouse-and-keyboard backend)
- [ ] Document input system wiring in [docs/Architecture.md](docs/Architecture.md)

---

## Phase 8 — Gameplay and Kernel Stubs

> **Status: 📋 Planned**

The goal of Phase 8 is a fully bootable game that runs a match end-to-end.

### Tasks

- [ ] Implement all remaining xboxkrnl.exe imports with real behavior
- [ ] Implement all remaining xam.xex imports with real behavior
- [ ] Implement Xbox 360 thread creation and management (`XThread`)
- [ ] Implement Xbox 360 synchronization primitives (`XEvent`, `XMutant`, `XSemaphore`)
- [ ] Implement Xbox 360 file I/O via the VFS layer
- [ ] Implement Xbox 360 memory allocation (`XMemAlloc`, `XMemFree`)
- [ ] Implement title storage and profile reading (STFS containers)
- [ ] Implement DLC content mounting if applicable
- [ ] Implement network stubs (for title update checks, online features)
- [ ] Reach main menu with all features functional
- [ ] Complete a single exhibition match end-to-end
- [ ] Document remaining stubs and known limitations
- [ ] Triage and fix critical crashes identified in Phases 5–7

---

## Phase 9 — Optimization and Stability

> **Status: 📋 Planned**

The goal of Phase 9 is a stable, performant executable suitable for regular use.

### Tasks

- [ ] Profile with Tracy to identify the top CPU hotspots in the recompiled code
- [ ] Investigate `REX_CONFIG_NON_VOLATILE_AS_LOCAL` and `REX_CONFIG_NON_ARGUMENT_AS_LOCAL` flags for context size reduction
- [ ] Investigate `REX_CONFIG_CTR_AS_LOCAL`, `REX_CONFIG_XER_AS_LOCAL`, `REX_CONFIG_CR_AS_LOCAL` for further reduction
- [ ] Profile GPU frame time — identify expensive draw calls or shader compilations
- [ ] Optimize shader cache loading (background compilation, persistent cache)
- [ ] Investigate 60 FPS unlock via frame pacing and vsync configuration
- [ ] Run UBSan builds and fix all undefined behavior reports
- [ ] Add automated crash reporting and log capture to `ReXApp`
- [ ] Test on AMD, NVIDIA, and Intel GPUs (D3D12 and Vulkan)
- [ ] Test on Linux (Ubuntu, Arch, Fedora) via the Vulkan backend
- [ ] Fix all known crashes in a 30-minute play session
- [ ] Optimize startup time (shader pre-compilation, VFS caching)
- [ ] Document optimization findings and configuration options

---

## Phase 10 — Release and Documentation

> **Status: 📋 Planned**

The goal of Phase 10 is a public release with complete documentation suitable for contributors and users.

### Tasks

- [ ] Tag version `1.0.0` following the ReXGlue versioning convention
- [ ] Write complete user setup guide (end-user, not developer focused)
- [ ] Write complete contributor guide for hooking new functions
- [ ] Document all `REX_HOOK` and `REX_STUB` implementations with explanations
- [ ] Create a wiki with quick-start instructions
- [ ] Create binary analysis walkthrough document (Ghidra → ReXGlue workflow)
- [ ] Publish GitHub Releases with Windows and Linux binaries
- [ ] Set up CI/CD workflows for automated builds on tag push
- [ ] Write a retrospective document describing what was learned
- [ ] Identify and document all known issues and missing features
- [ ] Plan post-1.0 roadmap: DLC support, modding API, multiplayer research

---

## Notes on Scope

This roadmap is ambitious. The technical challenges involved in Phase 5 (rendering) and Phase 8 (kernel stubs) are significant and will take considerable time. Each phase will be refined and expanded as knowledge grows.

The documentation phases (Research, Architecture, Glossary) are ongoing throughout all phases. Every discovery, dead end, and solution is worth documenting — this project is as much about the learning journey as the end result.
