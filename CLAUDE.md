# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Branch: 1.6.x** – this is a fork of the official Celestia 1.6.x branch, not master.
> The master branch uses CMake; this branch uses a Visual Studio solution.

## Build System

**1.6.x uses Visual Studio solution files, not CMake.**

### Dependencies (via vcpkg)

```
C:\vcpkg\vcpkg --triplet=x64-windows install libpng libjpeg-turbo gettext[tools] luajit cspice
C:\vcpkg\vcpkg integrate install
```

### Building

1. Open `celestia.sln` in Visual Studio 2022
2. Set configuration to **Release** / **x64**
3. Build → Build Solution (`Ctrl+Shift+B`)

Output: `x64/Release/celestia.exe`

## CI

GitHub Actions (`.github/workflows/ci.yml`) triggers on push to `1.6.x`.

- Installs vcpkg dependencies (cached via GitHub Packages NuGet feed at `nuget.pkg.github.com/jurakovic`)
- Builds x64 with MSBuild
- Uploads raw build output as artifact `celestia-x64-build` (exe + DLLs)
- Packages with Inno Setup (`celestia.iss`) → produces `Output/celestia-<version>.exe`
- Uploads installer as artifact `celestia-windows-installer`

## Architecture

Key source files for orbital mechanics (the main area of changes in this fork):

| File | Role |
|------|------|
| `src/celengine/orbit.h` | `EllipticalOrbit` class declaration |
| `src/celengine/orbit.cpp` | All orbital mechanics: position, velocity, sampling |
| `src/celengine/parseobject.cpp` | Parses `.ssc` orbit blocks into `EllipticalOrbit` objects |

`EllipticalOrbit` handles all conic sections (elliptic, parabolic, hyperbolic) via `eccentricity`:
- `e < 1` – elliptic (closed orbit)
- `e == 1` – parabolic (Barker's equation, Cardano closed-form)
- `e > 1` – hyperbolic (Laguerre-Conway iteration)

## Releasing a New Version

Three files must be updated together before tagging a release:

| File | What to change |
|------|----------------|
| `celestia.iss` | `#define CelestiaVersion "x.x.x.x"` |
| `src/celestia/res/resource.h` | `#define VERSION_STRING "x.x.x.x"` – controls About dialog and splash screen |
| `src/celestia/res/celestia.rc` | `FILEVERSION`, `PRODUCTVERSION`, `FileVersion`, `ProductVersion` – controls exe file properties |

After updating all three, commit and push. Then run the **Publish** workflow (`publish.yml`) with the version number — it will build, tag, package, and create the GitHub release automatically with all artifacts attached.
