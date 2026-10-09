# Clang-Tidy Check

Every pull request that changes C++ under `src/` or `tests/` runs a `clang-tidy` check in CI. For now it checks one thing: that a file includes the header for every symbol it uses, instead of relying on the precompiled header or on another header's includes. Code that relies on those builds normally but breaks the build without the precompiled header, or as soon as an unrelated header stops including something.

`scripts/run_clang_tidy.sh` (Linux and macOS) and `scripts\run_clang_tidy.ps1` (Windows) run the same check on your machine, so you can fix what it reports before you push.

- [What the Check Reports](#what-the-check-reports)
- [Before You Start](#before-you-start)
- [Linux](#linux)
- [macOS](#macos)
- [Windows](#windows)
- [Options](#options)
- [Fixing What It Reports](#fixing-what-it-reports)
- [How Closely It Matches CI](#how-closely-it-matches-ci)
- [Troubleshooting](#troubleshooting)

## What the Check Reports

The checks come from `.clang-tidy` in the repository root. The check only looks at what your branch changes compared with `main`:

- On the lines you added or changed, a symbol whose header the file does not include fails the check. Older code on lines you didn't touch is not held to it.
- A source file you changed must compile on its own, without the precompiled header.
- A header you changed is compiled on its own and fails only on errors your change adds.
- If you delete an `#include`, every use of what it provided fails, including on lines you didn't touch.

Headers that are a library's internals rather than the one to include, such as wxWidgets' per-platform headers or Boost and oneTBB internals, are listed under `IgnoreHeaders` in `.clang-tidy` and never asked for.

The scripts check uncommitted changes too, so you can run them while you work.

## Before You Start

The check needs a compile database, which comes from configuring OrcaSlicer, so you need [OrcaSlicer's dependencies built](how_to_build) for your platform. The scripts configure a separate `build-tidy` directory without the precompiled header and leave your normal build alone. They don't compile OrcaSlicer, so a run takes minutes rather than a full build.

They also need a few tools, and `clang-tidy` at exactly the version CI uses, which is pinned in `scripts/clang_tidy_requirements.txt`. A newer or older `clang-tidy` can report different findings, so the scripts install the pinned version into a Python virtual environment inside `build-tidy` rather than using one on your `PATH`.

When something is missing, the script names it and asks whether to install it. Answer `y` and it runs the install command. Answer `n`, or run it without a terminal, and it prints the command so you can run it yourself. Pass `--yes` (`-Yes` on Windows) to install without asking.

## Linux

Run from the repository root:

```shell
./scripts/run_clang_tidy.sh
```

It needs `git`, `python3` with `venv`, `cmake`, `ninja` and `clang`. It uses clang even if you normally build with GCC, because CI does and GCC's flags change what `clang-tidy` sees. To install them yourself:

```shell
# Debian, Ubuntu
sudo apt-get install -y git python3 python3-venv cmake ninja-build clang
# Fedora
sudo dnf install -y git python3 cmake ninja-build clang
# Arch
sudo pacman -S --needed git python cmake ninja clang
```

The dependencies are expected in `deps/build`, where `./build_linux.sh -d` puts them. If configuring fails on a missing system library, install them with `./build_linux.sh -u`.

## macOS

Run from the repository root:

```shell
./scripts/run_clang_tidy.sh
```

It needs `git`, `python3`, `cmake` and `ninja`, and the Xcode command line tools for the compiler. To install them yourself:

```shell
xcode-select --install
brew install git python cmake ninja
```

The dependencies are expected in `deps/build/arm64` on Apple silicon and `deps/build/x86_64` on Intel, where `./build_release_macos.sh -d -a <arch>` puts them.

## Windows

Run from the repository root in PowerShell:

```pwsh
powershell -ExecutionPolicy Bypass -File scripts\run_clang_tidy.ps1
```

It needs Git, Python 3 and Visual Studio with the C++ tools, whose CMake component provides CMake and Ninja. The script loads the Visual Studio developer environment itself, so any terminal works. To install them yourself:

```pwsh
build_win.bat --install-deps
build_win.bat --install-vs buildtools
winget install -e --id Python.Python.3.12
```

`--install-deps` installs CMake, Perl and Git, and `--install-vs` installs the Visual Studio Build Tools; see [Build on Windows](how_to_build_windows) for both. Open a new terminal after installing so `PATH` picks them up.

The script uses the dependency tree `build_win.bat -d` made: `deps\build` for MSVC or `deps\build-clang` for clang-cl, with `-arm64` appended on ARM64. It configures with the same compiler the tree was built with.

## Options

| Linux, macOS | Windows | What it does |
|---|---|---|
| `--fix` | `-Fix` | Add the missing includes `clang-tidy` names. |
| `-b`, `--base <rev>` | `-Base <rev>` | Compare against another revision. By default the script compares against `main` of the remote that points at `OrcaSlicer/OrcaSlicer`, so it works from a fork, and fetches it first. |
| `--no-fetch` | `-NoFetch` | Don't fetch `main` first. |
| `-B`, `--build-dir <dir>` | `-BuildDir <dir>` | Use another build directory than `build-tidy`. |
| `-d`, `--deps-dir <dir>` | `-DepsDir <dir>` | Use dependencies built somewhere else. |
| `-j`, `--jobs <n>` | `-Jobs <n>` | Limit how many files are checked at once. |
| `-y`, `--yes` | `-Yes` | Install missing tools without asking. |
| | `-Arch <x64\|arm64>` | Check for another architecture than this machine's. |

Set the `CLANG_TIDY` environment variable to use a `clang-tidy` you installed yourself; the script warns if it isn't the pinned version.

## Fixing What It Reports

Each finding names the file, the line and the symbol, and the header that provides it:

```
src/slic3r/GUI/Plater.cpp:1234:5: error: no header providing "Slic3r::Polygon" is directly included [misc-include-cleaner,-warnings-as-errors] (add #include <libslic3r/Polygon.hpp>)
```

Add the include it names, or run the script with `--fix` to add them all, then review the result before committing:

- `clang-tidy` spells every header with angle brackets. OrcaSlicer quotes its own headers, so write `#include "libslic3r/Polygon.hpp"`, and change the ones `--fix` added.
- Keep each include with the file's other includes, outside any `#if` block, unless the symbol is only used inside one.
- In a file that starts with a Windows block (`_WIN32_WINNT`, `NOMINMAX`, `<Windows.h>`), keep new includes below it.
- If it asks for a library's internal header instead of the one you include, that header belongs in `IgnoreHeaders` in `.clang-tidy`; say so in your pull request.

Run the script again until it reports `clang-tidy passed.`

## How Closely It Matches CI

The script runs what the CI job runs: the same `clang-tidy` version, the same configure options, the same `.clang-tidy` and the same `scripts/clang_tidy_diff.py`, compared against `main`. On Linux, with your branch up to date with `main`, it reports what CI reports.

It can differ in three ways:

- CI checks your branch merged into the latest `main`. If `main` has changed the same files since you branched, rebase to see the same result.
- CI runs on Linux. On Windows and macOS, `clang-tidy` also sees code inside `_WIN32` or `__APPLE__` blocks and that platform's standard library, so it can report findings CI does not, and miss ones in Linux-only code. A change that passes on your platform and on Linux passes CI.
- The script also checks uncommitted changes, which CI never sees.

## Troubleshooting

- **Configuring failed**: the last lines of the CMake output are printed, and the full log is `build-tidy/configure.log`. On Linux, a missing system library usually means `./build_linux.sh -u` hasn't been run. A wrong dependency path can be fixed with `--deps-dir`.
- **Could not create a Python virtual environment**: on Debian and Ubuntu, `venv` is a separate package, `python3-venv`.
- **Warning that `clang-tidy` is not the pinned version**: `CLANG_TIDY` points at another version. Unset it to use the pinned one.
- **`No changed C++ lines to check.`**: your branch has no C++ changes under `src/` or `tests/` compared with `main`.
- **Many unrelated findings**: you are probably comparing against an old `main`. Let the script fetch it, or rebase.
