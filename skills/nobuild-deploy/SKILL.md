---
name: nobuild-deploy
description: "Audit a Python/uv project's deployment readiness — check lib/ is fully operational offline, all deps installed, download uv.exe, and generate/fix prepare.bat/install.bat/run.bat/uninstall.bat for Windows service deployment. Use when user says: nobuild, deploy check, check deployment, prepare for prod, build .bat files, check prepare.bat, fix install.bat, prod deploy check, download uv."
---

# /nobuild_deploy

Audit a `uv`-managed Python project for offline Windows deployment. No venv on
prod — bundles Python interpreter + flat `lib/` directory. Nothing to break, no
symlinks, no runtime dependency resolution.

## Architecture

```
project/
├── python/<PYTHON_VER>/    # Standalone CPython (from uv)
├── lib/                    # All packages installed via --target
├── src/                    # Application source
├── nssm.exe                # Windows service wrapper
├── uv.exe                  # Only used at prepare time (dev machine)
├── prepare.bat             # Dev: bundle everything into a zip
├── install.bat             # Prod: install as Windows service
├── uninstall.bat           # Prod: stop + remove service
├── run.bat                 # Prod: entry point (called by nssm)
├── .env                    # Runtime config
└── pyproject.toml          # Dependency source of truth
```

**Key principle:** `uv` is a dev/build tool only. Production uses raw
`python.exe` + `PYTHONPATH`. No venv, no pip, no uv at runtime. No symlinks
to stale interpreters. No `.venv` metadata to invalidate. No network calls.

## Usage

```
/nobuild_deploy                  # full audit of current dir
/nobuild_deploy --fix            # audit + regenerate broken .bat files + download uv
/nobuild_deploy --bat-only       # just generate the 4 .bat files, skip checks
/nobuild_deploy --check-only     # just audit, don't write anything
/nobuild_deploy --download-uv    # download uv.exe only
```

## What It Checks

