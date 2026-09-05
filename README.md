# C++23 build and CI infrastructure template

A minimal, cross-platform C++23 starter repository with a production-minded development workflow.

It provides a small demonstration executable plus a complete development environment for Windows and Linux, including CMake, Ninja, formatting, static analysis, and editor integration. Use it as a clean starting point for command-line tools, libraries, experiments, or larger C++ projects.

## Included Tooling

* CMake 3.25+ and Ninja builds
* C++23 support
* Windows x64 builds using MSVC
* Linux builds using Clang
* Debug and Release configurations
* `clang-format` enforcement
* `clang-tidy` analysis with warnings treated as errors
* Optional CodeQL security scanning in CI
* GitHub Actions with reusable composite actions
* `compile_commands.json` generation for `clangd` and `clang-tidy`
* A PowerShell Windows build wrapper
* CI logs and compilation-database artifact uploads

## Repository Layout

```text
.
├── src/                              # C++ source files and demonstration entry point
├── .github/
│   ├── actions/
│   │   ├── process-analysis-config/
│   │   │   └── action.yml
│   │   ├── setup-cpp-dependencies/
│   │   │   └── action.yml
│   │   ├── setup-cpp-environment/
│   │   │   └── action.yml
│   │   ├── check-formatting/
│   │   │   └── action.yml
│   │   ├── configure-cmake/
│   │   │   └── action.yml
│   │   ├── post-process-compile-commands/
│   │   │   └── action.yml
│   │   ├── build-project/
│   │   │   └── action.yml
│   │   └── run-clang-tidy/
│   │       └── action.yml
│   ├── scripts/
│   │   ├── run-clang-tidy.py
│   │   └── RUN_CLANG_TIDY_README.md
│   ├── workflows/
│   │   └── static-code-analysis.yml
│   ├── static-code-analysis.json
│   └── CI.md
├── .clang-format                     # Formatting policy
├── .clang-tidy                       # Static-analysis and naming rules
├── .clangd                           # clangd configuration
├── CMakeLists.txt
├── CMakePresets.json
├── build-x64.ps1                     # Windows x64 MSVC build helper
└── README.md
```

## Requirements

### Windows

* Visual Studio or Visual Studio Build Tools with the MSVC C++ toolchain
* CMake 3.25+
* Ninja
* PowerShell
* `vswhere.exe`, normally installed with Visual Studio
* LLVM 23.1.0 tools when running formatting or static analysis locally:

  * `clang-format`
  * `clang-tidy`

### Linux

* CMake 3.25+
* Ninja
* A C++23-capable Clang toolchain
* Python when running the repository's clang-tidy runner directly
* LLVM 23.x tools:

  * `clang++`
  * `clang-format`
  * `clang-tidy`

## Build on Windows

The `build-x64.ps1` helper locates and initializes the installed MSVC environment, then configures and builds through the appropriate CMake preset.

Build Debug:

```powershell
.\build-x64.ps1
```

Build Release:

```powershell
.\build-x64.ps1 -Configuration Release
```

Remove the selected build directory before building:

```powershell
.\build-x64.ps1 -Clean
```

Clean and build Release:

```powershell
.\build-x64.ps1 -Configuration Release -Clean
```

You can also invoke the presets directly:

```powershell
cmake --preset x64-debug
cmake --build --preset x64-debug
```

```powershell
cmake --preset x64-release
cmake --build --preset x64-release
```

Build output is written beneath the associated preset build directory. Refer to `CMakeLists.txt` and `CMakePresets.json` for the target name and exact executable location rather than assuming a binary name.

## Build on Linux

Configure a Ninja build using Clang:

```bash
cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++
```

Build it:

```bash
cmake --build build --config Debug -j
```

For a Release build, substitute `Release` for `Debug` in both commands.

## Run the Demonstration

The repository contains an executable built from the source file that defines `main()`.

After building, run the executable from the active build directory. The executable target and output layout are defined by `CMakeLists.txt` and `CMakePresets.json`.

For example, on Windows the output commonly follows this shape:

```powershell
.\build\x64-debug\bin\<target-name>.exe
```

On Linux:

```bash
./build/bin/<target-name>
```

