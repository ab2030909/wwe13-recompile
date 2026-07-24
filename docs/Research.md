# Research Notes

> A living research notebook for the WWE '13 Recompilation project. This document collects external references, analysis findings, open questions, and experiments. It is updated continuously throughout the project.

---

## Table of Contents

- [Key Repositories](#key-repositories)
- [Documentation and Articles](#documentation-and-articles)
- [Research Papers](#research-papers)
- [Useful Videos](#useful-videos)
- [Xbox 360 Community Resources](#xbox-360-community-resources)
- [WWE 13 Binary Analysis](#wwe-13-binary-analysis)
- [Experiments](#experiments)
- [Open Questions](#open-questions)
- [Ideas and Hypotheses](#ideas-and-hypotheses)
- [Future Research](#future-research)

---

## Key Repositories

### ReXGlue Ecosystem

| Repository | Description | Relevance |
|-----------|-------------|-----------|
| [rexglue/rexglue-sdk](https://github.com/rexglue/rexglue-sdk) | The SDK we are using | Primary dependency |
| [hedge-dev/XenonRecomp](https://github.com/hedge-dev/XenonRecomp) | The modern static recompiler that inspired ReXGlue | Architecture reference, instruction translation reference |
| [rexdex/recompiler](https://github.com/rexdex/recompiler) | The original Xbox 360 static recompiler | Historical reference |

### Xbox 360 Emulation

| Repository | Description | Relevance |
|-----------|-------------|-----------|
| [xenia-project/xenia](https://github.com/xenia-project/xenia) | Xbox 360 emulator. ReXGlue's kernel/GPU layer is built on Xenia's foundations. | Kernel implementation reference, Xenos GPU reference |
| [xenia-project/xenia-canary](https://github.com/xenia-canary/xenia-canary) | Active fork of Xenia with additional fixes | Alternative reference for kernel behavior |

### PPC / Xbox 360 Reference

| Repository | Description | Relevance |
|-----------|-------------|-----------|
| [nicowillis/xbox360-powerpc-ref](https://github.com/nicowillis/xbox360-powerpc-ref) | PowerPC reference for Xbox 360 | Instruction set reference |
| [Free60 Project](https://free60.org) | Xbox 360 homebrew documentation wiki | XEX format, hardware documentation |

---

## Documentation and Articles

### PowerPC Architecture

- **IBM PowerPC Architecture Book** — The official ISA specification. Available from IBM's developer resources. Covers all instructions, registers, and the memory model.
- **PowerPC 2.02 Architecture**: The specific version used by the Xbox 360 Xenon CPU. Includes the VMX128 extensions.
- **[Xbox 360 CPU Whitepaper](https://www.ibm.com/downloads/cas/LBQBBBDN)** — IBM's technical overview of the Xenon chip design.

### Xenos GPU

- **[ATI Xenos: Direct3D on Xbox 360](https://www.beyond3d.com/content/articles/4/)** — Beyond3D technical breakdown of the Xenos architecture. Essential reading for understanding the unified shader model and eDRAM.
- **[Xenos GPU Architecture (David Blythe, SIGGRAPH 2006)](https://www.ati.com/developer/xbox360/siggraph2006.pdf)** — ATI's own SIGGRAPH presentation on Xenos. Covers the tiling architecture, shader processing, and eDRAM.
- Xenia source (`src/xenia/gpu/`) — The most complete open-source implementation of Xenos GPU command processing. Study the PM4 packet handler and register table.

### XEX Format

- **[Free60 XEX Documentation](https://free60.org/wiki/XEX)** — Community-documented XEX format with field descriptions.
- **[Xenia XEX loader](https://github.com/xenia-project/xenia/blob/master/src/xenia/cpu/xex_module.cc)** — The most complete open-source XEX parser and loader. Reference for import resolution and section mapping.

### Static Recompilation Theory

- **"N64 to C" — the original static recompiler approach** — Documents the general technique of function-granularity static recompilation.
- **[Static Recompilation of Xbox Binaries](https://github.com/hedge-dev/XenonRecomp)** — XenonRecomp's README documents the specific challenges of Xbox 360 static recompilation.

---

## Research Papers

The following academic papers are relevant to the technical foundations of static recompilation:

- **"UQBT: Adaptable Binary Translation at Low Cost"** — Cristina Cifuentes, Mike Van Emmerik. Describes a framework for binary translation including handling of indirect calls and data-as-code.
- **"Scalable and Precise Static Analysis of Binary Code"** — Discusses function boundary recovery and indirect control flow analysis in binaries.
- **"Reverse Compilation Techniques"** — Cristina Cifuentes, PhD thesis. Foundational theory of decompilation and code recovery.

> These papers are academic background. For practical work, the Xenia and XenonRecomp source code is more directly applicable.

---

## Useful Videos

### ReXGlue and Related Projects

| Resource | Description |
|----------|-------------|
| ReXGlue Discord | Active community with development updates and working examples of recompiled titles |
| XenonRecomp GitHub Discussions | Technical discussions about specific recompilation challenges |

### Xbox 360 Reverse Engineering

- Search YouTube for "Xbox 360 homebrewing" and "Xbox 360 reverse engineering" — the Free60 community produced tutorials on XEX loading and kernel development.
- GDC Vault contains several Xbox 360 game engine talks from THQ and Yuke's from 2008–2012 that may reveal architectural details of the WWE game engine.

### PowerPC Assembly

- IBM's developer resources include tutorial videos on PowerPC assembly.
- University course recordings on RISC architecture provide good background on the in-order execution model.

---

## Xbox 360 Community Resources

| Resource | URL | Description |
|----------|-----|-------------|
| Free60 Wiki | https://free60.org/wiki/ | Xbox 360 homebrew wiki. XEX format, kernel tables, hardware documentation |
| ReXGlue Discord | https://discord.gg/CNTxwSNZfT | Active community for ReXGlue development |
| Xbox 360 Kernel Export Table | Available on Free60 | Complete list of xboxkrnl.exe and xam.xex exports with ordinal numbers |

---

## WWE '13 Binary Analysis

> This section will be filled in during Phase 3 (Binary Analysis). Initial entries are placeholders for research to be conducted.

### Binary Facts

| Property | Value | Notes |
|---------|-------|-------|
| Title | WWE '13 | |
| Developer | Yuke's | |
| Publisher | THQ | |
| Platform | Xbox 360 | |
| Year | 2012 | |
| Game Engine | Yuke's proprietary engine | Shared with WWE '12, WWE '14 (modified) |
| XEX image base | TBD | To be confirmed via Ghidra |
| XEX image size | TBD | To be confirmed via Ghidra |
| Code section size | TBD | To be confirmed |

### Kernel Imports

> To be populated during Phase 3 analysis.

| Module | Count | Key Functions |
|--------|-------|--------------|
| xboxkrnl.exe | TBD | TBD |
| xam.xex | TBD | TBD |

### Function Count Estimate

> To be confirmed after running the codegen tool.

Static recompilation projects for similar-era games (Sonic Unleashed, Shenmue III ports) have produced 50,000–200,000 functions. WWE '13 is expected to be in a similar range.

---

## Experiments

### Experiment 1 — SDK Build Verification (Day 1) ✅ Complete

**Goal:** Confirm the ReXGlue SDK builds correctly with Clang 18 on Windows.

**Method:** Run `cmake --preset win-amd64 && cmake --build out/build/win-amd64 --config Debug`.

**Result:** Build succeeded. SDK libraries and binaries present in `out/win-amd64/Debug/`.

**Notes:** Build time was approximately 8 minutes on first run. Subsequent incremental builds are fast.

---

### Experiment 2 — First Codegen Run (Planned)

**Goal:** Run the ReXGlue codegen tool on the WWE '13 XEX and inspect the generated output.

**Method:** Create a minimal `manifest.toml`, run `rexglue`, inspect generated files.

**Expected:** Several `.cpp` files containing recompiled functions. `wwe13_init.h` with macros.

**Status:** Planned for Phase 4.

---

### Experiment 3 — First Build (Planned)

**Goal:** Build the generated project with stub hooks and observe the startup behavior.

**Method:** Wire up stubs for all kernel imports, build, run, capture the log.

**Expected:** Application starts, loads XEX, crashes or logs stubs being called.

**Status:** Planned for Phase 4.

---

## Open Questions

These questions do not have answers yet. They are tracked here to guide future research.

### Architecture Questions

1. **How does the codegen tool detect function boundaries in WWE '13?** Does it use recursive disassembly, linear sweep, or a hybrid? Are there known problem areas (e.g., computed branch tables)?

2. **Are there any self-modifying code patterns in WWE '13?** The Yuke's engine is known to use code hot-patching for debug builds — is this present in the retail XEX?

3. **How many indirect calls (CTR dispatch) does WWE '13 use?** A high number of indirect calls means a large `PPCFuncMappings[]` table and risk of missing addresses.

4. **Does WWE '13 use multiple XEX modules (DLLs)?** Many Xbox 360 games split functionality into satellite XEX files. If so, `RegisterModulesFunc` and `register_modules` in `PPCImageInfo` will be needed.

### Rendering Questions

5. **What render target resolution does WWE '13 use internally?** Most Xbox 360 games render at a sub-native resolution (e.g., 1280×720, or sub-HD with upscaling). The eDRAM size limits the render target area.

6. **Does WWE '13 use MSAA?** The Xenos eDRAM enables "free" 2× or 4× MSAA for the render target. What setting does WWE '13 use?

7. **How are character shaders structured?** WWE '13 uses a lot of skin rendering. Is it using a standard Blinn-Phong model or something more complex like screen-space subsurface scattering?

### Audio Questions

8. **Does WWE '13 use XMA or XMA2 audio?** XMA2 is the later version with better seeking support. The codec matters for implementing playback correctly.

9. **Are sound effects pre-buffered or streamed on demand?** This affects the audio system initialization complexity.

### Kernel Questions

10. **How does WWE '13 handle save data?** Does it use standard STFS containers (user profile) or a custom file format?

11. **Does WWE '13 use Xbox LIVE services?** If so, what happens when they are unavailable? Will the game boot in offline mode?

---

## Ideas and Hypotheses

### Hypothesis: WWE '13 Uses a Fixed Thread Layout

Based on the Xenon CPU's 3-core design, WWE '13 likely follows a pattern common in Yuke's engine games:
- Core 0: Main game thread (logic, physics, input)
- Core 1: Rendering thread (command buffer submission)
- Core 2: Streaming thread (audio, asset loading)

If this is confirmed via Ghidra analysis, we can configure ReXGlue's thread affinity to match and potentially improve performance.

### Idea: Port Save Game Compatibility

The game's save data uses Xbox 360 STFS containers bound to an Xbox LIVE profile. It may be possible to unpack these containers (using a tool like Horizon) and convert them to flat files readable by the recompilation's VFS. This would allow people to continue saves from their Xbox 360 hardware.

### Idea: 60 FPS Unlock

WWE '13 runs at 30 FPS on Xbox 360. The framerate is likely locked either in a vsync polling loop or an explicit `KeDelayExecutionThread` call. If the delay can be found and bypassed, and if the physics timestep can be adjusted to match, 60 FPS may be achievable. This is a known successful technique in other recompilation projects.

---

## Future Research

Topics to investigate in later phases:

- **Yuke's engine internals** — GDC talks, interviews with former Yuke's developers, and community modding resources may shed light on internal engine architecture.
- **WWE '14 / WWE 2K15 comparison** — These later games use the same engine base. Comparing import tables and function layouts may help identify common subsystems.
- **Collision and physics system** — WWE '13 has a complex physics system for grapple interactions. Understanding how it maps to Altivec operations will be important for correctness.
- **Network protocol** — If online features are desirable in the future, the XLive / Xbox LIVE communication protocol used by WWE '13 needs analysis.
- **Audio compression research** — XMA/XMA2 codec implementation options: use ffmpeg's XMA decoder, or investigate the `mspack` library (already vendored in ReXGlue).
