---
name: nobuild-deploy
description: "Audit a Python/uv project's deployment readiness — check .venv is fully operational offline, all deps installed, download uv.exe, and generate/fix prepare.bat/install.bat/run.bat for Windows service deployment. Use when user says: nobuild, deploy check, check deployment, prepare for prod, build .bat files, check prepare.bat, fix install.bat, prod deploy check, download uv."
---

# /nobuild_deploy

Audit a `uv`-managed Python project for Windows deployment. Checks the .venv, tests offline readiness, and generates/repairs `prepare.bat`, `install.bat`, `run.bat`.

## Usage

```
/nobuild_deploy                                    # full audit of current dir
/nobuild_deploy --fix                               # audit + regenerate broken .bat files + download uv
/nobuild_deploy --bat-only                          # just generate the 3 .bat files, skip checks
/nobuild_deploy --check-only                        # just audit, don't write anything
/nobuild_deploy --download-uv                       # download uv.exe only
```

## What It Checks

1. **pyproject.toml** — exists, valid, has dependencies
2. **uv.exe** — present in project dir (downloads if missing)
3. **.venv** — exists, correct Python version, has all deps
4. **Offline readiness** — `uv run --no-sync` imports work for all dependencies
5. **uv.lock** — exists, includes all deps from pyproject.toml
6. **prepare.bat** — correct uv sync + 7z pipeline, no `-xr!"*.dist-info"` issue
7. **install.bat** — correct service setup with admin elevation and logging
8. **run.bat** — correct env vars, `uv run --no-sync`, no path typos
9. **Known pitfalls** — empty `.venv/.lock`, missing entry script

## Implementation

### Step 1 — Find project root

Look for `pyproject.toml` in the current directory. If not found, search parent dirs. Fail with "No pyproject.toml found — nobuild_deploy requires a uv-managed Python project."

```bash
if [ ! -f "pyproject.toml" ]; then
  echo "FAIL: No pyproject.toml found in $(pwd)"
  exit 1
fi
echo "OK: Found pyproject.toml in $(pwd)"
```

### Step 2 — Check/download uv.exe

If `uv.exe` is not in the project directory, download it from the official GitHub release.

```bash
echo ""
echo "=== Checking uv.exe ==="
UV_BIN="./uv.exe"
if [ -f "$UV_BIN" ]; then
  UV_VER=$("$UV_BIN" --version 2>&1 | head -1)
  echo "OK: uv.exe found — $UV_VER"
else
  echo "NOT FOUND: uv.exe missing from project root"
  if [ "$1" = "--fix" ] || [ "$1" = "--download-uv" ] || [ "$1" = "--bat-only" ]; then
    echo "Downloading uv.exe from GitHub..."
    UV_URL="https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-pc-windows-msvc.zip"
    curl -sL -o uv.zip "$UV_URL"
    if [ -f "uv.zip" ]; then
      unzip -o uv.zip -d /tmp/uv_extract 2>/dev/null
      cp /tmp/uv_extract/uv.exe ./
      rm -rf uv.zip /tmp/uv_extract
      echo "OK: uv.exe downloaded"
    else
      echo "WARN: Download failed — try: winget install --id=astral-sh.uv"
      echo "      or: https://github.com/astral-sh/uv/releases"
    fi
  else
    echo "NOTE: Run with --fix or --download-uv to auto-download"
    echo "      Manual: winget install --id=astral-sh.uv"
    echo "              or download from https://github.com/astral-sh/uv/releases"
  fi
fi
```

### Step 3 — Parse dependencies from pyproject.toml

```bash
DEPS=$(python3 -c "
import re
with open('pyproject.toml') as f:
    content = f.read()
m = re.search(r'dependencies\s*=\s*\[(.*?)\]', content, re.DOTALL)
if m:
    deps = re.findall(r'\"([^\"=<>~!@]+)', m.group(1))
    for d in sorted(deps):
        print(d.strip())
" 2>/dev/null)

if [ -z "$DEPS" ]; then
  echo "FAIL: Could not parse dependencies from pyproject.toml"
  exit 1
fi

COUNT=$(echo "$DEPS" | wc -l)
echo "OK: Found $COUNT dependencies in pyproject.toml"
```

### Step 4 — Check .venv exists and has correct Python

