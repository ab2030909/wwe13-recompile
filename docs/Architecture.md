# Project Architecture

> System design and component relationships for the WWE '13 Recompilation project.

This document describes how all the pieces fit together: the repository layout, the ReXGlue SDK layers, the generated project, and each runtime subsystem. Diagrams use Mermaid syntax and render on GitHub.

---

## Table of Contents

- [High-Level System Diagram](#high-level-system-diagram)
- [Repository Layout](#repository-layout)
- [ReXGlue SDK Layers](#rexglue-sdk-layers)
- [Generated Project Structure](#generated-project-structure)
- [Runtime Architecture](#runtime-architecture)
- [Rendering Pipeline](#rendering-pipeline)
- [Audio Architecture](#audio-architecture)
- [Input Architecture](#input-architecture)
- [Asset Management and VFS](#asset-management-and-vfs)
- [Build System Architecture](#build-system-architecture)
- [Testing Architecture](#testing-architecture)

---

## High-Level System Diagram

This diagram shows how all major components relate from the WWE '13 binary to the running application.

```mermaid
graph TB
    subgraph Analysis["Phase: Analysis (offline)"]
        XEX["WWE '13 XEX Binary\n(PowerPC code)"]
        GHIDRA["Ghidra\n(binary analysis)"]
        MANIFEST["manifest.toml\n(project config)"]
        XEX --> GHIDRA
        GHIDRA --> MANIFEST
    end

    subgraph Codegen["Phase: Code Generation (offline)"]
        TOOL["rexglue codegen tool"]
        GEN_INIT["src/generated/wwe13_init.h\nwwe13_init.cpp"]
        GEN_FUNCS["src/generated/wwe13_text.cpp\n(one per code section)"]
        MANIFEST --> TOOL
        XEX --> TOOL
        TOOL --> GEN_INIT
        TOOL --> GEN_FUNCS
    end

    subgraph Hooks["Phase: Hook Implementation (manual)"]
        HOOK_CODE["src/hooks/*.cpp\nREX_HOOK, REX_EXPORT"]
        STUB_CODE["src/stubs/*.cpp\nREX_STUB"]
        APP_CLASS["src/wwe13_app.h\nclass WWE13App : ReXApp"]
    end

    subgraph Build["Phase: Build"]
        CLANG["Clang 18+ Compiler"]
        REXSDK["ReXGlue Runtime SDK\n(rexruntime.dll)"]
        EXE["wwe13.exe"]
        GEN_INIT --> CLANG
        GEN_FUNCS --> CLANG
        HOOK_CODE --> CLANG
        STUB_CODE --> CLANG
        APP_CLASS --> CLANG
        REXSDK --> EXE
        CLANG --> EXE
    end

    subgraph Runtime["Phase: Runtime"]
        EXE --> RT["rex::Runtime\n(all subsystems)"]
        RT --> MEM["Memory\n4GB guest space"]
        RT --> KS["KernelState\nthreads, objects"]
        RT --> GFX["Graphics\nD3D12 / Vulkan"]
        RT --> AUD["Audio\nSDL3 / XMA"]
        RT --> INP["Input\nXInput / SDL3"]
        RT --> VFS["VFS\ngame: / d: mounts"]
    end
```

---

## Repository Layout

```
wwe13-recompilation/
│
├── rexglue-sdk/                  # ReXGlue SDK (git submodule — do not modify)
│   ├── include/rex/              # All public SDK headers
│   ├── src/                      # SDK implementation
│   ├── cmake/                    # CMake helpers
│   └── resources/templates/      # Jinja2 code generation templates
│
├── src/                          # Project source (added as project progresses)
│   ├── generated/                # ← Output of rexglue codegen (do not edit)
│   │   ├── wwe13_init.h          # Master include: macros, declarations
│   │   ├── wwe13_init.cpp        # PPCImageConfig + PPCFuncMappings table
│   │   ├── wwe13_text.cpp        # Generated function bodies
│   │   └── rexglue.cmake         # CMake source file registration
│   ├── hooks/                    # REX_HOOK implementations
│   │   ├── xinput_hooks.cpp      # XInputGetState, XInputSetState
│   │   ├── audio_hooks.cpp       # XAudio2 kernel exports
│   │   └── graphics_hooks.cpp    # Xenos command processor hooks (if needed)
│   ├── stubs/                    # REX_STUB implementations
│   │   ├── xboxkrnl_stubs.cpp    # xboxkrnl.exe unimplemented exports
│   │   └── xam_stubs.cpp         # xam.xex unimplemented exports
│   ├── wwe13_app.h               # Application class: class WWE13App : rex::ReXApp
│   ├── wwe13_app.cpp             # Application class implementation
│   └── main.cpp                  # REX_DEFINE_APP(wwe13, WWE13App::Create)
│
├── docs/                         # Project documentation
├── cmake/                        # Project-specific CMake modules (if needed)
├── scripts/                      # Utility scripts
├── out/                          # Build output (git-ignored)
│   └── win-amd64/
│       ├── Debug/
│       ├── Release/
│       └── RelWithDebInfo/
│
├── README.md
├── ROADMAP.md
├── SETUP.md
├── DEVELOPMENT.md
├── CONTRIBUTING.md
├── CHANGELOG.md
├── LICENSE
├── .gitignore
├── CMakeLists.txt                # (to be created in Phase 4)
└── CMakePresets.json             # (to be created in Phase 4)
```

---

## ReXGlue SDK Layers

The SDK is organized into three conceptual layers:

```mermaid
graph TB
    subgraph L3["Layer 3 — Application Framework"]
        REXAPP["rex::ReXApp\nLifecycle coordinator"]
        WINDOWED["rex::ui::WindowedApp\nPlatform window + event loop"]
    end

    subgraph L2["Layer 2 — Runtime Services"]
        RUNTIME["rex::Runtime\nSubsystem owner"]
        KERNEL["rex::system::KernelState\nKernel objects, threads"]
        MEMORY["rex::memory::Memory\n4GB virtual space"]
        VFS["rex::filesystem::VirtualFileSystem"]
        GFX["rex::system::IGraphicsSystem\nD3D12 or Vulkan"]
        AUDIO["rex::system::IAudioSystem\nSDL3 / XMA"]
        INPUT["rex::system::IInputSystem\nXInput / SDL3"]
    end

    subgraph L1["Layer 1 — PPC Core"]
        CONTEXT["PPCContext\nAll CPU registers"]
        FUNCMAP["PPCFuncMappings[]\nGuest → host function table"]
        HOOK["Hook API\nREX_HOOK / REX_STUB / REX_EXPORT"]
        MACROS["Memory Macros\nREX_LOAD_U32 / REX_STORE_U32"]
    end

    L3 --> L2
    L2 --> L1
```

Each layer depends only on layers below it. Your hook implementations sit between Layer 1 (calling conventions, memory access) and Layer 2 (kernel services).

---

## Generated Project Structure

The codegen tool produces a project that follows this structure:

```mermaid
graph LR
    MANIFEST["manifest.toml"] --> TOOL["rexglue tool"]

    TOOL --> INIT_H["wwe13_init.h\n• Config defines\n• SDK includes\n• Memory macros\n• DECLARE_REX_FUNC for all fns"]
    TOOL --> INIT_CPP["wwe13_init.cpp\n• PPCImageConfig struct\n• PPCFuncMappings[] table"]
    TOOL --> FUNC_CPP["wwe13_text.cpp\n(one per code section)\n• REX_FUNC bodies"]
    TOOL --> CMAKE["rexglue.cmake\n• target_sources() calls"]
```

Every `.cpp` function file includes `wwe13_init.h` at the top. This gives each translation unit access to all macros and declarations.

Example generated function (simplified):

```cpp
#include "generated/wwe13_init.h"

// Function at 0x82012ABC — originally: void UpdateEntityTransform(Entity* e)
DEFINE_REX_FUNC(sub_82012ABC) {
    REX_FUNC_PROLOGUE();
    // lwz r4, 0(r3)  — load entity pointer
    ctx.r4.u32 = REX_LOAD_U32(ctx.r3.u32);
    // addi r5, r4, 0x10  — pointer arithmetic
    ctx.r5.u32 = ctx.r4.u32 + 0x10;
    // stfs f1, 0(r5)  — store float
    REX_STORE_U32(ctx.r5.u32, *(u32*)&ctx.f1.f32);
    // blr
    return;
}
```

---

## Runtime Architecture

```mermaid
classDiagram
    class ReXApp {
        +OnPreSetup(config)
        +OnPostSetup()
        +OnConfigurePaths(paths)
        +OnFinalizePaths(defaults, resume)
        +OnPostLoadXexImage()
        +OnPreLaunchModule()
        +OnPostLaunchModule(thread)
        -SetupEnvironment()
        -ConstructRuntime(paths)
        -SetupPresentation()
        -LaunchModule()
    }

    class Runtime {
        +Setup(image_info, config)
        +LoadXexImage(path)
        +PrepareModuleLaunch()
        +LaunchModule()
        +virtual_membase() uint8_t*
        -memory_ Memory
        -kernel_state_ KernelState
        -graphics_system_ IGraphicsSystem
        -audio_system_ IAudioSystem
        -input_system_ IInputSystem
        -file_system_ VirtualFileSystem
    }

    class KernelState {
        +kernel_memory() Memory*
        +object_table() XObjectTable*
        +ResolveExport(module, ordinal) PPCFunc*
        -threads_
        -exports_
    }

    class Memory {
        +virtual_membase() uint8_t*
        +Alloc(size, alignment) uint32_t
        +Free(guest_addr)
        -virtual_base_ uint8_t*
    }

    ReXApp --> Runtime
    Runtime --> KernelState
    Runtime --> Memory
    Runtime --> IGraphicsSystem
    Runtime --> IAudioSystem
    Runtime --> IInputSystem
    Runtime --> VirtualFileSystem
    KernelState --> Memory
```

---

## Rendering Pipeline

The path from Xbox 360 game code to a rendered frame on screen:

```mermaid
sequenceDiagram
    participant Game as Game Code (generated)
    participant MMIO as MMIOHandler
    participant CP as CommandProcessor
    participant Xenos as Xenos Register State
    participant Backend as D3D12 / Vulkan Backend
    participant Screen as Window

    Game->>MMIO: REX_MM_STORE_U32(0x7F000000 + reg, value)
    MMIO->>CP: CheckStore(reg, value)
    CP->>Xenos: Update RegisterFile
    CP->>CP: Process PM4 packet
    Note over CP: Decode draw call, shaders, state

    Game->>MMIO: REX_MM_STORE_U32(PM4_XE_SWAP)
    MMIO->>CP: EndOfFrame
    CP->>Backend: Submit draw commands
    Backend->>Backend: Compile/lookup shaders
    Backend->>Backend: Execute command list
    Backend->>Screen: Present frame
```

### Key Components

| Component | Header | Description |
|-----------|--------|-------------|
| `CommandProcessor` | `graphics/command_processor.h` | Reads PM4 packets from the ring buffer |
| `RegisterFile` | `graphics/register_file.h` | Stores all Xenos GPU register values |
| `SharedMemory` | `graphics/shared_memory.h` | Manages GPU-visible host memory |
| `PrimitiveProcessor` | `graphics/primitive_processor.h` | Handles vertex format translation |
| D3D12 Backend | `graphics/d3d12/` | Windows-only rendering backend |
| Vulkan Backend | `graphics/vulkan/` | Cross-platform rendering backend |

---

## Audio Architecture

```mermaid
graph LR
    GAME["Game Code\n(XAudio2 / XMA imports)"]
    KERNEL["KernelState\nXAudio2 export hooks"]
    AUDIO_SYS["IAudioSystem"]
    SDL["SDL3 Audio Backend"]
    XMA["XMA Decoder"]
    HOST_AUDIO["Host Audio Device"]

    GAME --> KERNEL
    KERNEL --> AUDIO_SYS
    AUDIO_SYS --> XMA
    XMA --> SDL
    SDL --> HOST_AUDIO
```

Audio hooks intercept XAudio2 source voice and mastering voice creation calls. Compressed XMA audio data from the XEX is decoded by an XMA decoder and fed to the SDL3 audio backend.

---

## Input Architecture

```mermaid
graph LR
    GAME["Game Code\n(XInputGetState calls)"]
    HOOK["REX_EXPORT(XInputGetState)"]
    INPUT_SYS["IInputSystem"]
    XINPUT["XInput Backend\n(Windows)"]
    SDL_INPUT["SDL3 Backend\n(cross-platform)"]
    CONTROLLER["Physical Controller"]

    GAME --> HOOK
    HOOK --> INPUT_SYS
    INPUT_SYS --> XINPUT
    INPUT_SYS --> SDL_INPUT
    XINPUT --> CONTROLLER
    SDL_INPUT --> CONTROLLER
```

XInput hook implementations read the physical controller state and write it into a guest-memory `XINPUT_GAMEPAD` structure that the game code can read. The structure must be written in big-endian byte order.

---

## Asset Management and VFS

```mermaid
graph TB
    GAME["Game Code\nNtCreateFile calls"]
    VFS["VirtualFileSystem\ngame: / d: / user: / update:"]
    GAME_DATA["game_data/\n(host filesystem)"]
    USER_DATA["user_data/\n(save data)"]
    UPDATE_DATA["update_data/\n(title updates)"]

    GAME --> VFS
    VFS --> |game: or d:| GAME_DATA
    VFS --> |user:| USER_DATA
    VFS --> |update:| UPDATE_DATA
```

The VFS translates Xbox 360 virtual paths to host filesystem paths:

| Xbox 360 Path | Host Path |
|--------------|-----------|
| `game:\default.xex` | `game_data/default.xex` |
| `d:\media\music.xma` | `game_data/media/music.xma` |
| `user:\profile.bin` | `user_data/profile.bin` |

Path configuration is done in `WWE13App::OnConfigurePaths(PathConfig& paths)`.

---

## Build System Architecture

```mermaid
graph LR
    PRESETS["CMakePresets.json\nwin-amd64 / linux-amd64"]
    CMAKE["CMakeLists.txt"]
    FIND_PKG["find_package(rexglue)"]
    SDK_HELPERS["rexglue_configure_target()\nrexglue_configure_module_target()"]
    NINJA["Ninja Multi-Config"]
    DEBUG["Debug/"]
    RELEASE["Release/"]
    RWD["RelWithDebInfo/"]

    PRESETS --> CMAKE
    CMAKE --> FIND_PKG
    FIND_PKG --> SDK_HELPERS
    SDK_HELPERS --> NINJA
    NINJA --> DEBUG
    NINJA --> RELEASE
    NINJA --> RWD
```

### Build Configurations

| Configuration | Optimizations | Debug Info | Tracy | Use Case |
|--------------|--------------|------------|-------|---------|
| Debug | None (`-O0`) | Full | Enabled | Development, step debugging |
| Release | Full (`-O3`, `-DNDEBUG`) | None | Disabled | Distribution |
| RelWithDebInfo | `-O2` | Full | Enabled | Profiling with Tracy |

---

## Testing Architecture

### PPC Instruction Tests (SDK level)

The ReXGlue SDK includes a PPC instruction test harness. These tests verify individual instruction translations are correct. They are enabled in the SDK build with `REXGLUE_BUILD_TESTS=ON`.

### Manual Regression Testing (project level)

At the project level, testing is observational:

| Phase | Pass Criteria |
|-------|-------------|
| Phase 4 | Application starts; no missing symbol assertions |
| Phase 5 | A frame appears on screen |
| Phase 6 | Audio device opens; no audio crash |
| Phase 7 | Controller navigation works |
| Phase 8 | Main menu loads; match starts |
| Phase 9 | 30 minutes of gameplay without crash |
| Phase 10 | Full game playable |

### Logging as a Test Signal

ReXGlue's logging system is the primary diagnostic tool. The `REXKRNL_WARN` messages from stubs tell you exactly which functions are being called and need implementation. A log file with zero stub warnings is a strong indicator of completion.

### Tracy Profiling

In `RelWithDebInfo` builds, Tracy profiler captures CPU timelines. Look for:
- Functions spending disproportionate time (unoptimized hot paths)
- Unexpected function call frequency (wrong frame rate, physics ticking too fast)
- Lock contention (thread synchronization issues)
