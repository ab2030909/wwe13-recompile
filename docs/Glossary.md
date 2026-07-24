# Glossary

> A beginner-friendly reference for every important term in the WWE '13 Recompilation project. Terms are grouped by category and cross-referenced where relevant.

If you encounter an unfamiliar term in any document, look it up here first.

---

## Table of Contents

- [General Programming Concepts](#general-programming-concepts)
- [Compilation and Toolchain](#compilation-and-toolchain)
- [Binary Analysis](#binary-analysis)
- [CPU and Architecture](#cpu-and-architecture)
- [PowerPC Specific](#powerpc-specific)
- [Memory and Addressing](#memory-and-addressing)
- [Executable Formats](#executable-formats)
- [Reverse Engineering](#reverse-engineering)
- [Recompilation Specific](#recompilation-specific)
- [ReXGlue Specific](#rexglue-specific)
- [Xbox 360 Specific](#xbox-360-specific)

---

## General Programming Concepts

### ABI (Application Binary Interface)

The low-level interface between compiled code and the operating system, or between two compiled modules. The ABI specifies:
- How function arguments are passed (which registers, which order)
- How return values are communicated
- How the stack is organized
- How data structures are laid out in memory

If two pieces of code agree on an ABI, they can call each other even if compiled by different compilers. If they disagree, function calls produce wrong results or crashes.

The Xbox 360 uses the PowerPC 64-bit ELF ABI adapted for its kernel. See [docs/Xbox360.md](Xbox360.md#calling-convention) for specifics.

---

### API (Application Programming Interface)

A defined set of functions, types, and protocols that software uses to communicate with another system or library.

The Xbox 360 kernel API includes functions like `ExCreateThread` (create a thread), `NtCreateFile` (open a file), and `XInputGetState` (read controller input). These are the functions WWE '13 calls — and the ones we must implement in our hooks.

---

### SDK (Software Development Kit)

A collection of tools, headers, and libraries that developers use to build software for a specific platform or framework.

The **ReXGlue SDK** provides:
- Public C++ headers (`include/rex/`)
- Pre-compiled libraries (`rexruntime.dll`)
- CMake integration helpers
- The `rexcodegen` tool

---

### Runtime

The code that is present while a program is executing. In ReXGlue's context, the **runtime** is the set of libraries and services that support the recompiled game at execution time: memory management, kernel emulation, graphics backend, audio, and input.

Contrast with **compile time** (when the compiler translates source code) and **codegen time** (when the ReXGlue tool translates PPC instructions to C++).

---

### Assembly Language

A human-readable representation of machine code. Each assembly instruction corresponds to exactly one machine code instruction. Assembly is CPU-specific — PowerPC assembly is completely different from x86-64 assembly.

Example PowerPC assembly:
```asm
addi    r3, r4, 0x10   ; Add immediate: r3 = r4 + 16
stw     r3, 0(r1)      ; Store word: memory[r1 + 0] = r3
blr                     ; Branch to link register (return)
```

---

### Machine Code

The binary encoding of processor instructions. Machine code is what is actually stored in an executable file and executed by the CPU. Assembly language is the human-readable form of machine code.

When we say "the Xbox 360 runs PowerPC machine code", we mean the XEX binary contains PowerPC instructions encoded as 32-bit binary values.

---

## Compilation and Toolchain

### Compiler

A program that translates source code (e.g., C++) into machine code (binary executable). For this project:
- **Clang 18+** is the compiler used to compile the ReXGlue SDK and generated code
- The Xbox 360's original game code was compiled by a proprietary Xbox 360 compiler (likely based on the IBM XL C++ compiler or a Microsoft variant)

---

### Linker

A program that combines multiple compiled object files and libraries into a single executable. After the compiler produces `.obj` files (one per `.cpp`), the linker combines them, resolves cross-references between them, and produces the final `.exe` or `.dll`.

A common source of confusion: compiler errors and linker errors look similar but are different. A compiler error means the code is syntactically or semantically wrong. A linker error (`LNK2001: unresolved external symbol`) means a function or variable is referenced but not defined anywhere.

---

### Build System

The tool that orchestrates compilation and linking. For this project:
- **CMake** is the build system *generator* — it reads `CMakeLists.txt` and writes Ninja build files
- **Ninja** is the build *executor* — it reads `build.ninja` and runs compiler and linker commands
- This separation allows the same CMakeLists to work with different build executors

---

### C++ Standard

C++ evolves over time, with new versions adding features. The versions are named by year:
- C++11: Modern C++ (lambdas, `auto`, smart pointers)
- C++14, C++17: Incremental improvements
- C++20: Concepts, ranges, coroutines, `std::format`
- **C++23**: Required by ReXGlue. Adds `std::byteswap`, improved `std::format`, `std::expected`, and more.

ReXGlue uses C++23 specifically for `std::byteswap` (used in the `rex::byte_swap<T>` helper), `std::endian` detection, and C++23 concepts in the hook marshaling system.

---

### Object File

The intermediate output of compilation, before linking. Each `.cpp` file compiles to one `.obj` file. Object files contain machine code and symbol tables (function names, addresses), but with unresolved cross-references that the linker fills in.

---

### Static Library vs. Dynamic Library

A **static library** (`.lib`, `.a`) is copied into the executable at link time. The executable is self-contained.

A **dynamic library** / shared library (`.dll`, `.so`) is loaded at runtime. The executable has a stub that resolves to the actual DLL at load time. Multiple programs can share one copy of the DLL.

The ReXGlue runtime is a **dynamic library** (`rexruntime.dll`). Your game executable links against it at runtime.

---

## Binary Analysis

### Disassembler

A tool that translates binary machine code back into human-readable assembly language. It does not attempt to understand the code — it only converts the binary encoding to text.

`Ghidra` can disassemble PowerPC binaries. When you open a XEX in Ghidra, it disassembles all code sections automatically.

---

### Decompiler

A more advanced tool that attempts to translate machine code back into a high-level language (usually C). Unlike a disassembler, it reconstructs `if` statements, loops, function calls, and variable types.

Ghidra includes a decompiler. Its output is approximate — variable names are made up, types may be wrong — but it provides a much higher-level view than raw assembly.

---

### Static Analysis

Analysis of a binary without executing it. Ghidra performs static analysis: it disassembles, builds call graphs, identifies functions, and infers types, all without running the program.

Advantages: safe, works on any binary, can analyze the entire binary.
Disadvantages: cannot observe runtime-only behavior (dynamic function dispatch, self-modifying code).

---

### Dynamic Analysis

Analysis of a binary by running it and observing its behavior. Tools: debuggers (set breakpoints, inspect memory), memory profilers, system call tracers.

For Xbox 360 binaries, dynamic analysis was originally done on hardware or via Xenia (emulator). In the recompilation context, attaching Visual Studio's debugger to the recompiled executable provides dynamic analysis on the translated code.

---

### Symbol

A name associated with an address in a binary. In an unstripped binary, every function and global variable has a symbol name (e.g., `CWWEEntityManager::UpdateTransforms`). In a stripped binary (typical for retail games), symbols are removed and functions have only addresses (e.g., `sub_82012ABC` — where `sub_` indicates "subroutine at address").

Ghidra can recover some symbols via:
- String references (error messages often contain function names)
- RTTI (run-time type information) for C++ class names
- Pattern matching against known library functions
- Import/export table entries (kernel functions have names)

---

### Relocation

When a binary is compiled, absolute addresses (e.g., "the function is at `0x82012ABC`") are written throughout the code and data. If the binary is loaded at a different address, all those hardcoded addresses are wrong. A **relocation** is a record that says "at this offset in the binary, there is an address that needs to be adjusted by the load address delta."

The XEX format uses relocations to allow loading at different base addresses. The ReXGlue runtime's `ExportResolver` handles these when loading the XEX image.

---

## CPU and Architecture

### ISA (Instruction Set Architecture)

The specification of what instructions a CPU can execute: their encodings, their behavior, and the registers they use. Examples: x86-64 (Intel/AMD), ARM64 (Apple/Qualcomm), PowerPC (IBM).

Two CPUs with the same ISA are binary-compatible: code compiled for one runs on the other. Two CPUs with different ISAs are not.

The Xbox 360 CPU uses the **PowerPC 2.02** ISA.

---

### RISC vs. CISC

Two philosophies of CPU design:

**RISC (Reduced Instruction Set Computer):** Small, uniform instruction set. Each instruction does one simple thing. Instructions are fixed-size. Examples: PowerPC, ARM.

**CISC (Complex Instruction Set Computer):** Large instruction set with complex, variable-length instructions. Individual instructions can do multiple operations. Examples: x86, x86-64.

PowerPC is RISC. The Xbox 360 Xenon and the PlayStation 3 Cell are both RISC designs. x86-64 (your PC CPU) is CISC. This architectural difference is one reason cross-platform ports require significant work.

---

### Endianness

The byte order used to store multi-byte integers in memory.

**Big-endian:** The most significant byte is stored first. `0x12345678` is stored as `12 34 56 78` in memory. Used by PowerPC (Xbox 360).

**Little-endian:** The least significant byte is stored first. `0x12345678` is stored as `78 56 34 12` in memory. Used by x86-64 (your PC).

This is one of the core challenges of recompilation: every multi-byte value read from Xbox 360 memory must be byte-swapped before use on a little-endian host. The `REX_LOAD_U32` macro handles this automatically.

---

### SIMD (Single Instruction, Multiple Data)

A class of CPU instructions that perform the same operation on multiple data elements simultaneously. SIMD instructions are used for physics, audio processing, image manipulation, and other data-parallel tasks.

On x86-64: SSE, SSE2, AVX, AVX2 are SIMD instruction sets.
On PowerPC: **Altivec** / VMX is the SIMD instruction set.

WWE '13 uses Altivec heavily for physics and animation. ReXGlue uses **simde** (a portability library) to translate Altivec intrinsics to equivalent x86 SSE/AVX intrinsics.

---

### Register

A small, fast storage location inside the CPU itself. Registers hold the current values being operated on. They are orders of magnitude faster than RAM.

PowerPC has: 32 general-purpose registers (r0–r31), 32 floating-point registers (f0–f31), 128 vector registers (v0–v127), and several special registers (CR, XER, LR, CTR, FPSCR).

In ReXGlue, all of these are stored in the `PPCContext` struct which is passed to every recompiled function.

---

### Stack

A region of memory used to store temporary data during function calls: local variables, saved register values, and the return address. The stack grows downward (from high addresses to low addresses) in most architectures including PowerPC.

The stack pointer register (r1 in PowerPC) always points to the current frame's base. When a function calls another function, it decrements r1 to allocate space for its own frame.

---

### Calling Convention

The rules that specify how function arguments are passed, how return values are communicated, which registers must be preserved across calls (non-volatile), and how the stack frame is organized.

See [docs/Xbox360.md](Xbox360.md#calling-convention) for the full Xbox 360 PowerPC calling convention.

---

## PowerPC Specific

### PPC / PowerPC

**PPC** is the common abbreviation for **PowerPC**, the processor architecture used by the Xbox 360. "PPC" appears throughout the ReXGlue codebase: `PPCContext`, `PPCFunc`, `PPCFuncMapping`, `REX_CALL_FUNC`, etc.

---

### GPR (General-Purpose Register)

One of the 32 integer registers (r0–r31) in the PowerPC register file. Used for integer arithmetic, address computation, and passing function arguments.

---

### FPR (Floating-Point Register)

One of the 32 floating-point registers (f0–f31). Used for floating-point arithmetic. Each FPR is 64 bits wide and holds IEEE 754 double-precision values.

---

### CR (Condition Register)

A special register divided into 8 4-bit fields (CR0–CR7). Each field has LT (less than), GT (greater than), EQ (equal), and SO (summary overflow) bits. Set by comparison instructions; read by conditional branch instructions.

In `PPCContext`, this is represented as `cr0` through `cr7`, each a `PPCCRRegister` struct with `lt`, `gt`, `eq`, `so` members.

---

### XER (Fixed-Point Exception Register)

A special register tracking arithmetic overflow and carry. Has three key bits: SO (summary overflow), OV (overflow), CA (carry). Used by instructions like `addo` (add with overflow).

---

### LR (Link Register)

A special register that holds the return address for function calls. The `bl` (branch and link) instruction stores the next instruction address into LR before jumping. The `blr` (branch to link register) instruction jumps back to LR to return from a function.

In `PPCContext`, this is `ctx.lr`.

---

### CTR (Count Register)

A special register primarily used as a loop counter. The `bdnz` instruction decrements CTR and branches if it is not zero. Also used for indirect function calls: `bctr` jumps to the address stored in CTR.

When the game calls a function through a pointer, it loads the function address into CTR and executes `bctr`. ReXGlue's `REX_CALL_INDIRECT_FUNC` macro handles this by looking up the address in `PPCFuncMappings[]`.

---

### Altivec / VMX

The SIMD extension on PowerPC. Altivec adds 32 128-bit vector registers (v0–v31). VMX128 (Xbox 360 extension) extends this to 128 vector registers (v0–v127).

Each vector register can hold: 4 floats, 4 int32s, 8 int16s, or 16 int8s.

ReXGlue uses **simde** to translate Altivec/VMX operations to x86 SSE operations.

---

### FPSCR (Floating-Point Status and Control Register)

Controls floating-point rounding mode and exception handling. Has fields for rounding mode (round to nearest, toward zero, up, down) and exception status.

ReXGlue carefully manages FPSCR to maintain IEEE 754 compliance. The `FPSCRRegister` struct in `PPCContext` and its `enableFlushMode` / `disableFlushMode` methods control the host CPU's corresponding floating-point control registers.

---

## Memory and Addressing

### Virtual Address Space

The address space seen by a program. On the Xbox 360, the virtual address space is 32-bit (0x00000000 to 0xFFFFFFFF). Programs use virtual addresses; the hardware MMU translates them to physical RAM locations.

In ReXGlue, the entire 32-bit virtual address space is simulated as a 4GB allocation on the host. The `base` pointer in recompiled functions is the start of this allocation.

---

### Physical Address

The actual address of a memory location in RAM hardware. Programs normally use virtual addresses; physical addresses are only relevant at the kernel/hardware level.

---

### MMIO (Memory-Mapped I/O)

A technique where hardware registers are accessed as if they were memory locations. Instead of using special I/O instructions, you read and write specific memory addresses to communicate with hardware.

The Xbox 360's Xenos GPU is controlled via MMIO. Writing to addresses like `0x7F000000 + offset` sends commands and data to the GPU. ReXGlue's `MMIOHandler` intercepts these writes and routes them to the graphics system.

---

### Guest Address

An address in the Xbox 360's virtual memory space. Always 32-bit (fits in `u32`). Stored in `PPCContext` registers and in guest memory structures as big-endian values.

To convert a guest address to a host pointer: `uint8_t* host = base + guest_addr;`

---

### Host Address

A native pointer on the recompilation host machine. 64-bit on x64 systems. The `base` pointer is a host address. After `uint8_t* host = base + guest_addr`, `host` is a host address.

---

### MappedPtr

A ReXGlue type (`rex::MappedPtr<T>`) that stores both a host pointer and the corresponding guest address. Used in hook arguments to provide access to both:

```cpp
// Hook argument: the game passes a guest address for an XINPUT_STATE structure
static void Hook_XInputGetState(u32 controller_index, mapped_u32 out_state) {
    // out_state.host_address() → uint32_t* pointing into the 4GB guest space
    // out_state.guest_address() → the original uint32_t Xbox 360 address
}
```

---

## Executable Formats

### ELF (Executable and Linkable Format)

The standard executable format on Linux and many Unix systems. An ELF file contains sections (`.text` for code, `.data` for data, `.rodata` for read-only data), a symbol table, and relocation information.

PowerPC ELF is used on Linux PowerPC systems. The Xbox 360 does not use ELF; it uses XEX.

---

### PE (Portable Executable)

The executable format used by Windows. `.exe` and `.dll` files are PE format. A PE file contains sections, an import table (list of DLLs and functions it uses), an export table (functions it provides), and a header with entry point and image base.

Xbox 360 XEX files embed a PE inside them. After decryption and decompression, the inner PE is mapped into the guest address space.

---

### XEX (Xbox EXecutable)

The executable format for Xbox 360 games and applications. A XEX wraps an encrypted, compressed PE with Xbox-specific metadata:
- Image base address and image size
- Import table (kernel function references with ordinals)
- Export table (for satellite module XEX files)
- Title ID (unique identifier for the game)
- Media type flags (disc, marketplace, etc.)

See [docs/Xbox360.md](Xbox360.md#executable-format--xex2) for the full format breakdown.

---

## Reverse Engineering

### Reverse Engineering

The process of analyzing a compiled binary to understand its design, behavior, and implementation without access to the original source code. Key techniques:
- **Disassembly** — reading the machine instructions
- **Decompilation** — converting machine code back to C pseudocode
- **Dynamic analysis** — running the code and observing behavior
- **Pattern matching** — recognizing known library code signatures
- **Cross-referencing** — following how data and functions reference each other

For this project, we use Ghidra to reverse engineer the WWE '13 XEX binary.

---

### Ghidra

A free, open-source reverse engineering tool developed by the NSA (National Security Agency, United States). Ghidra provides:
- Disassembly and decompilation
- Cross-reference analysis (who calls this function? who reads this variable?)
- Multiple processor support including PowerPC
- Scripting API (Python, Java) for automation
- Collaboration features for team analysis

---

### Binary

In the context of software engineering, "a binary" refers to a compiled executable file as opposed to source code. "Analyzing the binary" means examining the compiled game file rather than source code (which we don't have).

---

### IDA Pro / IDA

Interactive DisAssembler Pro — a commercial reverse engineering tool widely used in professional security research. More powerful than Ghidra in some areas (especially its decompiler, Hex-Rays), but expensive. Free versions (IDA Freeware) exist with limitations.

The ReXGlue SDK includes a script (`scripts/ida/export_named_funcs.py`) for exporting named function lists from IDA to use with the codegen tool. This script can be adapted for Ghidra.

---

### Cross-Reference (XRef)

A record of which code or data refers to a specific address. When Ghidra shows "xrefs to `sub_82012ABC`", it lists every place in the binary that calls or references that function. Cross-references are essential for understanding how a function is used.

---

## Recompilation Specific

### Static Recompilation

The process of translating a binary from one architecture to another architecture's source code, ahead of time. "Static" means it is done before execution, as opposed to dynamic (JIT) recompilation which happens at runtime.

ReXGlue performs static recompilation: the Xbox 360 PowerPC binary is translated to C++ source, which is then compiled to native x64 code. The translation happens once; the result runs natively forever.

---

### Dynamic Recompilation (JIT)

Translation from one architecture to another at runtime. Emulators like RPCS3 (PS3) and early versions of Xenia (Xbox 360) use dynamic recompilation: they translate PPC instructions to x86 machine code on demand as the game runs.

Advantages: handles self-modifying code, no upfront translation cost.
Disadvantages: translation overhead every run, complex implementation, harder to optimize.

---

### Stub / Thunk

A placeholder function implementation. When a function is called but not yet implemented, a stub is inserted instead. Stubs typically log a warning and return a default value.

In ReXGlue, `REX_STUB(XInputGetState)` inserts a stub that logs "XInputGetState STUB" when called. Stubs are replaced with real implementations as development progresses.

The term "thunk" specifically refers to a stub in an import table — a small piece of code that forwards a call to the actual implementation. In XEX import resolution, each imported function has a "thunk" address in the game's address space that is patched at load time with the hook function's address.

---

### Hook

A replacement for a function in the original binary. When a hook is installed for function `sub_82012ABC`, any call to `sub_82012ABC` in the generated code calls your hook instead.

In ReXGlue, `REX_HOOK(sub_82012ABC, MyNativeImplementation)` installs a hook. The `HostToGuestFunction` template translates the PowerPC register calling convention to/from native C++ function arguments automatically.

---

### Code Generation (Codegen)

The process of producing code. In ReXGlue's context, "codegen" refers specifically to the tool and process that translates PPC binary instructions into C++ source code. The output is "generated code."

---

### Function Boundary Recovery

The process of identifying where functions begin and end in a binary. Without source code, function boundaries must be inferred from code patterns: entry points, return instructions, cross-references, and prologues/epilogues.

The ReXGlue `FunctionScanner` performs function boundary recovery as part of the codegen analyze phase.

---

### Control Flow Graph (CFG)

A directed graph where each node is a basic block (a sequence of instructions with no branches in or out except at the beginning and end), and edges represent possible execution paths (branches, function calls).

CFG analysis is used by the codegen tool to understand the structure of each function and translate it correctly.

---

## ReXGlue Specific

### PPCContext

The struct that holds the entire state of the Xbox 360 CPU for one execution thread: all 32 GPRs, 32 FPRs, 128 VMX registers, CR, XER, LR, CTR, FPSCR, and MSR. Every recompiled function receives a reference to a `PPCContext` and must not modify registers it doesn't own (respecting the calling convention).

---

### PPCFunc

The C++ function signature for every recompiled function:
```cpp
using PPCFunc = void(PPCContext& ctx, uint8_t* base);
```
Every generated function, every hook, and every export has this signature. The `ctx` parameter is the CPU register state; `base` is the guest memory base pointer.

---

### PPCFuncMapping

A pair of `{guest_address, host_function_pointer}`. The `PPCFuncMappings[]` array is the complete table of all discovered functions. At runtime, when the game makes an indirect call, `ResolveIndirectFunction(address)` looks up this table to find the host C++ function to call.

---

### REX_HOOK

A macro that declares a `PPCFunc`-signature wrapper that calls a native C++ function, automatically translating register arguments using `HostToGuestFunction`. This is the primary way to replace game or kernel functions.

---

### REX_STUB

A macro that declares a no-op `PPCFunc` that logs a warning when called. Used for unimplemented functions. Always prefer `REX_STUB` over an empty function body, because the log output is essential for tracking what needs to be implemented.

---

### REX_EXPORT

A `REX_HOOK` that also registers the function in the global PPC function registry. Used for kernel exports (xboxkrnl, xam) so they can be found by the `ExportResolver` at import resolution time.

---

### REX_IMPORT

A macro that creates a typed callable wrapper (`rex::ppc::ImportFunction<Sig>`) around a recompiled game function. Allows hooks to call back into the generated game code in a type-safe way.

---

### rexcrt

ReXGlue's replacement C runtime. Provides standard C library functions (heap allocation, string operations, math) that the game's own CRT calls into. When `rexcrt_heap = true` in `PPCImageInfo`, the rexcrt heap is initialized.

---

## Xbox 360 Specific

### Xenon

The codename for the Xbox 360 console, and also the name of its IBM CPU chip. The Xenon CPU has 3 PowerPC cores, each dual-threaded.

---

### Xenos

The codename for the Xbox 360's custom ATI GPU. It pioneered the unified shader model. Xenos is controlled by writing PM4 packets to a command buffer ring.

---

### XAM (Xbox Achievement Manager)

The Xbox 360 kernel module (`xam.xex`) providing user-facing services: achievements, profiles, gamertag information, title storage, marketplace, and friends list.

---

### XBOXKRNL

The Xbox 360 core kernel module (`xboxkrnl.exe`). Provides fundamental OS services: memory allocation, thread management, synchronization primitives, file I/O, and input.

---

### XMA (Xbox Media Audio)

The proprietary audio codec used by Xbox 360 games. XMA is similar to WMA (Windows Media Audio). XMA2 is the improved second version with better seeking. WWE '13 uses XMA2 for music and voice.

---

### STFS (Secure Transacted File System)

The Xbox 360's container format for downloadable content, game saves, and title updates. An STFS file is a single file that contains a complete virtual filesystem. Types: CON (console-created), LIVE (marketplace), PIRS (Microsoft-signed).

---

### PM4

The GPU command packet protocol used by ATI/AMD graphics hardware, including the Xenos. PM4 packets command the GPU to draw primitives, set state, write registers, and signal events. The Xbox 360's command processor reads PM4 packets from the ring buffer.

---

### eDRAM

Embedded DRAM — a small, high-bandwidth memory block inside the Xenos GPU die. The Xbox 360's Xenos has 10 MB of eDRAM used exclusively for render targets (color buffers, depth buffers). Operations on the eDRAM (including MSAA resolve) are effectively free due to its bandwidth. The 10 MB limit constrains render target resolution.

---

### VFS (Virtual File System)

ReXGlue's abstraction for file system access. Maps Xbox 360 virtual path prefixes (`game:`, `d:`, `user:`, `update:`) to host filesystem directories. Allows the game to access its data files without knowing they are on a PC.