```bash
UV_PYTHON=".venv/Scripts/python.exe"
if [ ! -f "$UV_PYTHON" ]; then
  echo "FAIL: .venv not found or missing python.exe"
  echo "       Run: uv sync"
  exit 1
fi

PY_VER=$("$UV_PYTHON" --version 2>&1)
echo "OK: $PY_VER"

REQ_VER=$(python3 -c "
import re
with open('pyproject.toml') as f:
    content = f.read()
m = re.search(r'requires-python\s*=\s*\"(>=.*?)\"', content)
print(m.group(1) if m else 'unknown')
" 2>/dev/null)
echo "    Requires: Python $REQ_VER"
```

### Step 5 — Verify all deps are installed in .venv

```bash
echo ""
echo "=== Checking installed packages ==="
MISSING=""
for dep in $DEPS; do
  case "$dep" in
    opencv-python|opencv-contrib-python) IMPORT="cv2" ;;
    python-dotenv) IMPORT="dotenv" ;;
    python-json-logger) IMPORT="pythonjsonlogger" ;;
    pillow|Pillow) IMPORT="PIL" ;;
    pyinstaller) IMPORT="PyInstaller" ;;
    scikit-image) IMPORT="skimage" ;;
    psycopg2) IMPORT="psycopg2" ;;
    python-docx) IMPORT="docx" ;;
    *) IMPORT="$dep" ;;
  esac
  if "$UV_PYTHON" -c "__import__('$IMPORT')" 2>/dev/null; then
    echo "  OK  $dep"
  else
    echo "  MISS  $dep"
    MISSING="$MISSING $dep"
  fi
done

if [ -n "$MISSING" ]; then
  echo ""
  echo "WARN: Missing packages:$MISSING"
  echo "      Run: uv sync"
else
  echo ""
  echo "OK: All dependencies installed"
fi
```

### Step 6 — Test offline readiness

```bash
echo ""
echo "=== Testing offline readiness (uv run --no-sync) ==="

UV_BIN="./uv.exe"
[ ! -f "$UV_BIN" ] && UV_BIN="uv"

# Build import test from parsed deps
IMPORT_TEST=$(python3 -c "
import_map = {
    'opencv-python': 'cv2', 'opencv-contrib-python': 'cv2',
    'python-dotenv': 'dotenv', 'python-json-logger': 'pythonjsonlogger',
    'pillow': 'PIL', 'Pillow': 'PIL',
    'pyinstaller': 'PyInstaller', 'scikit-image': 'skimage',
    'psycopg2': 'psycopg2', 'python-docx': 'docx',
}
deps = '''$DEPS'''
for d in deps.strip().split('\n'):
    d = d.strip()
    if not d: continue
    imp = import_map.get(d, d)
    print(f'    try: __import__(\"{imp}\")\n    except: failed.append(\"{d}\")')
")

"$UV_BIN" run --no-sync --python "$UV_PYTHON" python3 -c "
import sys
sys.path.insert(0, 'src')
failed = []
$IMPORT_TEST
if failed:
    for p in failed:
        print(f'  FAIL {p}')
    sys.exit(1)
else:
    print('All dependencies import OK')
" 2>&1

if [ $? -ne 0 ]; then
  OFFLINE_STATUS="not ready"
  echo "FAIL: Offline imports failed"
  echo "       The .venv is not self-contained. Run: uv sync"
  exit 1
fi
OFFLINE_STATUS="ready"
echo "OK: uv run --no-sync works offline"
```

### Step 7 — Check uv.lock freshness

```bash
echo ""
echo "=== Checking uv.lock ==="
if [ ! -f "uv.lock" ]; then
  echo "FAIL: uv.lock not found"
  echo "       Run: uv lock"
  exit 1
fi

MISSING_LOCK=""
for dep in $DEPS; do
  if grep -qi "name = \"$dep\"" uv.lock; then
    :
  else
    MISSING_LOCK="$MISSING_LOCK $dep"
  fi
done

if [ -n "$MISSING_LOCK" ]; then
  echo "WARN: Missing from uv.lock:$MISSING_LOCK"
  echo "       Run: uv lock"
else
  echo "OK: uv.lock contains all dependencies"
fi

if [ -f ".venv/.lock" ] && [ ! -s ".venv/.lock" ]; then
  echo "NOTE: .venv/.lock is empty — normal for uv venvs, not a problem"
fi
```

### Step 8 — Audit prepare.bat

