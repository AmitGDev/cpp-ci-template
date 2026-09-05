# C++23 build and CI infrastructure template

A minimal, cross-platform C++23 starter repository with a production-minded development workflow.

It provides a small demonstration executable plus a complete development environment for Windows and Linux, including CMake, Ninja, formatting, static analysis, and editor integration. Use it as a clean starting point for command-line tools, libraries, experiments, or larger C++ projects.

## Included Tooling

- CMake 3.25+ and Ninja builds
- C++23 support
- Windows x64 builds using MSVC
- Linux builds using Clang++
- Debug and Release configurations
- clang-format enforcement
- clang-tidy analysis with warnings treated as errors
- Optional CodeQL security scanning in CI
- GitHub Actions with reusable composite actions
- `compile_commands.json` generation for clangd and clang-tidy
- A PowerShell Windows build wrapper
- CI logs and compilation-database artifact uploads

## Repository Layout

```
.
├── .github/                          # Generic CI infrastructure
├── .github-ext/                      # Optional repository-specific CI configuration/setup
├── .clang-format                     # Formatting policy
├── .clang-tidy                       # Static-analysis and naming rules
├── .clangd                           # clangd configuration
├── src/                              # C++ source files and demonstration entry point
├── CMakeLists.txt
├── CMakePresets.json
├── build-x64.ps1                     # Windows x64 MSVC build helper
└── README.md
```

The `.github/` directory contains the generic CI infrastructure and should normally not be modified in a repository derived from this template.

Repository-specific CI configuration and environment setup belong in `.github-ext/`.

## Requirements

### Windows

- Visual Studio or Visual Studio Build Tools with the MSVC C++ toolchain
- CMake 3.25+
- PowerShell 7
- `vswhere.exe`, normally installed with Visual Studio
- Python when running extension scripts in `.github-ext`
- LLVM 23.1.0 tools when running formatting or static analysis locally:
    - `clang-format`
    - `clang-tidy`
- Ninja 1.13.2

### Linux

- A C++23-capable Clang toolchain
- CMake 3.25+
- Python when running extension scripts in `.github-ext`
- LLVM 23.x tools:
    - `clang++`
    - `clang-format`
    - `clang-tidy`
- Ninja 1.13.2

## Build on Windows

The `build-x64.ps1` helper locates and initializes the installed MSVC environment, then configures and builds through the appropriate CMake preset.

Build Debug:

```
.\build-x64.ps1
```

Build Release:

```
.\build-x64.ps1 -Configuration Release
```

Remove the selected build directory before building:

```
.\build-x64.ps1 -Clean
```

Clean and build Release:

```
.\build-x64.ps1 -Configuration Release -Clean
```

The build script also prepares the compilation database for clangd and clang-tidy. If repository-specific compile-command post-processing is required, the project-specific processing is applied; otherwise the generated database is used unchanged.

You can also invoke the CMake presets directly:

```
cmake --preset x64-debug
cmake --build --preset x64-debug
```

```
cmake --preset x64-release
cmake --build --preset x64-release
```

The direct preset invocation does not perform any additional synchronization performed by `build-x64.ps1`.

Build output is written beneath the associated preset build directory. Refer to `CMakeLists.txt` and `CMakePresets.json` for the target name and exact executable location rather than assuming a binary name.

## Build on Linux

Use the same CMake presets that are used by CI:

```
cmake --preset linux-debug
cmake --build --preset linux-debug
```

For Release:

```
cmake --preset linux-release
cmake --build --preset linux-release
```

`CMakePresets.json` is the single source of truth for compiler, generator, build type, architecture, binary directory, and other build configuration.

## Run

The repository contains a minimal "Hello World" executable built from the source file that defines `main()`.

The executable exists primarily as a minimal build target for validating the CI infrastructure. It is not intended to provide useful application functionality.

After building, run the executable from the active build directory. The executable target and output layout are defined by `CMakeLists.txt` and `CMakePresets.json`.

For example, on Windows the output commonly follows this shape:

```
.\build\x64-debug\bin\main.exe
```

On Linux:

```
./build/linux-debug/bin/main
```

The expected behavior is a `Hello World` message.

## Static Code Analysis

SCA definitions are mirrored from:

<https://github.com/AmitGDev/cpp-coding-standard> 

## Code Formatting

Check formatting locally:

```
clang-format --style=file --dry-run --Werror src/*.cpp
```

