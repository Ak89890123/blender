# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language

與使用者互動時一律使用繁體中文（回覆、說明、提問、PR 描述皆同）。程式碼、程式碼註解與 commit 訊息維持英文，以符合上游 Blender 的慣例。

## Repository context

- This is a **GitHub mirror** of Blender. Development, code review and the bug tracker live on <https://projects.blender.org>; see `.github/README.md` and `.gitea/pull_request_template.yaml`. Cloning the mirror can hit Git LFS errors; use `GIT_LFS_SKIP_SMUDGE=1` for the initial clone.
- The upstream PR template links to a policy on AI-assisted contributions (developer.blender.org handbook, "contributing/ai_contributions"). Check it before preparing anything for upstream.
- Commit subjects follow `Area: Subarea: Description` (e.g. `EEVEE: Allow reference spheres to scale down with camera zoom`, `Fix: Grease Pencil: Carver tool locked materials`, `Fix #164389: EEVEE: ...`). Match this style.
- Precompiled libraries live in `lib/<os>_<arch>` git submodules configured with `update = none`. They are empty until `make update` enables and fetches them (needs access to projects.blender.org). Without them, a build needs system-installed dependencies or `make deps`. Test data in `tests/files/` is tracked in this repo (many binary types are Git LFS, see `.gitattributes`).

## Build

The top-level `GNUmakefile` (Linux/macOS; `make.bat` on Windows) wraps an out-of-source CMake build in `../build_<os>` (override with `BUILD_DIR=...`). Targets combine freely; order doesn't matter, and each suffix gets its own build dir (e.g. `../build_linux_debug`).

```sh
make update                # fetch libs + sub-repos (make update_code: skip libs)
make developer ninja ccache   # recommended dev setup: asserts, ASAN, GTests, compile_commands.json
make debug | lite | full | headless | release | bpy | cycles
make help                  # all convenience targets; `make help_features` lists CMake options
make config                # ccmake/cmake-gui on the build dir
```

- `developer` applies `build_files/cmake/config/blender_developer.cmake` (`WITH_GTESTS=ON`, `WITH_ASSERT_ABORT`, ASAN on non-Windows). The other configs are in `build_files/cmake/config/`. Extra CMake args: `BUILD_CMAKE_ARGS="-DWITH_X=ON"`.
- The `make` build runs `install` into `<build>/bin`, and tests run against that installed layout, so rebuild after changes before testing.
- `make bpy` builds Blender as a Python module (`WITH_PYTHON_MODULE`); GTests are incompatible with it.
- `make deps` builds third-party libraries from `build_files/build_environment` into `lib/` (intended for platform maintainers).

## Tests

`make test` runs `ctest` in the build dir (`build_files/utils/make_test.py`). Most tests are skipped or absent unless configured: GTests need `WITH_GTESTS`, and Python tests need `tests/files/` plus (for render tests) `oiiotool`. UI tests need `WITH_UI_TESTS`; GPU-draw tests use `make test_gpu_draw`.

```sh
cd ../build_linux
ctest -N                        # list registered tests
ctest -R script_pyapi_bpy_app --output-on-failure   # one test by name/regex
```

- **Python tests** (`tests/python/*.py`) run inside Blender and are registered in `tests/python/CMakeLists.txt` via `add_blender_test(name --python <script> ...)`. `TEST_BLENDER_EXE_PARAMS` (`tests/CMakeLists.txt`) adds `--background --factory-startup --debug-memory --debug-exit-on-error --python-exit-code 1`, so any logged error fails the test. To run one by hand: `<build>/bin/blender --background --factory-startup --python tests/python/<script>.py`. A new test script is not run until it is registered in that CMake file.
- **GTests** live next to the code in `source/blender/<module>/tests/` (e.g. `BLI_*_test.cc`) and in `intern/*`, and are registered per library with `blender_add_test_suite_lib` / `blender_add_test_suite_executable` (`build_files/cmake/testing.cmake`). With `WITH_TESTS_SINGLE_BINARY` they build into one `blender_test` binary and each GTest becomes its own ctest, otherwise one `<name>_test` binary per source file. Either binary accepts `--gtest_filter=Suite.Case`. Ctest runs GTests with the install dir as the working directory.
- Performance tests: `tests/performance` (`make benchmark`).

## Formatting and static checks

- `make format [PATHS="source/blender/blenlib ..."]` runs clang-format (pinned to **20.1.8** in `tools/utils_maintenance/clang_format_paths.py`; a copy is looked up in `lib/<platform>/llvm/bin`) and autopep8 (`pyproject.toml`). It also re-tabs `CMakeLists.txt`/`.cmake`/`.sh` files. `extern/` is never formatted.
- Style from `.editorconfig`/`.clang-format`: C/C++/CMake are 2-space indent with a 99-column limit; Python is 4-space with a 120-column limit; `GNUmakefile` uses tabs.
- Other checks, all `make check_*`: `check_cmake` (file lists in CMake vs disk), `check_licenses` (every file needs an SPDX header; allowed identifiers are in `doc/license/SPDX-license-identifiers.txt`), `check_pep8`, `check_mypy`, `check_spelling_{c,py,shaders,cmake}`, `check_cppcheck`, `check_descriptions` (needs a built Blender), `check_docs_file_structure`.
- New source files must be added to the module's `CMakeLists.txt` file lists by hand (`tools/utils_maintenance/cmake_sort_filelists.py` sorts them), otherwise they aren't built; `check_cmake` catches the omission.