```bash
echo ""
echo "=== Auditing prepare.bat ==="
HAS_PREPARE=false
ISSUES=0
FIXES=""

if [ -f "prepare.bat" ]; then
  HAS_PREPARE=true
  echo "Found prepare.bat"

  if grep -qi "dist-info" prepare.bat; then
    echo "  ISSUE: 7z command excludes *.dist-info — uv thinks packages are missing"
    echo "         Fix: remove -xr!\"*.dist-info\" from the 7z command"
    ISSUES=$((ISSUES+1))
    FIXES="$FIXES remove_dist-info"
  fi

  if grep -qi "uv sync" prepare.bat; then
    echo "  OK: uv sync present"
  else
    echo "  ISSUE: uv sync missing — .venv won't be populated"
    ISSUES=$((ISSUES+1))
  fi

  if grep -qi "7z" prepare.bat; then
    echo "  OK: 7z packaging present"
  else
    echo "  NOTE: No 7z packaging — zip step may be missing"
  fi
else
  echo "  MISSING: prepare.bat not found"
  ISSUES=$((ISSUES+1))
  FIXES="$FIXES create_prepare"
fi
```

### Step 9 — Audit install.bat

```bash
echo ""
echo "=== Auditing install.bat ==="
HAS_INSTALL=false

if [ -f "install.bat" ]; then
  HAS_INSTALL=true
  echo "Found install.bat"

  if grep -qi "admin\|RunAs\|elevat" install.bat; then
    echo "  OK: Admin elevation check present"
  else
    echo "  NOTE: No admin elevation check — service install usually needs admin"
  fi

  if grep -qi "stdout\|stderr\|log" install.bat; then
    echo "  OK: Logging/output configured"
  else
    echo "  NOTE: No logging/output configured"
  fi
else
  echo "  MISSING: install.bat not found"
  ISSUES=$((ISSUES+1))
  FIXES="$FIXES create_install"
fi
```

### Step 10 — Audit run.bat

```bash
echo ""
echo "=== Auditing run.bat ==="
HAS_RUN=false

if [ -f "run.bat" ]; then
  HAS_RUN=true
  echo "Found run.bat"

  if grep -qi "no-sync" run.bat; then
    echo "  OK: Uses uv run --no-sync (offline-safe)"
  else
    echo "  ISSUE: Missing --no-sync — uv may try to download on prod without internet"
    ISSUES=$((ISSUES+1))
  fi

  # Detect what entry script it runs
  ENTRY=$(grep -oP '\.\/\S+\.py' run.bat 2>/dev/null || grep -oP '\S+\.py' run.bat 2>/dev/null || echo "unknown")
  if [ "$ENTRY" != "unknown" ] && [ -n "$ENTRY" ]; then
    echo "  OK: Entry script: $ENTRY"
  else
    echo "  NOTE: No .py entry script detected in run.bat"
  fi

  if grep -qi "UV_PYTHON_INSTALL_DIR\|UV_PYTHON" run.bat; then
    echo "  OK: Python path configured"
  else
    echo "  NOTE: No UV_PYTHON_INSTALL_DIR set — uv uses default Python discovery"
  fi
else
  echo "  MISSING: run.bat not found"
  ISSUES=$((ISSUES+1))
  FIXES="$FIXES create_run"
fi
```

### Step 11 — Summary

```bash
echo ""
echo "=========================================="
echo "  nobuild_deploy Audit Summary"
echo "=========================================="
if [ $ISSUES -eq 0 ] && [ -z "$MISSING" ]; then
  echo "  STATUS: ✅ Ready for deployment"
elif [ $ISSUES -eq 0 ] && [ -n "$MISSING" ]; then
  echo "  STATUS: ⚠️  All .bat files OK, but missing packages"
else
  echo "  STATUS: ❌ $ISSUES issues found"
fi
echo ""
echo "  Config files:"
echo "    prepare.bat: $([ "$HAS_PREPARE" = true ] && echo 'exists' || echo 'missing')"
echo "    install.bat: $([ "$HAS_INSTALL" = true ] && echo 'exists' || echo 'missing')"
echo "    run.bat:     $([ "$HAS_RUN" = true ] && echo 'exists' || echo 'missing')"
echo "  Dependencies: $COUNT in pyproject.toml, $([ -z "$MISSING" ] && echo 'all installed' || echo "$MISSING missing")"
echo "  Offline:      $OFFLINE_STATUS"
echo "=========================================="
```

### Step 12 — Generate/Fix .bat files (if --fix or --bat-only)

If the user passed `--fix` or `--bat-only`, or if any of the three .bat files are missing, generate them. All three files use the project name from `pyproject.toml` or default to `MyApp`.

```bash
# Get project name
PROJECT_NAME=$(python3 -c "
import re
with open('pyproject.toml') as f:
    content = f.read()
m = re.search(r'name\s*=\s*\"(.+?)\"', content)
print(m.group(1) if m else 'MyApp')
" 2>/dev/null)
[ -z "$PROJECT_NAME" ] && PROJECT_NAME="MyApp"
```

