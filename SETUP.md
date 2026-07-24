# Setup Guide

This document walks through a complete development environment setup for the WWE '13 Recompilation project. Every dependency is explained before installation instructions are given.

> **Platform:** Windows 10 / 11 (64-bit). Linux instructions are included where the steps differ.

---

## Table of Contents

- [Overview](#overview)
- [1. Visual Studio 2022](#1-visual-studio-2022)
- [2. LLVM / Clang](#2-llvm--clang)
- [3. CMake](#3-cmake)
- [4. Ninja](#4-ninja)
- [5. Python](#5-python)
- [6. Git](#6-git)
- [7. Ghidra](#7-ghidra)
- [8. ReXGlue SDK](#8-rexglue-sdk)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

---

## Overview

The toolchain for this project has two layers:

1. **Build tools** — used to compile the C++ code that ReXGlue generates (Visual Studio, Clang, CMake, Ninja)
2. **Analysis tools** — used to reverse engineer the WWE '13 XEX binary (Ghidra)

ReXGlue *strictly requires* Clang 18+. It will refuse to configure with MSVC or GCC. Visual Studio 2022 is needed for its Windows SDK and debugger, not its compiler.

---

## 1. Visual Studio 2022

### What it is

Visual Studio is Microsoft's integrated development environment. For this project, it provides:
- The **Windows SDK** — header files and libraries for Windows APIs (D3D12, DXGI, etc.)
- The **MSVC linker** — used alongside Clang to link Windows executables
- A **debugger** — essential for diagnosing crashes in the recompiled code

You do not use the MSVC compiler (`cl.exe`) for this project, only its supporting tools.

### Installation

1. Download from [visualstudio.microsoft.com](https://visualstudio.microsoft.com/downloads/) — the **Community** edition is free.
2. Run the installer. In the **Workloads** tab, select:
   - **Desktop development with C++**
3. In the **Individual components** tab, also select:
   - **Windows 11 SDK (10.0.22621.0)** or the latest available
   - **C++ Clang tools for Windows** (optional — provides a secondary Clang installation)
4. Click **Install**.

### Verification

Open a **Developer Command Prompt for VS 2022** and run:

```cmd
cl /?
```

You should see the MSVC compiler version. This confirms the Windows SDK and linker are available.

---

## 2. LLVM / Clang

### What it is

LLVM is a collection of compiler infrastructure tools. Clang is the C/C++ compiler front-end built on LLVM.

ReXGlue **requires Clang 18 or newer**. This is enforced in `CMakeLists.txt` — the build will fail with a fatal error if you use any other compiler. The reason is that ReXGlue uses C++23 features that are only fully implemented in recent Clang versions.

### Installation (Windows)

1. Go to [github.com/llvm/llvm-project/releases](https://github.com/llvm/llvm-project/releases)
2. Find the latest 18.x or newer release.
3. Download the Windows installer: `LLVM-18.x.x-win64.exe`
4. Run the installer. When prompted, choose **"Add LLVM to the system PATH for all users"**.
5. Complete the installation.

### Verification

Open a new command prompt and run:

```cmd
clang --version
clang++ --version
```

Expected output (version numbers will differ):
```
clang version 18.1.8
Target: x86_64-pc-windows-msvc
Thread model: posix
```

If `clang` is not found, you may need to add `C:\Program Files\LLVM\bin` to your system `PATH` manually.

### Adding to PATH manually (if needed)

1. Press `Win + S`, search for **"Environment Variables"**
2. Under **System Variables**, find `Path` and click **Edit**
3. Click **New** and add `C:\Program Files\LLVM\bin`
4. Click **OK** on all dialogs
5. Open a new command prompt and verify again

---

## 3. CMake

### What it is

CMake is a build system generator. It reads `CMakeLists.txt` files and generates build files for a chosen build system (in our case, Ninja). CMake itself does not compile code — it just describes *how* to compile it.

ReXGlue requires **CMake 3.25 or newer** for CMake Presets v6 support.

### Installation (Windows)

1. Go to [cmake.org/download](https://cmake.org/download/)
2. Download the latest Windows installer: `cmake-3.x.x-windows-x86_64.msi`
3. Run the installer. When prompted, select **"Add CMake to the system PATH for all users"**.
4. Complete the installation.

### Verification

```cmd
cmake --version
```

Expected output:
```
cmake version 3.29.x
```

---

## 4. Ninja

### What it is

Ninja is a small, fast build tool. While `make` or MSBuild read large build files and make decisions at build time, Ninja simply executes a list of commands as fast as possible. ReXGlue uses Ninja Multi-Config, which supports building Debug, Release, and RelWithDebInfo configurations without re-running CMake.

### Installation (Windows)

**Option A — via winget (recommended):**
```cmd
winget install Ninja-build.Ninja
```

**Option B — manual:**
1. Go to [github.com/ninja-build/ninja/releases](https://github.com/ninja-build/ninja/releases)
2. Download `ninja-win.zip`
3. Extract `ninja.exe` to a folder on your `PATH` (e.g. `C:\tools\ninja\`)
4. Add that folder to your system `PATH`

### Verification

```cmd
ninja --version
```

Expected output:
```
1.11.1
```

---

## 5. Python

### What it is

Python is used for tooling scripts in this project. The ReXGlue SDK includes Python scripts for tasks like exporting named functions from IDA Pro. Python 3.10 or newer is required.

### Installation (Windows)

1. Go to [python.org/downloads](https://www.python.org/downloads/)
2. Download the latest Python 3.x installer.
3. Run the installer. **Important:** check **"Add Python to PATH"** before clicking Install.
4. Complete the installation.

### Verification

```cmd
python --version
pip --version
```

Expected output:
```
Python 3.12.x
pip 24.x from ...
```

---

## 6. Git

### What it is

Git is a distributed version control system. It tracks every change to every file in the project. This project uses Git submodules (the ReXGlue SDK is a submodule), so you need Git installed.

### Installation (Windows)

1. Go to [git-scm.com/downloads](https://git-scm.com/downloads)
2. Download the Windows installer.
3. During installation:
   - Select **"Git from the command line and also from 3rd-party software"**
   - Select **"Use Windows' default console window"**
   - Leave other settings at their defaults
4. Complete the installation.

### Initial Configuration

After installation, open a terminal and configure your identity:

```cmd
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.autocrlf true
```

### Verification

```cmd
git --version
```

Expected output:
```
git version 2.45.x.windows.1
```

---

## 7. Ghidra

### What it is

Ghidra is a free, open-source reverse engineering tool developed by the NSA. It disassembles and decompiles binary executables, allowing you to analyze WWE '13's machine code without the original source code.

Ghidra supports the PowerPC processor architecture used by the Xbox 360.

Ghidra requires a Java Runtime Environment (JDK 17+).

### Install Java (prerequisite)

1. Go to [adoptium.net](https://adoptium.net/) and download **Eclipse Temurin JDK 21** (LTS).
2. Run the installer and complete the installation.
3. Verify:

```cmd
java -version
```

### Install Ghidra

1. Go to [ghidra-sre.org](https://ghidra-sre.org/) and download the latest release.
2. Extract the ZIP to a permanent location (e.g. `C:\tools\ghidra\`).
3. To launch Ghidra, run `ghidraRun.bat` from the extracted folder.

No PATH configuration is needed for Ghidra — it runs as a standalone application.

### First-Run Setup

When you first open a WWE '13 XEX binary in Ghidra:

1. Create a new project: **File → New Project**
2. Import the XEX: **File → Import File**
3. Ghidra will detect the file format. Select **Xbox 360 XEX** if prompted, or **Binary** with **PowerPC Big Endian 32-bit** processor.
4. Run **Auto Analysis** (this takes several minutes).

---

## 8. ReXGlue SDK

### What it is

The ReXGlue SDK is the core toolkit for this project. It provides:

- The **codegen tool** (`rexglue` CLI) that analyzes XEX binaries and generates C++ code
- The **runtime library** that emulates the Xbox 360 kernel, GPU, audio, and input
- **Public headers** (`include/rex/`) that you use when writing hooks and the app class
- **CMake helpers** (`rexglue_configure_target`, etc.) that wire your project together

### Setting Up the Submodule

If you cloned this repository without `--recurse-submodules`, initialize it now:

```cmd
git submodule update --init --recursive
```

This downloads the ReXGlue SDK source into `rexglue-sdk/`.

### Configuring the SDK

From the repository root, configure the SDK with CMake:

```cmd
cmake --preset win-amd64 -S rexglue-sdk
```

This generates build files in `rexglue-sdk/out/build/win-amd64/`.

### Building the SDK

```cmd
cmake --build rexglue-sdk/out/build/win-amd64 --config Debug
```

The compiled SDK libraries and binaries are placed in `rexglue-sdk/out/win-amd64/Debug/`.

### Available Build Presets

| Preset | Platform | Architecture |
|--------|----------|-------------|
| `win-amd64` | Windows | x64 |
| `win-arm64` | Windows | ARM64 |
| `linux-amd64` | Linux | x64 |
| `linux-arm64` | Linux | ARM64 |

Each preset supports three configurations: `Debug`, `Release`, `RelWithDebInfo`.

---

## Verification

After completing all installations, verify your complete setup with these commands in a new terminal:

```cmd
clang --version
clang++ --version
cmake --version
ninja --version
python --version
git --version
```

All six commands should return version strings without errors.

Then verify the SDK configured and built:

```cmd
dir rexglue-sdk\out\win-amd64\Debug\
```

You should see `rexruntime.dll`, `rexcodegen.exe` (or similar), and accompanying `.lib` files.

---

## Troubleshooting

### `clang` is not recognized

LLVM was not added to PATH. Manually add `C:\Program Files\LLVM\bin` to your system PATH (see [LLVM section](#2-llvm--clang)) and open a new terminal.

### CMake error: "ReXGlue requires Clang compiler"

CMake found a different compiler. Ensure `cmake --preset win-amd64` is run — the preset explicitly sets `CMAKE_C_COMPILER=clang` and `CMAKE_CXX_COMPILER=clang++`. Do not override these values.

### CMake error: "version < 18"

Your Clang installation is too old. Download Clang 18+ from the LLVM releases page.

### `ninja` is not recognized

Ninja is not on PATH. If you installed via `winget`, close and reopen the terminal. If you installed manually, confirm the folder containing `ninja.exe` is in your PATH.

### Git submodule is empty

Run `git submodule update --init --recursive` from the repository root.

### Ghidra crashes on launch

Ensure JDK 17 or later is installed and `JAVA_HOME` points to the JDK directory. Set it via: **System Properties → Environment Variables → New** with name `JAVA_HOME` and the JDK path.

For more troubleshooting help, see [docs/Troubleshooting.md](docs/Troubleshooting.md).