1. **pyproject.toml** — exists, valid, has dependencies
2. **uv.exe** — present in project dir (downloads if missing)
3. **python/** — bundled interpreter exists
4. **lib/** — exists, has all deps importable via PYTHONPATH
5. **Offline readiness** — raw `python.exe` with `PYTHONPATH=lib/` imports all deps
6. **uv.lock** — exists, includes all deps from pyproject.toml
7. **prepare.bat** — correct export + `pip install --target lib/` + 7z pipeline
8. **install.bat** — correct service setup with admin elevation and logging
9. **run.bat** — uses bundled python.exe + PYTHONPATH, no uv at runtime
10. **uninstall.bat** — stop + remove service with admin elevation

## Implementation

### Step 1 — Find project root

Look for `pyproject.toml` in the current directory. Fail if not found.

### Step 2 — Check/download uv.exe

If `uv.exe` is not in the project directory, download it from GitHub releases.

### Step 3 — Check bundled Python

```bash
# Detect PYTHON_VER from .python-version or pyproject.toml requires-python
PYTHON_VER_DIR=$(ls python/ 2>/dev/null | head -1)
PYTHON_EXE="python/$PYTHON_VER_DIR/python.exe"

if [ -f "$PYTHON_EXE" ]; then
  echo "OK: Bundled Python found: $PYTHON_EXE"
else
  echo "FAIL: No bundled Python in python/"
  echo "      Run prepare.bat to copy from uv cache"
fi
```

### Step 4 — Check lib/ directory

```bash
if [ -d "lib" ]; then
  PKG_COUNT=$(ls lib/ | wc -l)
  echo "OK: lib/ exists ($PKG_COUNT items)"
else
  echo "FAIL: lib/ not found — run prepare.bat"
fi
```

### Step 5 — Verify all deps importable via PYTHONPATH

```bash
MISSING=""
for dep in $DEPS; do
  IMPORT=$(map_dep_to_import "$dep")
  if PYTHONPATH=lib "$PYTHON_EXE" -c "__import__('$IMPORT')" 2>/dev/null; then
    echo "  OK  $dep"
  else
    echo "  MISS  $dep"
    MISSING="$MISSING $dep"
  fi
done
```

Import name mapping:
```
opencv-python|opencv-contrib-python → cv2
python-dotenv → dotenv
python-json-logger → pythonjsonlogger
pillow|Pillow → PIL
pyinstaller → PyInstaller
scikit-image → skimage
psycopg2 → psycopg2
python-docx → docx
beautifulsoup4 → bs4
pyyaml → yaml
```

### Step 6 — Check uv.lock freshness

Verify all pyproject.toml deps appear in uv.lock.

### Step 7 — Audit prepare.bat

Check for:
- ✅ Copies Python from uv cache to `python/`
- ✅ Exports requirements: `uv export --no-hashes --frozen --no-header --no-annotate`
- ✅ Installs to lib/: `uv pip install --python ... --target lib/ -r _requirements.txt`
- ✅ Cleans lib/ before install (`rmdir /S /Q lib`)
- ✅ Bundles MSVC runtime DLLs (msvcp140.dll, vcruntime140.dll, vcruntime140_1.dll)
- ✅ Deletes old zip before creating new one (prevents stale files persisting)
- ✅ Error handling with `if %errorLevel% neq 0` after each critical step
- ✅ 7z includes `lib` and `python/`, excludes `.venv`, `.uv_cache`, `uv.exe`
- ❌ ISSUE if excludes `*.dist-info` from 7z (breaks package discovery in lib/)
- ❌ ISSUE if still uses `.venv` in the zip (old pattern, not needed)
- ❌ ISSUE if 7z updates existing archive instead of creating fresh (stale files persist)
- ❌ ISSUE if `copy` targets end with `\` (creates `nul` file on Windows instead of suppressing output)

### Step 8 — Audit run.bat

Check for:
- ✅ Sets `PYTHONPATH=%~dp0lib`
- ✅ Sets `PATH=%~dp0lib\<native_pkg>\libs;%PATH%` for packages with native DLLs
- ✅ Calls bundled `python.exe` directly (no uv at runtime)
- ✅ Uses `-Xutf8` flag
- ✅ Pre-flight checks: validates python/ and lib/ exist before launching
- ✅ No `pause` (hangs nssm service if Python exits)
- ❌ ISSUE if uses `uv run` (old pattern — uv not needed on prod)
- ❌ ISSUE if references `.venv` (old pattern — no venv on prod)
- ❌ ISSUE if has `pause` at end (blocks nssm from detecting crashes)
- ❌ ISSUE if native DLL directories not on PATH (DLL load failures)

### Step 9 — Audit install.bat / uninstall.bat

Check for:
- ✅ Admin elevation check
- ✅ Creates `logs/` directory before nssm binds stdout/stderr
- ✅ Smoke test (import critical native modules) before starting service
- ✅ Actionable error messages on failure
- ❌ ISSUE if no `mkdir logs` before nssm log binding (silent log loss)
- ❌ ISSUE if no smoke test (DLL failures only discovered after service starts)

### Step 10 — Summary

Report status and next steps.

---

## Generated File Templates

### prepare.bat

```bat
@echo off
set PYTHON_VER=cpython-3.12.13-windows-x86_64-none
set SRC=%APPDATA%\uv\python\%PYTHON_VER%
set DST=%~dp0python\%PYTHON_VER%

:: Bundle Python interpreter
if exist "%DST%" (
    echo [OK] Python already bundled, skipping.
) else (
    echo [..] Copying %PYTHON_VER% from uv cache...
    xcopy /E /I /Q "%SRC%" "%DST%"
    if %errorLevel% neq 0 (
        echo [FAIL] Could not copy Python from uv cache.
        pause & exit /b 1
    )
    echo [OK] Done.
)

:: Export locked requirements and install to flat lib/
echo [..] Exporting locked requirements...
"%~dp0uv.exe" export --no-hashes --frozen --no-header --no-annotate > "%~dp0_requirements.txt"
if %errorLevel% neq 0 (
    echo [FAIL] uv export failed. Is uv.lock present?
    pause & exit /b 1
)

echo [..] Installing packages to lib/ ...
if exist "%~dp0lib" rmdir /S /Q "%~dp0lib"
"%~dp0uv.exe" pip install --python "%DST%\python.exe" --target "%~dp0lib" -r "%~dp0_requirements.txt"
if %errorLevel% neq 0 (
    echo [FAIL] pip install --target failed.
    pause & exit /b 1
)
del "%~dp0_requirements.txt"
echo [OK] lib/ ready.

:: Bundle MSVC runtime DLLs (prod machines may not have VC++ Redistributable)
:: Use explicit target filenames — never end with \ (creates a 'nul' file on Windows)
echo [..] Bundling MSVC runtime...
copy /Y "%SystemRoot%\System32\msvcp140.dll" "%~dp0lib\<NATIVE_PKG>\libs\msvcp140.dll" >nul 2>&1
copy /Y "%SystemRoot%\System32\vcruntime140.dll" "%~dp0lib\<NATIVE_PKG>\libs\vcruntime140.dll" >nul 2>&1
copy /Y "%SystemRoot%\System32\vcruntime140_1.dll" "%~dp0lib\<NATIVE_PKG>\libs\vcruntime140_1.dll" >nul 2>&1
copy /Y "%SystemRoot%\System32\vcomp140.dll" "%~dp0lib\<NATIVE_PKG>\libs\vcomp140.dll" >nul 2>&1
echo [OK] MSVC runtime bundled.

:: Bundle Universal CRT forwarder DLLs (needed on older Windows builds)
echo [..] Bundling UCRT forwarders...
for %%f in ("%SystemRoot%\System32\downlevel\api-ms-win-crt-*.dll") do copy /Y "%%f" "%~dp0lib\<NATIVE_PKG>\libs\%%~nxf" >nul 2>&1
echo [OK] UCRT forwarders bundled.

:: Co-locate native pkg DLLs alongside .pyd so Windows finds them without PATH tricks
echo [..] Co-locating DLLs...
copy /Y "%~dp0lib\<NATIVE_PKG>\libs\*.dll" "%~dp0lib\<NATIVE_PKG>\base\" >nul 2>&1
echo [OK] DLLs co-located.

:: Archive (always delete old zip first — 7z update mode keeps stale files)
echo [..] Zipping...
pushd "%~dp0"
if exist "..\<PROJECT_NAME>.7z" del "..\<PROJECT_NAME>.7z"
7z a -mx=1 "..\<PROJECT_NAME>.7z" * lib .python-version ^
    -xr!"__pycache__" -xr!".git" -xr!"*.pyc" ^
    -x!"predicted_data" -x!"data" ^
    -xr!".venv" -xr!".uv_cache" -xr!"uv.exe" -x!"tests"
if %errorLevel% neq 0 (
    echo [FAIL] 7z packaging failed.
    popd & pause & exit /b 1
)
popd
echo [OK] <PROJECT_NAME>.7z ready.

:: Quick import validation directly from lib/
echo [..] Validating imports...
set PYTHONPATH=%~dp0lib
set PATH=%~dp0lib\<NATIVE_PKG>\libs;%PATH%
"%DST%\python.exe" -Xutf8 -c "import <critical_modules>; print('[OK] All imports passed')"
if %errorLevel% neq 0 (
    echo [FAIL] Import validation failed. Fix before deploying.
    pause & exit /b 1
)
echo [OK] Validation passed. Safe to deploy.
pause
```

### run.bat

```bat
@echo off
set ENV=prod
set PYTHONPATH=%~dp0lib
set PATH=%~dp0lib\<NATIVE_PKG>\libs;%PATH%

:: Pre-flight checks
if not exist "%~dp0python\<PYTHON_VER>\python.exe" (
    echo [FATAL] python/ not found. Re-extract the deployment zip. >&2
    exit /b 1
)
if not exist "%~dp0lib\<CANARY_PKG>" (
    echo [FATAL] lib/ missing or incomplete. Re-extract the deployment zip. >&2
    exit /b 1
)

"%~dp0python\<PYTHON_VER>\python.exe" -Xutf8 ./src/main.py
```

> **No `pause`** — nssm manages the process lifecycle. `pause` would hang the
> service after a crash, making it appear alive while doing nothing.

### install.bat

```bat
@echo off
set SERVICE_NAME=<ServiceName>
set PROJECT_DIR=%~dp0
set NSSM_EXE="%PROJECT_DIR%nssm.exe"
set LAUNCHER_BAT="%PROJECT_DIR%run.bat"

net session >nul 2>&1
if %errorLevel% neq 0 (
    echo [!] Requesting Administrative Privileges...
    powershell -Command "Start-Process -FilePath '%0' -Verb RunAs"
    exit /b
)

%NSSM_EXE% install %SERVICE_NAME% %LAUNCHER_BAT%
%NSSM_EXE% set %SERVICE_NAME% AppDirectory %PROJECT_DIR%

:: Ensure logs directory exists before binding stdout/stderr
if not exist "%PROJECT_DIR%logs" mkdir "%PROJECT_DIR%logs"
%NSSM_EXE% set %SERVICE_NAME% AppStdout "%PROJECT_DIR%logs\service_out.log"
%NSSM_EXE% set %SERVICE_NAME% AppStderr "%PROJECT_DIR%logs\service_err.log"
echo [OK] Service installed.

:: Smoke test before starting — catches missing DLLs/modules at install time
echo [..] Running import smoke test...
cd /d "%PROJECT_DIR%"
set PYTHONPATH=%PROJECT_DIR%lib
set PATH=%PROJECT_DIR%lib\<NATIVE_PKG>\libs;%PATH%
"%PROJECT_DIR%python\<PYTHON_VER>\python.exe" -Xutf8 -c "import <critical_modules>; print('[OK] All critical imports passed')"
if %errorLevel% neq 0 (
    echo [FAIL] Smoke test failed. Check missing DLLs or incomplete lib/.
    echo        Try installing Visual C++ Redistributable 2015-2022.
    pause
    exit /b 1
)

%NSSM_EXE% start %SERVICE_NAME%
```

### uninstall.bat

```bat
@echo off
set SERVICE_NAME=<ServiceName>
set PROJECT_DIR=%~dp0
set NSSM_EXE="%PROJECT_DIR%nssm.exe"

net session >nul 2>&1
if %errorLevel% neq 0 (
    echo [!] Requesting Administrative Privileges...
    powershell -Command "Start-Process -FilePath '%0' -Verb RunAs"
    exit /b
)

%NSSM_EXE% stop %SERVICE_NAME%
%NSSM_EXE% remove %SERVICE_NAME% confirm
echo [OK] Service removed.
pause
```

---

## Deployment Workflow

```
Dev machine (internet)         Prod machine (offline)
─────────────────────          ─────────────────────
1. git pull                    4. Extract .7z
2. Run prepare.bat             5. Copy .env.template → .env
3. Transfer .7z via VNC/USB    6. Run install.bat (admin)
                               Done — service is running
```

---

## What Goes in the Zip

| Include | Exclude |
|---------|---------|
| `python/` (interpreter) | `.venv/` |
| `lib/` (all packages + MSVC DLLs) | `.uv_cache/` |
| `src/` (app code) | `.git/` |
| `nssm.exe` | `uv.exe` (dev-only, ~40MB) |
| `run.bat`, `install.bat`, `uninstall.bat` | `__pycache__/`, `*.pyc` |
| `.env.template`, `pyproject.toml` | `tests/` |

---

## .gitignore additions

```
python/
lib/
.venv
.uv_cache
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `No module named X` | `lib/` missing or incomplete | Re-run `prepare.bat`, redeploy |
| `DLL load failed` | Missing MSVC runtime on prod | Bundle msvcp140/vcruntime140 DLLs in prepare.bat |
| `DLL load failed` (native pkg) | DLL directory not on PATH | Add `set PATH=%~dp0lib\<pkg>\libs;%PATH%` to run.bat |
| Service appears running but does nothing | `pause` in run.bat blocking after crash | Remove `pause` from run.bat |
| Service won't start | Wrong path in nssm | `nssm edit <ServiceName>` |
| Stale files in zip after rebuild | 7z update mode keeps old files | Delete old .7z before `7z a` |
| `-xr!"data"` strips `lib/skimage/data/` | `-xr!` is recursive, hits package subfolders | Use `-x!"data"` for top-level project folders |
| `*.pyi` exclusion breaks scikit-image | skimage uses `.pyi` stubs at runtime via lazy_loader | Never exclude `*.pyi` from zip |
| CWD-relative paths fail as service | `'../..'` depends on CWD; nssm sets CWD to AppDirectory | Use `os.path.dirname(os.path.abspath(__file__))` |
| `nul` file created in lib/ | `copy /Y ... "path\"` with trailing backslash | Use explicit target filename |
| Import errors after upgrade | Stale `lib/` from old deploy | `prepare.bat` always does clean `rmdir /S /Q lib` |
| Python not found | `python/` not in zip | Check `prepare.bat` xcopy step |
| No logs written on crash | `logs/` dir doesn't exist | Add `mkdir logs` in install.bat before nssm binding |
| Smoke test fails after elevation | CWD changed to System32 | Add `cd /d "%PROJECT_DIR%"` before smoke test |
| `.venv` keeps breaking | You're on the old pattern | Switch to lib/ pattern (this skill) |

---

## Why Not .venv?

The old pattern (`uv run --no-sync` with a bundled `.venv`) breaks when:
- The interpreter symlink points to a non-existent path (different machine, Python upgrade)
- uv nukes the stale venv and recreates it empty
- `--no-sync` means nothing gets reinstalled
- `--offline` requires a populated cache that may not exist

The `lib/` pattern has zero moving parts: it's just files on disk. Python reads
them via `PYTHONPATH`. Nothing to resolve, nothing to invalidate, works forever.

---

## Known Pitfalls (Hard-Won)

1. **MSVC runtime not on prod machines.** Offline Windows machines often lack VC++ Redistributable. Bundle `msvcp140.dll`, `vcruntime140.dll`, `vcruntime140_1.dll` from System32 into the native package's libs/ directory.

2. **`copy /Y "src" "dir\"` creates a `nul` file.** On Windows, `>nul` after a `copy` whose target ends with `\` can create a literal file named `nul`. Always use explicit target filenames: `copy /Y "src" "dir\filename.dll"`.

3. **7z update mode keeps deleted files.** If you re-run `7z a` on an existing archive, files removed from disk persist in the archive. Always `del` the old .7z before creating.

4. **`pause` hangs nssm services.** If the Python process crashes, `pause` blocks waiting for a keypress that will never come. nssm sees the cmd.exe as alive. Never use `pause` in service entry scripts.

5. **Admin elevation changes CWD to System32.** After `Start-Process -Verb RunAs`, the working directory becomes `C:\Windows\System32`. Always `cd /d "%~dp0"` or use absolute paths after elevation.

6. **Native packages need DLL directories on PATH.** Packages like PaddlePaddle put .pyd files in one directory and their dependency DLLs in a sibling `libs/` directory. Python's DLL loader doesn't find them without `set PATH=...;%PATH%`.

7. **`*.dist-info` exclusion breaks metadata.** Many packages use `importlib.metadata` for version discovery or plugin loading. Don't exclude dist-info from the zip.

9. **`-xr!"data"`** vs **`-x!"data"`** — `-xr!` excludes recursively (hits `lib/skimage/data/`, `lib/paddle/dataset/`, etc.). Use `-x!` for top-level project folders you want to exclude without affecting packages inside `lib/`.

10. **CWD-relative paths in application code break as a service.** If app code uses relative paths like `'../..'` to find credentials or config, those paths depend on CWD. When nssm runs the service, CWD is `AppDirectory` (project root). Fix: resolve paths relative to `__file__` using `os.path.dirname(os.path.abspath(__file__))` — never relative to CWD.
