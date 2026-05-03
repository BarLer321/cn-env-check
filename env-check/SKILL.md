---
name: env-check
description: >
  Validate a Chinese Windows/WSL development environment by checking Node.js, Python,
  Chrome, LibreOffice, Pandoc, Poppler, and Git. Uses Tsinghua/USTC mirrors for missing
  tools, prefers local installs over downloads, and produces a GO/NO-GO report.
  TRIGGER when the user asks to check their environment, validate dependencies, run
  pre-flight checks, mentions setup issues, network/download problems, "is X installed?",
  "is my environment ready?", or needs tool verification before starting work.
  Especially important before any project that depends on these tools.
---

# Environment Validator for Chinese Windows/WSL

Run a comprehensive pre-flight check of the user's development environment. This is a
read-only diagnostic workflow — produce a clear GO/NO-GO report, not a repair session
(unless the user explicitly asks for fixes).

## The core idea

Your job is to answer one question: _can the user actually get work done right now?_
Check tools, report versions and paths, flag problems, and give an honest assessment.
Default to GO when the essentials are in place, even if some nice-to-haves are missing.

## Tool checklist

Run each check via Bash. Report status with one of:

- **OK** — installed, reasonable version, path is known
- **MISSING** — not found anywhere
- **OLD** — found but version is suspiciously ancient (use your judgment)
- **BROKEN** — found but can't execute or path looks wrong

### 1. Node.js and npm

```bash
node --version 2>/dev/null && echo "NODE_PATH: $(which node 2>/dev/null || command -v node)" || echo "NODE: MISSING"
npm --version 2>/dev/null && echo "NPM_PATH: $(which npm 2>/dev/null || command -v npm)" || echo "NPM: MISSING"
```

If missing, the fix command is:

```bash
# nvm with Tsinghua mirror:
export NVM_NODEJS_ORG_MIRROR=https://mirrors.tuna.tsinghua.edu.cn/nodejs-release/
```

### 2. Python and pip

```bash
python3 --version 2>/dev/null && echo "PYTHON3_PATH: $(which python3 2>/dev/null || command -v python3)" || echo "PYTHON3: MISSING"
python --version 2>/dev/null && echo "PYTHON_PATH: $(which python 2>/dev/null || command -v python)" || echo "PYTHON: MISSING"
pip3 --version 2>/dev/null || pip --version 2>/dev/null || echo "PIP: MISSING"
```

If pip is missing, the fix is:

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### 3. Chrome

Check multiple possible locations — Chrome may be in Program Files, Program Files (x86),
or the user's AppData. In WSL, check the Windows host paths.

```bash
# Standard Windows paths (accessible from both Windows and WSL via /mnt/c)
for p in \
  "/mnt/c/Program Files/Google/Chrome/Application/chrome.exe" \
  "/mnt/c/Program Files (x86)/Google/Chrome/Application/chrome.exe" \
  "/c/Program Files/Google/Chrome/Application/chrome.exe" \
  "/c/Program Files (x86)/Google/Chrome/Application/chrome.exe"; do
  [ -f "$p" ] && echo "CHROME: $p" && break
done

# Also search in LOCALAPPDATA (Windows-side)
CHROME_LOCAL=$(find /mnt/c/Users /c/Users -name "chrome.exe" -path "*/Google/Chrome/*" 2>/dev/null | head -3)
[ -n "$CHROME_LOCAL" ] && echo "CHROME_ALT: $CHROME_LOCAL"
```

**Never attempt to download Chrome.** It's almost certainly blocked. If Chrome is not
found, report the path where it should be and tell the user to install it manually.

### 4. LibreOffice

```bash
soffice --version 2>/dev/null && echo "SOFFICE_PATH: $(which soffice 2>/dev/null || command -v soffice)" || echo "SOFFICE: not in PATH"

# Check common install directories
for p in \
  "/mnt/c/Program Files/LibreOffice/program/soffice.exe" \
  "/mnt/c/Program Files (x86)/LibreOffice/program/soffice.exe" \
  "/c/Program Files/LibreOffice/program/soffice.exe" \
  "/c/Program Files (x86)/LibreOffice/program/soffice.exe"; do
  [ -f "$p" ] && echo "SOFFICE_PATH: $p" && break
done
```

### 5. Pandoc

```bash
pandoc --version 2>/dev/null | head -1 && echo "PANDOC_PATH: $(which pandoc 2>/dev/null || command -v pandoc)" || echo "PANDOC: MISSING"
```