## Architecture

Top-level layout:

- `source/blender/` holds the application, split into ~35 modules that each build a static library with prefixed headers (`BLI_`, `BKE_`, `DNA_`, `RNA_`, `ED_`, `WM_`, `GPU_`, `DRW_`, `NOD_`, `FN_`, `GEO_`, `COM_`, ...). C++ headers use `.hh`; C-compatible ones (notably all `DNA_*`) use `.h`. Each file carries an SPDX header and a doxygen `\ingroup` tag (groups in `doc/doxygen/doxygen.source.h`). `source/creator/` is the `main()` and command-line handling (`creator_args.cc`).
- `intern/` is code maintained by Blender itself but with a small, self-contained API (Cycles in `intern/cycles`, `guardedalloc`, `ghost` windowing/OS layer, `clog` logging, `libmv`, ...). `extern/` is vendored third-party code, kept out of formatting and license checks.
- `scripts/` is the Python half of the app. It is installed alongside the binary: `scripts/startup/bl_ui` (UI panels/menus), `scripts/startup/bl_operators`, `scripts/modules` (`bpy_extras`, `addon_utils`, ...), and `scripts/addons_core`. Much of what looks like "the UI" is Python, not C++.
- `release/` (icons, splash, default `startup.blend`, `userdef`, colour management), `locale/` (translations), `build_files/` (CMake modules, config presets, `build_environment` for deps, buildbot/packaging), `tools/` (dev tooling: `check_source`, `utils_maintenance`, ...), `doc/` (doxygen, Python API sphinx generation, `.blend` file format docs).

How the core modules relate (this takes reading several directories):

- **Layering.** `blenlib` (BLI: containers, math, threading, no Blender data knowledge) is the base. `makesdna` defines the data structs, `blenkernel` (BKE) holds data-level logic (`Main`, ID datablocks, evaluation-independent operations), and `depsgraph` schedules evaluation. `editors/*` implements operators and space types (one directory per editor, e.g. `space_view3d`, `mesh`, `sculpt_paint`), `windowmanager` owns windows/events/keymaps/operator dispatch, and `python` exposes everything to `bpy`. Keep dependencies pointing down this stack (e.g. `blenlib` and `blenkernel` must not call into `editors` or `windowmanager`).
- **DNA and RNA are code-generated at build time.** `makesdna` parses `DNA_*_types.h` to produce the struct catalogue serialized into `.blend` files (`intern/dna_rename_defs.h` handles renames). `makesrna` parses `makesrna/intern/rna_*.cc` to generate the RNA property system, which is the single source for the Python API (`bpy.types`/`bpy.props`), the UI property editors, animation/driver paths and library overrides. Adding a property to a struct usually means touching a `DNA_*_types.h` and the matching `rna_*.cc` (and often `blenloader` versioning).
- **File compatibility.** `.blend` files are DNA dumps, so changing DNA layout or default values needs versioning code in `blenloader/intern/versioning_<version>.cc` (dispatched from `readfile.cc`). Bump `BLENDER_FILE_SUBVERSION` in `blenkernel/BKE_blender_version.h` and guard the new code with `MAIN_VERSION_FILE_ATLEAST`; `versioning_xxx_template.cc` is the template for a new release series. Default values for new structs are `_DNA_DEFAULT_*` macros inside the `DNA_*_types.h` headers; defaults for the startup file are patched in `versioning_defaults.cc`.
- **Evaluation and drawing.** Scene data is copied into an evaluated depsgraph before drawing or rendering. `draw` contains the viewport engines (EEVEE, Workbench, overlay, ...) and sits on the `gpu` abstraction, which has OpenGL, Vulkan and Metal backends. `render` is the render-engine API; Cycles (`intern/cycles`) is a separate library that plugs in through `intern/cycles/blender`.
- **Nodes.** `nodes` implements shader/geometry/compositor/texture node types (`NOD_*` headers). Geometry Nodes evaluate through `functions` (lazy-function graphs and multi-function/field system) on top of `geometry` (`GEO_*` algorithms) and `bmesh`/`blenkernel` geometry types. The GPU compositor is in `compositor`.
- **Python bindings** are in `source/blender/python`: `intern` (`bpy`, `bpy_rna`, operators, `bpy.app`), `generic` (non-Blender-specific helpers), `mathutils`, `bmesh`, `gpu`. `bpy_rna` is generic over the RNA layer, so most API changes come from the RNA files rather than hand-written bindings.
