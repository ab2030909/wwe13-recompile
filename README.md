# WWE '13 Recompilation

> A complete, professionally documented static recompilation project for the Xbox 360 game **WWE '13**, built with the [ReXGlue SDK](https://github.com/rexglue/rexglue-sdk).

[![Status](https://img.shields.io/badge/status-early%20development-orange)](ROADMAP.md)
[![ReXGlue](https://img.shields.io/badge/ReXGlue-SDK%200.8.x-blue)](rexglue-sdk/README.md)
[![License](https://img.shields.io/badge/license-BSD%203--Clause-green)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux-lightgrey)](docs/BuildGuide.md)

---

## Overview

This repository documents the complete journey of statically recompiling **WWE '13** (Xbox 360, THQ / Yuke's, 2012) into a native PC executable using the ReXGlue SDK. The project targets both Windows (Direct3D 12) and Linux (Vulkan).

Static recompilation is fundamentally different from emulation. Instead of simulating the Xbox 360 hardware at runtime, the game's original PowerPC binary is translated into portable C++23 source code ahead of time. That generated C++ is then compiled and run natively — no interpreter, no JIT, no emulator loop.

This is an educational open-source project. It documents every step: environment setup, binary analysis, code generation, runtime wiring, and system-level reverse engineering.

---

## Purpose

- Learn and document static recompilation from the ground up.
- Produce a working native binary for WWE '13 on modern platforms.
- Create reference documentation that future contributors and learners can follow.
- Build a professional-quality open-source project that demonstrates real software engineering practices.

---

## Goals

| Goal | Description |
|------|-------------|
| **Understand ReXGlue** | Master the SDK API, codegen pipeline, and hook system |
| **Analyze WWE '13** | Reverse engineer the XEX binary, identify functions, stubs, and kernel imports |
| **Generate C++ code** | Use the ReXGlue codegen tool to produce compilable C++ from the binary |
| **Implement hooks** | Replace kernel calls and game subsystems with working native implementations |
| **Rendering** | Wire the Xenos GPU command stream to D3D12 / Vulkan |
| **Audio** | Implement XMA audio decoding and playback |
| **Input** | Map XInput / SDL controller input to the game's input system |
| **Run the game** | Reach a bootable, playable state on Windows and Linux |

---

## Current Status

> **Phase 1 — Environment Setup** ✅ Complete

| Component | Status |
|-----------|--------|
| Visual Studio 2022 | ✅ Installed |
| LLVM / Clang 18+ | ✅ Installed |
| Ninja | ✅ Installed |
| CMake 3.25+ | ✅ Installed |
| Python 3.x | ✅ Installed |
| Git | ✅ Installed |
| Ghidra | ✅ Installed |
| ReXGlue SDK (submodule) | ✅ Present at `rexglue-sdk/` |
| Project documentation | ✅ Initial structure created |

> **Next:** Generate the ReXGlue project, load the WWE '13 XEX binary, and begin binary analysis.

---

## Repository Structure

```
wwe13-recompilation/
├── rexglue-sdk/            # ReXGlue SDK (git submodule)
├── docs/                   # All project documentation
│   ├── Day01.md            # Development journal — Day 1
│   ├── Architecture.md     # Project architecture and system design
│   ├── ReXGlue.md          # ReXGlue SDK deep-dive reference
│   ├── Xbox360.md          # Xbox 360 hardware and platform reference
│   ├── Research.md         # Research notes, links, and experiments
│   ├── BuildGuide.md       # Advanced build and configuration guide
│   ├── Troubleshooting.md  # Common problems and solutions
│   └── Glossary.md         # Beginner-friendly terminology reference
├── README.md               # This file
├── ROADMAP.md              # Long-term project roadmap
├── SETUP.md                # Complete environment setup guide
├── DEVELOPMENT.md          # Developer handbook and coding standards
├── CONTRIBUTING.md         # How to contribute to this project
├── CHANGELOG.md            # Version history
├── LICENSE                 # BSD 3-Clause License
└── .gitignore              # Git ignore rules
```

> Source code, generated files, and hook implementations will be added as the project progresses. See [ROADMAP.md](ROADMAP.md) for the full plan.

---

## Development Environment

This project requires the following tools. See [SETUP.md](SETUP.md) for complete installation instructions.

| Tool | Version | Purpose |
|------|---------|---------|
| Visual Studio 2022 | 17.x | Windows compiler toolchain, debugger, IDE |
| LLVM / Clang | 18+ | Required compiler for ReXGlue (strictly enforced) |
| Ninja | Latest | Fast build system used by CMake presets |
| CMake | 3.25+ | Build system generator |
| Python | 3.10+ | Scripting, tooling |
| Git | Latest | Version control |
| Ghidra | Latest | Binary analysis and reverse engineering |
| ReXGlue SDK | 0.8.x | Xbox 360 static recompilation toolkit |

---

## Technologies Used

### ReXGlue SDK

ReXGlue converts Xbox 360 PowerPC binaries into C++23 source code. It provides:

- **Codegen pipeline** — analyzes the XEX binary and generates function-per-function C++ source
- **Runtime SDK** — emulates the Xbox 360 kernel, Xenos GPU, XMA audio, and XInput
- **Hook API** — type-safe function interception using `REX_HOOK`, `REX_STUB`, `REX_IMPORT`
- **Application framework** — `rex::ReXApp` base class wiring everything together

### Xbox 360 Architecture

WWE '13 runs on Xbox 360 hardware:

- **CPU**: IBM Xenon — three PowerPC cores, each dual-threaded (6 hardware threads total)
- **GPU**: ATI Xenos — custom unified shader architecture
- **Memory**: 512 MB GDDR3 shared between CPU and GPU
- **Executable format**: XEX2 (Xbox Executable version 2)

### Build System

- CMake 3.25+ with Ninja Multi-Config generator
- Clang 18+ with C++23 standard
- Targets: Windows AMD64 (D3D12), Linux AMD64/ARM64 (Vulkan)

---

## Learning Objectives

This project is explicitly a learning journey. The documentation is written for beginners in reverse engineering and recompilation. By following this repository you will learn:

- What static recompilation is and how it differs from emulation
- The Xbox 360 PowerPC ABI, register file, and instruction set
- How the ReXGlue SDK's codegen pipeline works end-to-end
- How to analyze a binary with Ghidra and export symbol information
- How to hook kernel functions and implement platform-specific stubs
- How the Xenos GPU command processor works at a high level
- How CMake, Ninja, and Clang work together on a large C++ project

---

## Documentation

| Document | Description |
|----------|-------------|
| [SETUP.md](SETUP.md) | Complete environment setup guide with verification steps |
| [DEVELOPMENT.md](DEVELOPMENT.md) | Developer handbook: workflow, standards, branching, testing |
| [ROADMAP.md](ROADMAP.md) | Long-term project roadmap with phase checklists |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute: branches, commits, pull requests |
| [CHANGELOG.md](CHANGELOG.md) | Version history in Keep a Changelog format |
| [docs/Day01.md](docs/Day01.md) | Development journal — Day 1 setup notes |
| [docs/Architecture.md](docs/Architecture.md) | Project architecture with Mermaid diagrams |
| [docs/ReXGlue.md](docs/ReXGlue.md) | ReXGlue SDK deep-dive reference |
| [docs/Xbox360.md](docs/Xbox360.md) | Xbox 360 hardware and platform reference |
| [docs/Research.md](docs/Research.md) | Research notes and external resources |
| [docs/BuildGuide.md](docs/BuildGuide.md) | Advanced build and debugging guide |
| [docs/Troubleshooting.md](docs/Troubleshooting.md) | Troubleshooting handbook |
| [docs/Glossary.md](docs/Glossary.md) | Beginner-friendly glossary of all key terms |

---

## Roadmap

The project is organized into ten phases. See [ROADMAP.md](ROADMAP.md) for the complete checklist.

| Phase | Title | Status |
|-------|-------|--------|
| 1 | Environment Setup | ✅ Complete |
| 2 | ReXGlue SDK Mastery | 🔄 In Progress |
| 3 | Xbox 360 Architecture Research | 📋 Planned |
| 4 | Project Generation & Binary Analysis | 📋 Planned |
| 5 | Rendering (Xenos → D3D12/Vulkan) | 📋 Planned |
| 6 | Audio (XMA decoding) | 📋 Planned |
| 7 | Input (XInput / SDL) | 📋 Planned |
| 8 | Gameplay & Kernel Stubs | 📋 Planned |
| 9 | Optimization & Stability | 📋 Planned |
| 10 | Release & Documentation | 📋 Planned |

---

## Future Plans

Beyond reaching a bootable state, longer-term goals include:

- Comprehensive hook documentation for all kernel imports used by WWE '13
- Save game compatibility via VFS path mapping
- DLC and title update support
- Performance optimization using Tracy profiler integration
- 60 FPS unlock via frame pacing research
- Modding toolchain documentation

---

## Disclaimer

This project is an independent educational endeavor. It is not affiliated with or endorsed by **THQ**, **2K Games**, **Yuke's**, **Microsoft**, **Xbox**, or any other rights holder.

This repository contains no game files, no ROM data, and no proprietary content. You must legally own a copy of WWE '13 for the Xbox 360 to use this project.

ReXGlue itself carries a similar disclaimer: it is not affiliated with Microsoft or Xbox and is intended for educational and development purposes only.

This project does not promote piracy or unauthorized use of copyrighted material.

---

## Frequently Asked Questions

**Q: Is this an emulator?**
No. Static recompilation translates the game's PowerPC machine code into C++ source ahead of time. The compiled result runs natively on your CPU. No Xbox 360 hardware is simulated at runtime.

**Q: Do I need an Xbox 360 to use this?**
No Xbox 360 console is required. You do need to legally own the game to obtain the XEX binary.

**Q: Will this work on Windows and Linux?**
Yes. ReXGlue targets Windows (D3D12) and Linux (Vulkan), on both AMD64 and ARM64.

**Q: How is this different from RPCS3 or Xenia?**
Xenia and RPCS3 are emulators — they simulate the hardware in real time. This project compiles the game code to native C++ once. The result is faster and more portable.

**Q: Can I contribute?**
Absolutely. Read [CONTRIBUTING.md](CONTRIBUTING.md) for the full guide. Beginners are welcome — detailed documentation is a contribution too.

**Q: Is the game fully playable yet?**
No. This is early-stage development. See [ROADMAP.md](ROADMAP.md) for the current state.

**Q: What C++ standard is used?**
C++23, compiled with Clang 18+. The ReXGlue SDK strictly enforces this.

---

*Built with ❤️ using the [ReXGlue SDK](https://github.com/rexglue/rexglue-sdk) — Xbox 360 static recompilation for modern platforms.*