#### prepare.bat

```bash
if [ ! -f "prepare.bat" ] || [ "$1" = "--fix" ] || [ "$1" = "--bat-only" ]; then
  echo ""
  echo "=== Generating prepare.bat ==="
  cat > prepare.bat << BATEOF
@echo off
setlocal enabledelayedexpansion

:: Bundle Python from uv cache (whole cache, automatically finds the right version)
set SRC=%APPDATA%\uv\python
set DST=%~dp0python
if exist "%DST%" (
    echo [OK] Python already bundled, skipping.
) else (
    echo [..] Copying Python from uv cache...
    xcopy /E /I /Q "%SRC%" "%DST%"
    echo [OK] Done.
)

echo [..] Syncing dependencies into .venv...
"%~dp0uv.exe" sync
if %errorLevel% neq 0 (
    echo [FAIL] uv sync failed
    pause
    exit /b 1
)
echo [OK] .venv ready.

echo [..] Zipping...
pushd "%~dp0"
7z a -mx=1 "..\$PROJECT_NAME.7z" * .venv .python-version -xr!"__pycache__" -xr!".git" -xr!"*.pyc" -xr!"tests"
popd
if %errorLevel% equ 0 (
    echo [OK] $PROJECT_NAME.7z ready.
) else (
    echo [FAIL] 7z packaging failed
)
pause
BATEOF
  echo "  OK: prepare.bat written (project: $PROJECT_NAME)"
fi
```

#### install.bat

```bash
if [ ! -f "install.bat" ] || [ "$1" = "--fix" ] || [ "$1" = "--bat-only" ]; then
  echo ""
  echo "=== Generating install.bat ==="
  cat > install.bat << BATEOF
@echo off
set SERVICE_NAME=${PROJECT_NAME}
set PROJECT_DIR=%~dp0

:: Check for Admin rights
net session >nul 2>&1
if %errorLevel% neq 0 (
    echo [!] Requesting Administrative Privileges...
    powershell -Command "Start-Process -FilePath '%0' -Verb RunAs"
    exit /b
)

:: Edit this section to match your service manager (NSSM, winsw, sc, etc.)
:: Example using NSSM:
:: set NSSM_EXE="%PROJECT_DIR%nssm.exe"
:: set LAUNCHER_BAT="%PROJECT_DIR%run.bat"
::
:: %%NSSM_EXE%% install %%SERVICE_NAME%% %%LAUNCHER_BAT%%
:: %%NSSM_EXE%% set %%SERVICE_NAME%% AppDirectory %%PROJECT_DIR%%
:: %%NSSM_EXE%% set %%SERVICE_NAME%% AppStdout "%%PROJECT_DIR%%logs\service_out.log"
:: %%NSSM_EXE%% set %%SERVICE_NAME%% AppStderr "%%PROJECT_DIR%%logs\service_err.log"
:: echo [OK] Service installed.
:: %%NSSM_EXE%% start %%SERVICE_NAME%%

echo [!!] No service manager configured. Edit install.bat with your tool.
pause
BATEOF
  echo "  OK: install.bat written (template with placeholders)"
fi
```

#### run.bat

```bash
if [ ! -f "run.bat" ] || [ "$1" = "--fix" ] || [ "$1" = "--bat-only" ]; then
  echo ""
  echo "=== Generating run.bat ==="

  # Find the main entry script
  ENTRY_SCRIPT="src/main.py"
  [ ! -f "$ENTRY_SCRIPT" ] && ENTRY_SCRIPT=$(find . -name "main.py" -not -path "./.venv/*" 2>/dev/null | head -1)
  [ -z "$ENTRY_SCRIPT" ] && ENTRY_SCRIPT="src/main.py"

  cat > run.bat << BATEOF
@echo off
set ENV=production
set UV_PYTHON_INSTALL_DIR=%~dp0python
echo Initializing environment...
"%~dp0uv.exe" run --no-sync ./$ENTRY_SCRIPT
pause
BATEOF
  echo "  OK: run.bat written (entry: $ENTRY_SCRIPT)"
fi
```

### Step 13 — Final instructions

```bash
echo ""
echo "=== Next Steps ==="
echo "  1. Review the audit results above"
echo "  2. Run prepare.bat on the BUILD machine to create the .7z"
echo "  3. Copy the .7z to the PROD machine and extract"
echo "  4. Configure install.bat with your service manager (NSSM, winsw, sc, etc.)"
echo "  5. Run install.bat as Administrator on PROD to register the service"
echo ""
echo "  Quick fix: /nobuild_deploy --fix   to regenerate all .bat files"
