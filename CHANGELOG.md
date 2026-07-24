# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Planned
- Generate ReXGlue project from WWE '13 XEX binary
- Initial binary analysis with Ghidra
- First pass kernel stub implementations

---

## [0.1.0] — 2026-07-25

### Added
- Initial repository structure created
- `rexglue-sdk` added as a Git submodule at `rexglue-sdk/`
- `README.md` — project overview, goals, status, FAQ
- `ROADMAP.md` — ten-phase development roadmap with detailed checklists
- `SETUP.md` — complete environment setup guide for all dependencies
- `DEVELOPMENT.md` — developer handbook covering workflow, standards, and build process
- `CONTRIBUTING.md` — contributor guide with branch naming, commit conventions, and PR guidelines
- `CHANGELOG.md` — this file
- `LICENSE` — BSD 3-Clause License
- `.gitignore` — rules for CMake build output, IDE files, and OS artifacts
- `docs/Day01.md` — development journal entry for Day 1
- `docs/Architecture.md` — project architecture with Mermaid diagrams
- `docs/ReXGlue.md` — comprehensive ReXGlue SDK reference
- `docs/Xbox360.md` — Xbox 360 hardware and platform reference
- `docs/Research.md` — research notebook with initial resource links
- `docs/BuildGuide.md` — advanced build and debugging guide
- `docs/Troubleshooting.md` — common problems and solutions handbook
- `docs/Glossary.md` — beginner-friendly glossary of all key terms

### Environment
- Visual Studio 2022 installed and verified
- LLVM / Clang 18+ installed and on PATH
- CMake 3.25+ installed and verified
- Ninja installed and verified
- Python 3.x installed and verified
- Git configured with user identity
- Ghidra installed with JDK 21
- ReXGlue SDK configured with `cmake --preset win-amd64`
- ReXGlue SDK built in Debug configuration

---

[Unreleased]: https://github.com/OWNER/wwe13-recompilation/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/OWNER/wwe13-recompilation/releases/tag/v0.1.0
