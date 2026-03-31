# Building Celestia 1.6.x on Windows

## Prerequisites

- **Visual Studio 2022** (or 2017/2019) with the C++ workload
- **vcpkg** – install it somewhere, e.g. `C:\vcpkg`

## 1. Install vcpkg

```
git clone https://github.com/microsoft/vcpkg C:\vcpkg
C:\vcpkg\bootstrap-vcpkg.bat
```

## 2. Install dependencies via vcpkg

```
C:\vcpkg\vcpkg --triplet=x64-windows install libpng libjpeg-turbo gettext[tools] luajit cspice
C:\vcpkg\vcpkg integrate install
```

> `vcpkg integrate install` makes Visual Studio find vcpkg packages automatically.

## 3. Open and build the solution

1. Open `windows/celestia.sln` in Visual Studio
2. Set configuration to **Release** / **x64**
3. Build → Build Solution (`Ctrl+Shift+B`)

The output exe will be at:
```
windows/Release/Celestia.exe
```

## 4. Run it

Celestia expects its data files (stars, textures, etc.) in a specific layout next to
the exe. The easiest way is to copy the built `Celestia.exe` into an existing Celestia
1.6.x installation folder (from the official installer), replacing the exe there.
