# Contributing

Thank you for your interest in contributing to the WWE '13 Recompilation project. This guide explains how to get set up, how to submit changes, and what is expected of contributors.

Contributions of all kinds are welcome: code, documentation, research notes, bug reports, and questions. This is a learning-oriented project — if you are new to reverse engineering or recompilation, you are welcome here.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setting Up Your Fork](#setting-up-your-fork)
- [How to Submit Changes](#how-to-submit-changes)
- [Branch Naming](#branch-naming)
- [Commit Conventions](#commit-conventions)
- [Pull Request Guidelines](#pull-request-guidelines)
- [Reporting Issues](#reporting-issues)
- [Coding Style](#coding-style)
- [Documentation Style](#documentation-style)
- [Review Expectations](#review-expectations)

---

## Prerequisites

Before contributing, make sure you have a working development environment:

1. Complete the setup in [SETUP.md](SETUP.md)
2. Confirm you can build the project with `cmake --preset win-amd64` and `cmake --build`
3. Read [DEVELOPMENT.md](DEVELOPMENT.md) for the full developer handbook
4. Read [docs/ReXGlue.md](docs/ReXGlue.md) to understand the SDK you are working with

You do not need to be an expert in reverse engineering or Xbox 360 architecture to contribute. Research, documentation, and stub implementations are valuable at any skill level.

---

## Setting Up Your Fork

1. **Fork** the repository on GitHub using the Fork button.

2. **Clone** your fork locally:
   ```cmd
   git clone https://github.com/YOUR_USERNAME/wwe13-recompilation.git
   cd wwe13-recompilation
   ```

3. **Initialize submodules:**
   ```cmd
   git submodule update --init --recursive
   ```

4. **Add the upstream remote** so you can pull future changes:
   ```cmd
   git remote add upstream https://github.com/OWNER/wwe13-recompilation.git
   ```

5. **Verify your setup:**
   ```cmd
   cmake --preset win-amd64
   cmake --build out/build/win-amd64 --config Debug
   ```

---

## How to Submit Changes

1. **Sync with upstream** before starting work:
   ```cmd
   git fetch upstream
   git checkout development
   git merge upstream/development
   ```

2. **Create a branch** from `development`:
   ```cmd
   git checkout -b feature/your-feature-name
   ```

3. **Make your changes.** Follow the coding and documentation standards below.

4. **Build and verify** your changes compile:
   ```cmd
   cmake --build out/build/win-amd64 --config Debug
   ```

5. **Commit** your changes following the [commit conventions](#commit-conventions).

6. **Push** to your fork:
   ```cmd
   git push -u origin feature/your-feature-name
   ```

7. **Open a Pull Request** on GitHub targeting the `development` branch of the upstream repository.

---

## Branch Naming

Use a descriptive, lowercase, hyphen-separated name with a type prefix:

| Prefix | Use for |
|--------|---------|
| `feature/` | New hooks, new capabilities, new subsystems |
| `fix/` | Bug fixes |
| `docs/` | Documentation-only changes |
| `stub/` | Adding or improving stub implementations |
| `research/` | Adding research notes or analysis |
| `chore/` | Build system, tooling, or dependency updates |

**Examples:**

```
feature/xinput-hook-implementation
fix/rendering-crash-on-null-command-buffer
docs/xbox360-calling-convention
stub/xboxkrnl-memory-exports
research/ghidra-startup-analysis
```

Keep branch names short — under 50 characters is ideal.

---

## Commit Conventions

This project follows [Conventional Commits](https://www.conventionalcommits.org/).

### Format

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### Types

| Type | Use for |
|------|---------|
| `feat` | New hook, feature, or capability |
| `fix` | Bug fix |
| `docs` | Documentation changes only |
| `chore` | Build system, submodule update, tooling |
| `refactor` | Code restructuring without behavior change |
| `stub` | New stub implementation |
| `research` | Research notes or binary analysis findings |
| `perf` | Performance improvement |

### Scopes

Use the affected area: `hooks`, `stubs`, `audio`, `input`, `graphics`, `kernel`, `build`, `docs`, `codegen`

### Examples

```
feat(hooks): implement XInputGetState for SDL3 backend

Reads controller state from SDL_GetGamepadState and maps buttons
to the Xbox 360 XINPUT_GAMEPAD layout expected by the game.

Closes #12
```

```
stub(kernel): add REX_STUB for all xboxkrnl memory exports

XMemAlloc, XMemFree, XMemProtect, MmAllocatePhysicalMemory,
MmFreePhysicalMemory - all return 0 with a warning log.
```

```
docs(xbox360): document PPC calling convention and register usage
```

### Subject Rules

- Imperative mood: "add", "fix", "implement" — not "added", "fixes", "implements"
- No capital letter at the start
- No period at the end
- Maximum 72 characters

---

## Pull Request Guidelines

### Title

Use the same format as a commit message: `type(scope): subject`

Example: `feat(hooks): implement XInputGetState for SDL3 backend`

### Description

Your PR description must include:

1. **What** — What does this PR change?
2. **Why** — Why is this change needed?
3. **How** — Brief explanation of the approach if not obvious
4. **Testing** — How did you verify the change works?
5. **Issues** — Reference any related issues with `Closes #N` or `Relates to #N`

### Checklist

Before submitting, confirm:

- [ ] The code compiles in Debug and Release configurations
- [ ] No new compiler warnings or errors
- [ ] All new hooks have a comment explaining the original PPC function
- [ ] CHANGELOG.md is updated under `[Unreleased]`
- [ ] Documentation is updated if the change affects behavior or APIs
- [ ] Commit messages follow the Conventional Commits format

### Scope

Keep PRs focused. A PR that implements one hook is better than a PR that implements ten. Smaller PRs are easier to review and less likely to have conflicts.

Documentation-only PRs are always welcome and do not need code review — they only need a quick check for accuracy.

---

## Reporting Issues

Use GitHub Issues to report bugs, request features, or ask about research.

### Bug Reports

Include:
- What you were doing when the bug occurred
- The exact error message or crash output (from the log file or terminal)
- Your operating system and GPU
- The build configuration (Debug / Release / RelWithDebInfo)
- Steps to reproduce

### Research Questions

If you have found something interesting in the binary but do not know what it is, open an issue with:
- The function address in the XEX binary (e.g. `sub_82012345`)
- The disassembly or decompiled output from Ghidra
- Your hypothesis about what it does
- Any related kernel imports it calls

Collaborative reverse engineering is one of the most valuable ways to contribute.

### Feature Requests

Describe the capability you want, why it is useful, and (if possible) how it might be implemented.

---

## Coding Style

Follow the standards in [DEVELOPMENT.md](DEVELOPMENT.md#coding-standards). Key points:

- C++23, Clang 18+, no compiler-specific extensions
- `PascalCase` for types and functions, `snake_case` for variables and files
- All hook implementations must have a comment explaining the original function
- Use `REX_LOAD_U32` / `REX_STORE_U32` macros for all guest memory access — never raw pointer arithmetic
- Use `REX_STUB` for unimplemented functions, not empty function bodies
- Prefer `REXKRNL_WARN` over `printf` or `std::cout` for diagnostic output

The `.clang-format` and `.clang-tidy` files in the repository root are authoritative. Format your code with:

```cmd
clang-format -i src/**/*.cpp src/**/*.h
```

---

## Documentation Style

Documentation is held to the same standard as code.

- Write in clear, plain English. Avoid jargon without explanation.
- When introducing a concept (reverse engineering, PPC, XEX, etc.), explain it before using it.
- Use tables for comparisons, lists of items with multiple properties, or reference data.
- Use Mermaid diagrams (` ```mermaid `) for architecture diagrams and flows.
- Use code blocks with language identifiers for all code, commands, and file contents.
- Internal links between documents use relative paths: `[ReXGlue](docs/ReXGlue.md)`
- New technical terms should be added to [docs/Glossary.md](docs/Glossary.md)
- Keep paragraphs focused: one idea per paragraph.

---

## Review Expectations

### As an Author

- Respond to review comments promptly. If you disagree, explain your reasoning.
- Make requested changes in new commits, do not force-push during review.
- If a review suggests a fundamentally different approach, discuss before rewriting.

### As a Reviewer

- Be respectful and constructive. This is a learning project — not everyone has reverse engineering experience.
- Explain *why* something should change, not just that it should.
- Distinguish between required changes ("this is incorrect") and suggestions ("this could be cleaner").
- Approve promptly once issues are resolved — do not let PRs stall.
- For documentation PRs, focus on accuracy and clarity, not style preferences.

---

## Questions?

If you have questions about the project, the codebase, or how to contribute:
- Open a GitHub Discussion or Issue
- Ask in the comments of a relevant issue
- Check [docs/Glossary.md](docs/Glossary.md) and [docs/ReXGlue.md](docs/ReXGlue.md) first — your question may already be answered

All questions are welcome. There are no stupid questions in a learning project.