The expected behavior is documented in `.github/demonstration-validation.md` when that file is present. If it differs from the behavior implemented in `main()`, treat that as a documentation/code discrepancy to resolve deliberately.

## Code Formatting

The project uses a Google-derived `.clang-format` configuration.

The formatting policy includes:

* Two-space indentation
* 80-column limit
* Repository-wide C++ file discovery in CI

Check formatting locally:

```powershell
clang-format --style=file --dry-run --Werror src/*.cpp
```

Apply formatting:

```powershell
clang-format -i src/*.cpp
```

On Linux, use your shell's preferred file expansion or pass explicit file paths:

```bash
clang-format --style=file --dry-run --Werror src/*.cpp
clang-format -i src/*.cpp
```

CI automatically discovers tracked files ending in `.cpp`, `.h`, `.hpp`, `.cc`, and `.cxx` throughout the repository.

## Static Analysis

The `.clang-tidy` configuration enables a broad set of checks, including:

* Bug-prone and Clang Static Analyzer checks
* Concurrency checks
* C++ Core Guidelines checks
* Google and LLVM checks
* Modernization and performance checks
* Portability and readability checks
* Selected HICPP and CERT checks

All clang-tidy warnings fail validation. Prefer fixing the underlying issue over suppressing it. Use a narrowly scoped `NOLINT` only when the exception is intentional and can be justified.

After configuring the project, run:

```powershell
python .github\scripts\run-clang-tidy.py -p build "src/.*"
```

On Windows, compilation commands generated for MSVC require Clang's MSVC driver mode:

```powershell
python .github\scripts\run-clang-tidy.py `
  -p build `
  --extra-arg-before=--driver-mode=cl `
  "src/.*"
```

Do not pass `--driver-mode=cl` for Linux Clang compilation commands.

The repository uses a vendored runner because it supports reliable parallel execution and avoids depending on distribution-specific availability of `run-clang-tidy.py`.

## CI Pipeline

GitHub Actions provides cross-platform validation for the project.

The CI pipeline runs on Windows and Linux and validates the configured build matrix, including:

* Debug and Release builds
* `clang-format`
* `compile_commands.json` generation
* CodeQL analysis
* `clang-tidy`

The pipeline is triggered by pushes, pull requests, scheduled runs, and manual dispatch. Documentation-only changes are excluded according to the repository's CI trigger policy.

The analysis matrix and optional features are controlled through:

```text
.github/static-code-analysis.json
```

The complete CI architecture, trigger policy, configuration validation, workflow sequence, composite actions, artifacts, failure behavior, and maintenance guidance are documented in:

**[`.github/CI.md`](.github/CI.md)**

## Development Conventions

* Keep code portable between Windows and Linux.
* Isolate platform-specific code behind clear platform guards.
* Use modern C++23 facilities where they improve clarity and safety.
* Preserve clear ownership and exception-safety boundaries.
* Follow the repository's `.clang-format` and `.clang-tidy` policies.
* Follow the existing naming conventions:

  * Functions and methods: `CamelCase`
  * Local variables and parameters: `lower_case`
  * `constexpr` variables: `kCamelCase`
  * Namespaces: `lower_case`
* Use `static` for internal-linkage free functions and objects, rather than anonymous namespaces.
* Avoid unnecessary dependencies and speculative abstractions.

## Editor Integration

The project generates a compilation database at:

```text
build/compile_commands.json
```

The `.clangd` file configures `clangd` to use the compilation database, enabling accurate diagnostics, completion, navigation, and static-analysis support in compatible editors.

After changing build configuration, reconfigure the project so `compile_commands.json` reflects the active compiler flags and include paths.

## Customizing the Template

To turn this template into a project:

1. Rename or replace the source files under `src/`.
2. Update the executable target, output layout, and install rules in `CMakeLists.txt`.
3. Update preset names or configuration details in `CMakePresets.json` if needed.
4. Update this README with the project's public API, usage examples, and requirements.
5. Update `.github/static-code-analysis.json` if the CI analysis matrix or optional analysis features need to change.
6. Update `.github/CI.md` whenever CI architecture or behavior changes.

---

**Last Updated:** 09/2026
**Maintainer:** AmitGDev
