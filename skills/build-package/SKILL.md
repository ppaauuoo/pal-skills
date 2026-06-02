---
name: build-package
description: Build a Python app into a Windows EXE with PyInstaller then package it into a Windows installer with Inno Setup. Use when user says "build the package", "build installer", "package the app", "create setup exe", or asks to produce a distributable Windows installer from a Python project.
---

# build-package

Two-step Windows packaging pipeline: PyInstaller → EXE, Inno Setup → installer.

## Prerequisites

- Python project with `pyinstaller` available (via `uv`, `pip`, or `conda`)
- Inno Setup 6.x — `iscc.exe` on `PATH` (or set `ISCC` env var to full path)
- An `installer.iss` script in the project root (Inno Setup config)

## Workflow

### Step 1 — Build EXE with PyInstaller

```bat
REM Clean previous build
rmdir /s /q "dist\<AppName>"

REM Basic one-dir windowed build
pyinstaller ^
    --noconfirm --clean ^
    --name "<AppName>" ^
    --onedir --windowed ^
    --paths "src" ^
    "<entry_point.py>"
```

Common additions:

| Flag | Purpose |
|------|---------|
| `--add-data "config;config"` | Bundle a data folder (src;dest, Windows separator `;`) |
| `--collect-all <pkg>` | Pull in all files for a package (needed for GUI libs, cv2, etc.) |
| `--hidden-import <mod>` | Force-include modules PyInstaller misses |
| `--exclude-module <mod>` | Strip heavy unused packages (torch, onnxruntime, etc.) |
| `--contents-directory .` | Flatten resources into dist root instead of `_internal\` |
| `--onefile` | Single EXE alternative (slower startup, simpler distribution) |

#### uv projects

```bat
uv run --extra dev pyinstaller <flags> entry.py
```

#### pip / venv projects

```bat
.venv\Scripts\pyinstaller <flags> entry.py
```

### Step 2 — Package installer with Inno Setup

```bat
iscc installer.iss
```

Output: path defined in `installer.iss`, typically `installer\<AppName>_Setup.exe`.

## Outputs

| Path | Description |
|------|-------------|
| `dist\<AppName>\<AppName>.exe` | Standalone EXE (one-dir) |
| `dist\<AppName>.exe` | Standalone EXE (one-file) |
| `installer\<AppName>_Setup.exe` | Windows installer (Inno Setup) |

## Always rebuild clean

Delete `dist\<AppName>\` before each build to avoid stale files leaking into the package:

```bat
if exist "dist\%NAME%" rmdir /s /q "dist\%NAME%"
```

## Common failures

| Error | Fix |
|-------|-----|
| `ModuleNotFoundError` at runtime | Add `--hidden-import <module>` |
| DLL / `.pyd` missing at runtime | Add `--collect-all <package>` |
| `iscc` not found | Install Inno Setup 6.x; verify with `where iscc` |
| Build succeeds but EXE crashes silently | Drop `--windowed` temporarily to see console errors |
| `dist` too large | Add `--exclude-module` for unused heavy deps (torch, scipy, etc.) |
| Resources not found at runtime | Check `--add-data` paths; use `sys._MEIPASS` in code for one-file mode |

## `installer.iss` essentials

Minimal Inno Setup script structure:

```iss
[Setup]
AppName=MyApp
AppVersion=1.0
DefaultDirName={autopf}\MyApp
OutputDir=installer
OutputBaseFilename=MyApp_Setup

[Files]
Source: "dist\MyApp\*"; DestDir: "{app}"; Flags: recursesubdirs

[Icons]
Name: "{autoprograms}\MyApp"; Filename: "{app}\MyApp.exe"
```

## `build_package.bat` template

Drop this in the project root and edit the three variables at the top:

```bat
@echo off
setlocal
cd /d "%~dp0"

set NAME=MyApp
set ENTRY=src\main.py
set ISCC=iscc

if exist "dist\%NAME%" rmdir /s /q "dist\%NAME%"

echo [1/2] Building EXE ...
uv run --extra dev pyinstaller ^
    --noconfirm --clean ^
    --name "%NAME%" ^
    --onedir --windowed ^
    --paths "src" ^
    "%ENTRY%"
if errorlevel 1 ( echo [ERROR] PyInstaller failed. & pause & exit /b 1 )

echo [2/2] Packaging installer ...
%ISCC% installer.iss
if errorlevel 1 ( echo [ERROR] Inno Setup failed. & pause & exit /b 1 )

echo [OK] installer\%NAME%_Setup.exe
pause
endlocal
```
