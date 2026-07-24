# Xbox 360 Platform Reference

> A beginner-friendly technical reference for the Xbox 360 hardware, software, and executable format, written specifically for recompilation work.

---

## Table of Contents

- [Overview](#overview)
- [CPU — IBM Xenon](#cpu--ibm-xenon)
- [PowerPC Architecture](#powerpc-architecture)
- [Register File](#register-file)
- [Calling Convention](#calling-convention)
- [Memory System](#memory-system)
- [GPU — ATI Xenos](#gpu--ati-xenos)
- [Executable Format — XEX2](#executable-format--xex2)
- [Kernel Modules](#kernel-modules)
- [File Formats](#file-formats)
- [Boot Process](#boot-process)
- [Debugging on Xbox 360](#debugging-on-xbox-360)
- [Why Recompilation is Difficult](#why-recompilation-is-difficult)

---

## Overview

The Xbox 360 (codename "Xenon") was released by Microsoft in November 2005. It was a significant departure from the Intel x86 architecture of desktop PCs — it used a custom IBM PowerPC processor, a custom ATI GPU, and a shared memory architecture uncommon in consumer hardware at the time.

Understanding these differences is essential for recompilation work: the tools and assumptions you have from PC programming do not directly transfer.

| Property | Xbox 360 | Modern PC |
|----------|---------|-----------|
| CPU Architecture | PowerPC (big-endian) | x86-64 (little-endian) |
| CPU Cores | 3 cores × 2 threads = 6 hardware threads | 8–32+ cores |
| CPU Clock | 3.2 GHz (each core) | 3.0–5.0 GHz |
| RAM | 512 MB GDDR3 (shared CPU/GPU) | 8–64 GB DDR5 (separate GPU RAM) |
| GPU | ATI Xenos (unified shader) | NVIDIA / AMD (discrete) |
| Storage | DVD-ROM + HDD | NVMe SSD |
| OS | Xbox 360 OS (proprietary, based on Windows kernel) | Windows / Linux |

---

## CPU — IBM Xenon

The CPU in the Xbox 360 is the **IBM Xenon**, a custom chip co-designed by IBM, Microsoft, and Sony (the same chip family powers the PlayStation 3's SPU complex). It is based on IBM's PowerPC 970 (G5) design.

### Key Properties

- **3 physical cores**, each running at **3.2 GHz**
- Each core supports **2 hardware threads** (simultaneous multithreading), giving **6 logical threads** total
- **64-bit PowerPC 2.02** instruction set architecture (ISA)
- **Big-endian** byte order (see [Endianness](#endianness) below)
- **In-order execution** (unlike modern out-of-order x86 CPUs)
- L1 cache: 32 KB per core; L2 cache: 1 MB shared

### What "In-Order" Means

x86 CPUs reorder instructions at runtime to keep execution units busy. PowerPC Xenon executes instructions strictly in program order. This was a deliberate design choice for predictability in a game console. It also means the compiler is more responsible for instruction scheduling to avoid pipeline stalls.

For recompilation: the generated C++ runs on an out-of-order x86 CPU, which is generally fine — the semantics are preserved, and x86's reordering only improves performance without changing results. However, memory ordering assumptions in the original code may need careful attention for multi-threaded code.

### Altivec / VMX

Each Xenon core includes an **Altivec** (also called VMX — Vector Multimedia Extension) unit. This is a 128-bit SIMD unit capable of processing 4 floats, 8 shorts, or 16 bytes in a single instruction. WWE '13 uses Altivec extensively for physics, animation, and audio processing.

The Xbox 360 also supports **VMX128**, an extended instruction set with 128 vector registers (v0–v127) rather than Altivec's standard 32. This is documented in `rexglue-sdk/docs/ppc/vmx128.txt`.

---

## PowerPC Architecture

### What PowerPC Is

PowerPC (PPC) is a RISC (Reduced Instruction Set Computer) architecture developed by the Apple-IBM-Motorola alliance in 1991. RISC architectures use a small set of simple instructions that each execute in one clock cycle, relying on the compiler to schedule them well.

This contrasts with CISC (Complex Instruction Set Computer) architectures like x86, where individual instructions can do complex multi-step operations. x86 has hundreds of instructions with irregular encoding; PPC has a smaller, more uniform set.

### Instruction Encoding

All PowerPC instructions are **32 bits wide** and **word-aligned** (aligned to 4-byte boundaries). This uniformity makes disassembly straightforward: you always know where instructions start, unlike x86 where instructions have variable length (1–15 bytes).

```
PPC instruction (32 bits):
[OPCODE 6 bits][operands vary by instruction type]
```

### Key Instruction Families

| Family | Examples | Description |
|--------|---------|-------------|
| Integer arithmetic | `add`, `sub`, `mullw`, `divw` | Integer math, register-to-register |
| Logical | `and`, `or`, `xor`, `nor` | Bit operations |
| Shift/rotate | `slw`, `srw`, `rlwinm` | Bit shifting |
| Load/store | `lwz`, `stw`, `lhz`, `stb` | Memory access |
| Branch | `b`, `bl`, `blr`, `bctr` | Control flow |
| Compare | `cmpw`, `cmplw`, `cmpwi` | Sets Condition Register |
| Float | `fadd`, `fmul`, `fdiv`, `fctiwz` | Floating-point math |
| VMX | `vaddfp`, `vmulfp`, `lvx`, `stvx` | 128-bit SIMD |

### Endianness

**Endianness** describes the byte order used to store multi-byte integers in memory.

- **Big-endian (BE):** Most significant byte first. `0x12345678` stored as `12 34 56 78`.
- **Little-endian (LE):** Least significant byte first. `0x12345678` stored as `78 56 34 12`.

The Xbox 360 (PowerPC) is **big-endian**. Modern x86 PCs are **little-endian**. This is one of the most important differences for recompilation.

When you load a 32-bit integer from Xbox 360 memory on a PC, the bytes are reversed. The `REX_LOAD_U32` macro handles this automatically with `__builtin_bswap32`:

```cpp
// Load a 32-bit big-endian value from guest memory:
u32 value = REX_LOAD_U32(guest_address);
// Equivalent to: __builtin_bswap32(*(u32*)(base + guest_address))
```

All multi-byte data in guest memory is big-endian. Single bytes (`u8`) need no swap.

---

## Register File

The PowerPC register file is the set of registers available to the CPU. Every recompiled function receives a `PPCContext` struct containing all of these.

### General-Purpose Registers (GPRs) — r0 to r31

32 integer registers, each 64 bits wide.

| Registers | Role |
|-----------|------|
| r0 | Volatile; sometimes used as a temporary in function prologues |
| r1 | Stack pointer — always points to the current stack frame |
| r2 | Table of Contents (TOC) pointer — rarely used in games |
| r3–r10 | Function arguments (integer). r3 also holds the return value. |
| r11–r12 | Volatile temporaries |
| r13 | Thread pointer (TLS base) — the "thread-local storage" base |
| r14–r31 | Non-volatile (callee-saved) — must be preserved across function calls |

In ReXGlue's `PPCContext`, these are accessed as `ctx.r3.u32`, `ctx.r3.s32`, `ctx.r3.u64`, etc. The `Register` union allows accessing the same register as signed/unsigned 32-bit or 64-bit values.

### Floating-Point Registers (FPRs) — f0 to f31

32 floating-point registers, each 64 bits wide (IEEE 754 double precision).

| Registers | Role |
|-----------|------|
| f0 | Volatile |
| f1–f13 | Function arguments (float/double). f1 also holds the return value. |
| f14–f31 | Non-volatile (callee-saved) |

### Vector Registers (VMX/Altivec) — v0 to v127

128 vector registers, each 128 bits wide (the standard 32 Altivec registers, plus 96 VMX128 extensions).

Each vector register can hold: 4 × `float`, 4 × `u32`, 8 × `u16`, or 16 × `u8`.

### Special Registers

| Register | Purpose |
|----------|---------|
| LR (Link Register) | Holds the return address after a function call (`bl` saves PC+4 here) |
| CTR (Count Register) | Loop counter; also used for indirect calls (`bctr` jumps to CTR) |
| CR (Condition Register) | 8 fields (CR0–CR7), each 4 bits: LT, GT, EQ, SO (summary overflow) |
| XER | Fixed-point Exception Register: SO (summary overflow), OV (overflow), CA (carry) |
| FPSCR | Floating-Point Status and Control Register |
| MSR | Machine State Register — controls processor mode |

---

## Calling Convention

The Xbox 360 uses the **PowerPC 64-bit ELF Procedure Call Standard** (ABI), adapted for the Xbox 360 kernel environment.

### Integer Arguments

Arguments are passed in registers r3 through r10, in order:

| Argument Position | Register |
|-------------------|---------|
| 1st | r3 |
| 2nd | r4 |
| 3rd | r5 |
| 4th | r6 |
| 5th | r7 |
| 6th | r8 |
| 7th | r9 |
| 8th | r10 |
| 9th and beyond | Stack, at r1 + 0x54 + ((n-8) × 8) |

Return values go in **r3** (integer) or **f1** (float/double).

### Float/Double Arguments

Float and double arguments go in f1 through f13, regardless of their position in the argument list. The integer and float argument registers are tracked separately:

```
// C++ function: void foo(int a, float b, int c, double d)
// a → r3, b → f1, c → r4, d → f2
```

`ArgTranslator` in `rexglue-sdk/include/rex/ppc/function.h` implements this mapping for auto-marshaled hooks.

### Stack Frame Layout

Each function creates a stack frame:
- Minimum frame size: 0x70 bytes
- r1 always points to the frame base
- Saved registers are stored below the caller's frame
- The back-chain word at r1+0 points to the caller's r1

---

## Memory System

### Address Space

The Xbox 360's virtual memory is a 32-bit address space (4 GB total). Key regions:

| Region | Address Range | Contents |
|--------|-------------|----------|
| NULL guard | `0x00000000` | Unmapped — null pointer dereference trap |
| XEX image | `0x82000000` | Game code and data (typical load address) |
| Stack | Near top of RAM | Thread stacks, grows downward |
| MMIO | `0x7F000000–0x7FFFFFFF` | GPU registers, hardware I/O |
| Physical | `0xE0000000+` | Direct physical memory (Xbox kernel use) |

In ReXGlue, the entire 4 GB space is simulated as a 4 GB host allocation:
```cpp
uint8_t* base = runtime->virtual_membase(); // 4 GB allocation
uint8_t* ptr = base + guest_addr;           // Any guest address → host pointer
```

### Shared Memory

The Xbox 360 has a single pool of 512 MB GDDR3 RAM shared between the CPU and GPU. There is no separate GPU VRAM. This is unusual compared to modern PCs where the GPU has its own dedicated memory.

For recompilation, this means GPU resources (textures, vertex buffers) live in the same address space as game data.

---

## GPU — ATI Xenos

The Xenos GPU is a custom chip designed by ATI (now AMD) for the Xbox 360. It was groundbreaking at the time for its **unified shader architecture** — unlike contemporary PC GPUs that had separate vertex and pixel shader hardware, the Xenos has a single pool of shader processors that can be assigned to any task.

### Key Properties

- 48 shader processors (arranged as 3 × 16)
- Shader Model 3.0+ equivalent (with extensions)
- 10 MB internal eDRAM (for render targets — zero-cost MSAA and depth)
- Supports DirectX 9.0c-era feature set plus Xbox 360 extensions
- Programmable via a register-based command buffer protocol called **PM4**

### GPU Command Buffer

The CPU communicates with the Xenos GPU by writing packets to a ring buffer. These packets are in the **PM4** format, originally designed by ATI. Packet types include:

- `PM4_DRAW_INDX` — draw primitives
- `PM4_LOAD_REGISTER_IMM` — write GPU registers
- `PM4_EVENT_WRITE` — GPU event notification
- `PM4_XE_SWAP` — present the frame (Xbox 360 specific)

ReXGlue's `rex::graphics::CommandProcessor` reads and processes these packets, translating them to D3D12 or Vulkan commands.

### Xenos Registers

The Xenos has hundreds of registers controlling every aspect of rendering. In ReXGlue, `rex::graphics::RegisterFile` stores these register values, and `rex::graphics::registers.h` defines the register table.

GPU registers are accessed via the MMIO region (`0x7F000000+`). When the game writes to a GPU register address, the `MMIOHandler` intercepts it and calls the command processor.

### Shaders

The Xenos uses a custom shader bytecode format (not HLSL or GLSL). ReXGlue includes a shader translation layer that converts Xbox 360 shaders to DXIL (for D3D12) or SPIR-V (for Vulkan) at runtime.

---

## Executable Format — XEX2

XEX2 (Xbox Executable version 2) is the executable format for Xbox 360 software. It is conceptually similar to PE (Windows `.exe`) but with Xbox-specific additions.

### File Structure

```
XEX2 Header
  ├── Magic: "XEX2" (0x58455832)
  ├── Module flags
  ├── PE data offset (points to embedded PE)
  ├── Security info
  └── Optional header list
        ├── Base address
        ├── Entry point
        ├── Title ID
        ├── Image flags
        ├── Import table (list of kernel modules and functions)
        └── Export table (for DLL modules)
PE/COFF Data
  └── Compressed, encrypted sections
        ├── .text  — code
        ├── .rdata — read-only data
        ├── .data  — initialized data
        └── .bss   — zero-initialized data
```

### Import Table

The import table lists every kernel function the game uses. Each import entry contains:
- **Module name:** e.g., `xboxkrnl.exe`, `xam.xex`
- **Ordinal:** a number identifying the function within the module
- **Thunk address:** a location in guest memory that is patched at load time with the function's address

When ReXGlue loads the XEX, it resolves imports by calling `ExportResolver`, which patches each thunk with the address of the corresponding `REX_EXPORT` implementation.

### Common Import Modules

| Module | Description |
|--------|-------------|
| `xboxkrnl.exe` | Core Xbox 360 kernel: memory, threads, synchronization, file I/O |
| `xam.xex` | Xbox Achievement Manager: achievements, profiles, UI |
| `xbdm.xex` | Xbox Debug Manager: debug communication (development builds only) |

---

## Kernel Modules

### xboxkrnl.exe

The Xbox 360 kernel. Provides:
- **Memory:** `XMemAlloc`, `XMemFree`, `MmAllocatePhysicalMemory`, `VirtualAlloc`
- **Threads:** `ExCreateThread`, `KeDelayExecutionThread`, `KeSetBasePriorityThread`
- **Synchronization:** `NtCreateEvent`, `NtSetEvent`, `NtWaitForSingleObject`
- **File I/O:** `NtCreateFile`, `NtReadFile`, `NtWriteFile`, `NtClose`
- **Time:** `KeQueryPerformanceFrequency`, `KeQuerySystemTime`
- **Input:** `XInputGetState`, `XInputSetState`

### xam.xex

The Xbox Achievement Manager and title services module. Provides:
- Achievement unlocking and progress
- User profiles and gamertag information
- Title storage (persistent save data outside STFS)
- Network presence and friends list
- Marketplace and DLC enumeration

---

## File Formats

### STFS (Secure Transacted File System)

The Xbox 360 uses STFS packages for save games, DLC, and title updates. An STFS package is a single file that contains a complete file tree.

Types:
- `CON` — Consumer content (game saves, created by the console)
- `LIVE` — Marketplace content (DLC, purchased)
- `PIRS` — Microsoft-signed content (system updates)

### Xbox Media File Formats

| Format | Description |
|--------|-------------|
| `.xex` | Xbox Executable — game binary |
| `.xex2` | Xbox Executable v2 (same as .xex, alternate extension) |
| `.xcp` | Xbox Content Package (older DLC format) |
| `.nxe` | Xbox Experience Update |
| `.pam` / `.bik` | Video formats used in cutscenes |

### WWE '13 Specific Formats

WWE '13 (built by Yuke's on their proprietary engine) uses several custom asset formats. These are not yet fully documented. Binary analysis with Ghidra will be required to understand the game's asset loading and file format code.

---

## Boot Process

Understanding the Xbox 360 boot process helps you understand what happens between power-on and the game running.

1. **BootROM** — small ROM on the CPU executes first, verifies the hypervisor
2. **Hypervisor** — runs in highest privilege mode, enforces code signing
3. **Xbox 360 OS (kernel)** — loads, initializes hardware, starts the dashboard
4. **Dashboard / NXE** — the user interface. Launches games.
5. **Game XEX load** — the kernel decrypts, decompresses, and maps the XEX
6. **Import resolution** — kernel patches the import thunks with function addresses
7. **Entry point** — the game's `_start` or `WinMainCRTStartup` equivalent runs
8. **CRT initialization** — global constructors run, heap is initialized
9. **WinMain equivalent** — the game's initialization code runs

In a recompilation, steps 1–4 are replaced entirely by `rex::Runtime::Setup()`. Steps 5–6 are handled by `Runtime::LoadXexImage()`. Steps 7–9 happen when `Runtime::LaunchModule()` calls the entry point.

---

## Debugging on Xbox 360

In original Xbox 360 development:
- **XBDM** (Xbox Debug Manager) provided a communication protocol between the console and a PC dev kit
- **PIX** (Performance Investigator for Xbox) captured GPU traces
- Microsoft's **Xbox 360 SDK** provided the headers and libraries

For recompilation:
- The XBDM module is not needed. Attach Visual Studio's debugger to the host process.
- GPU traces can be captured with ReXGlue's `rex::graphics::trace_writer`.
- Tracy profiler provides CPU-side performance analysis.
- The ReXGlue debug overlay surfaces log output in the application window.

---

## Why Recompilation is Difficult

Static recompilation sounds conceptually simple: read instructions, write C++. In practice, several challenges make it significantly harder.

### Indirect Calls

When the game calls a function through a pointer (via the CTR register: `bctrl`), the codegen tool cannot know at analysis time which function will be called. The generated code must look up the target address at runtime in `PPCFuncMappings[]`. If the game calls a function whose address was not discovered during analysis, it will crash.

### Self-Modifying Code

Some games patch their own code at runtime. Static recompilation assumes code is fixed — if the game writes to a code address, the translated C++ will not reflect that change. WWE '13 is unlikely to use this pattern, but it is a known challenge for other titles.

### Data as Code

If the game stores function pointers in data sections (vtables, dispatch tables, callbacks), the codegen tool must identify these to add them to `PPCFuncMappings[]`. The `VtableScanner` and `SigScanner` in ReXGlue attempt to handle this automatically.

### Big-Endian Memory

Every multi-byte value in guest memory is stored big-endian. Forgetting to byte-swap a structure field causes incorrect reads. The `rex::be<T>` template and `REX_LOAD_*` macros mitigate this, but complex structures require careful attention.

### Kernel Implementation Completeness

`xboxkrnl.exe` exports hundreds of functions. Many are complex operating system primitives (virtual memory management, fiber scheduling, low-level I/O). Each one must be correctly implemented as a `REX_EXPORT` or the game will misbehave.

### Floating-Point Precision

PowerPC uses IEEE 754 strictly. x86 historically uses 80-bit extended precision internally. Differences in floating-point rounding and denormal handling can cause physics simulations or AI calculations to diverge subtly. ReXGlue uses `-ffp-model=strict` to mitigate this.

### Thread Safety and Memory Ordering

The Xenon CPU uses a relaxed memory model for multi-threaded code. Synchronization is done via `lwsync` and `sync` instructions. Translating these to correct x86 memory barriers (`mfence`, `sfence`, etc.) requires careful analysis of each barrier's intent.

### Compressed Code Size

Recompiled games produce very large C++ translation units. A single game function can expand from 100 PPC instructions to 500+ lines of C++. A full game binary of 100,000 functions results in tens of millions of lines of C++. ReXGlue uses `-mcmodel=large` and splits output into many translation units to handle this.