### 6. Poppler (pdftotext, pdfinfo, etc.)

```bash
pdftotext -v 2>/dev/null | head -1 && echo "PDFTOTEXT_PATH: $(which pdftotext 2>/dev/null || command -v pdftotext)" || echo "PDFTOTEXT: MISSING"
pdfinfo -v 2>/dev/null | head -1 && echo "PDFINFO_PATH: $(which pdfinfo 2>/dev/null || command -v pdfinfo)" || echo "PDFINFO: MISSING"
pdfimages -v 2>/dev/null | head -1 && echo "PDFIMAGES_PATH: $(which pdfimages 2>/dev/null || command -v pdfimages)" || echo "PDFIMAGES: MISSING"
```

On WSL the fix is: `sudo apt-get install poppler-utils`
On native Windows, poppler is typically bundled with tools that need it (e.g., pdf2image Python package).

### 7. Git

Always check — it's fundamental:

```bash
git --version 2>/dev/null && echo "GIT_PATH: $(which git 2>/dev/null || command -v git)" || echo "GIT: MISSING"
```

## The report

After running all checks, produce this exact structure:

```
========================================
  Environment Validation Report
  [date] [platform: Windows/WSL/both]
========================================

Tool          Status   Version          Path
────          ──────   ───────          ────
Node.js       OK       v20.11.0         /usr/bin/node
npm           OK       10.2.4           /usr/bin/npm
Python        OK       3.12.1           /usr/bin/python3
pip           OK       24.0             /usr/bin/pip3
Chrome        OK       120.0.6099.109   /c/Program Files/Google/Chrome/Application/chrome.exe
LibreOffice   OK       7.6.4.1          /c/Program Files/LibreOffice/program/soffice.exe
Pandoc        MISSING  —                —
Poppler       OLD      0.24.5           /usr/bin/pdftotext
Git           OK       2.43.0           /usr/bin/git

========================================
  GO / NO-GO: GO
========================================

Summary: 6/8 tools available. 1 missing (Pandoc), 1 old (Poppler).

Issues:
  - Pandoc is not installed (needed for document conversion)
  - Poppler is v0.24.5 (from 2015 — may lack features)

Recommended actions:
  - Pandoc: download from https://github.com/jgm/pandoc/releases (GitHub usually accessible)
  - Poppler: sudo apt-get update && sudo apt-get install poppler-utils (WSL)
```

## GO / NO-GO rules

**GO** when all of these are true:

- Node.js and Python are installed and on a version from the last 2 years
- Git is installed
- Chrome is found at a known path (at least one valid location)

**NO-GO** when any of these are true:

- Node.js or Python are completely missing (showstoppers)
- Three or more tools are missing, suggesting the environment was never set up
- Basic shell commands fail (fishy PATH, broken WSL, etc.)

If borderline (e.g., Chrome missing but everything else is fine), lean toward GO with a
clear note about what's missing.

## Mirror configuration

When the user asks you to fix missing tools, configure these mirrors _before_ any install:

```bash
# Python pip — Tsinghua
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# npm — npmmirror (USTC-backed CDN)
npm config set registry https://registry.npmmirror.com

# apt (WSL/Ubuntu) — USTC
sudo sed -i 's|//archive.ubuntu.com|//mirrors.ustc.edu.cn|g' /etc/apt/sources.list
sudo sed -i 's|//security.ubuntu.com|//mirrors.ustc.edu.cn|g' /etc/apt/sources.list
```

## Non-negotiable rules

1. **Never download from blocked domains.** googleapis.com, dl.google.com, storage.googleapis.com,
   and similar are almost certainly blocked by GFW. Do not attempt — it wastes time and
   leaves the user with a half-finished install.

2. **Check local before downloading.** Many tools the user needs are already installed
   (as the LibreOffice experience showed). Check every known path before suggesting a download.

3. **Use mirrors for everything.** Tsinghua (tuna.tsinghua.edu.cn) first, USTC
   (mirrors.ustc.edu.cn) as fallback. For npm, npmmirror.com. Never use the default
   package registry without checking if a mirror is available.

4. **Be honest about what you can't do.** If a tool can't be found and can't be downloaded
   from a mirror, say so clearly. Do not fabricate, guess paths, or pretend success.
   Report it as MISSING with a note about why.

5. **Report first, fix later.** This skill is primarily diagnostic. Don't start installing
   things unless the user explicitly asks. The report gives them control over what happens next.