Apply formatting:

```
clang-format -i src/*.cpp
```

On Linux, use your shell's preferred file expansion or pass explicit file paths.

CI automatically discovers tracked files ending in `.cpp`, `.h`, `.hpp`, `.cc`, and `.cxx` throughout the repository.

## Static Analysis

Use a narrowly scoped `NOLINT` only when the exception is intentional and can be justified.

After configuring the project, run:

```
clang-tidy -p build "src/.*"
```

The CI pipeline runs clang-tidy against the compilation database generated by the configured CMake preset.

## CI Pipeline

GitHub Actions provides cross-platform validation for the project.

The CI pipeline runs on Windows and Linux and validates the configured build matrix, including:

- Debug and Release builds
- clang-format
- `compile_commands.json` generation
- clang-tidy
- optional CodeQL analysis

The pipeline is triggered by pushes, pull requests, scheduled runs, and manual dispatch. Documentation-only changes are excluded according to the repository's CI trigger policy.

The analysis matrix and optional features are controlled through the repository-specific configuration:

```
.github-ext/static-code-analysis.json
```

The complete CI architecture, trigger policy, configuration validation, workflow sequence, composite actions, artifacts, failure behavior, and maintenance guidance are documented in:

```
.github/CI.md
```

## Preset Contracts

This template's build configuration lives in exactly one place, `CMakePresets.json`, and two other parts of the template are built against it as fixed contracts. This section only documents those contracts; for the CI/SCA architecture itself, see [CI.md](https://amitgdev.atlassian.net/wiki/spaces/SD/pages/118816769/CI.md).

### `CMakePresets.json` ↔ `CMakeLists.txt`

Each preset owns the compiler, generator, build type, architecture, and binary directory for one configuration. `CMakeLists.txt` does not hardcode any of these - it consumes what the active preset supplies (`CMAKE_BUILD_TYPE`, `CMAKE_CXX_COMPILER`, and any preset `cacheVariables`) and defines the target(s) on top of that. Adding or changing a build configuration means editing `CMakePresets.json`, not `CMakeLists.txt`.

### `CMakePresets.json` ↔ SCA pipeline

The pipeline drives the project through the same presets rather than reconstructing compiler flags itself. For each `os`/`configurations` matrix entry (from `.github-ext/static-code-analysis.json`, or `.github/static-code-analysis.json` if no override exists), the pipeline lowercases the configuration name and selects the preset `windows-<config>` or `linux-<config>`, then runs `cmake --preset <name>` and `cmake --build --preset <name>`. A preset renamed or added in `CMakePresets.json` must keep this `<os>-<config>` naming convention, or the corresponding matrix entry has no preset to configure against. Full resolution and validation details are in [CI.md](https://amitgdev.atlassian.net/wiki/spaces/SD/pages/118816769/CI.md).

## CI Development Conventions

- Keep `.github/` generic and reusable.
- Put repository-specific CI configuration in `.github-ext/`.
- Do not add project-specific dependencies or tools to generic composite actions.
- Use `CMakePresets.json` as the single source of truth for build configuration.
- Add repository-specific environment setup through `.github-ext/setup_environment.py`.
- Keep compiler-specific compilation-database processing repository-specific.

## Editor Integration

The CI and local build configuration generate a compilation database:

```
build/compile_commands.json
```

The `.clangd` file configures clangd to use the compilation database, enabling accurate diagnostics, completion, navigation, and static-analysis support in compatible editors.

After changing build configuration, reconfigure the project so `compile_commands.json` reflects the active compiler flags and include paths.

## Customizing the Template

To turn this template into a project:

1. Rename or replace the source files under `src/`.
2. Update `CMakeLists.txt`.
3. Update `CMakePresets.json` with the project's required build configuration.
4. Update this README with the project's public API, usage examples, and requirements.
5. Copy `.github/static-code-analysis.json` to `.github-ext/static-code-analysis.json` and modify the repository-specific analysis matrix or optional analysis features as required.
6. If the project requires additional dependencies, SDKs, tools, environment variables, or other CI setup, implement them in `.github-ext/setup_environment.py`.
7. Do not modify `.github/` if you want to receive drag-and-drop CI infrastructure updates. Put repository-specific CI configuration and setup in `.github-ext/`.

---

**Last Updated:** 2026-09-22  
**Maintainer:** AmitGDev
