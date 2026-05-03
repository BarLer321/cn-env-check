# cn-env-check
国内使用Claude code检测网络环境以及是否被墙，并配置镜像网站
 Checks 7 tools (Node.js, Python, Chrome, LibreOffice, Pandoc, Poppler, Git), reports version + path for each, and
  issues a GO/NO-GO verdict:

  ┌─────────┬─────────────────────────────────────┐
  │ Status  │               Meaning               │
  ├─────────┼─────────────────────────────────────┤
  │ OK      │ Installed, good version, path known │
  ├─────────┼─────────────────────────────────────┤
  │ MISSING │ Not found anywhere                  │
  ├─────────┼─────────────────────────────────────┤
  │ OLD     │ Found but ancient version           │
  ├─────────┼─────────────────────────────────────┤
  │ BROKEN  │ Found but can't execute             │
  └─────────┴─────────────────────────────────────┘

  Key design decisions

  - Read-only by default — report first, fix only when you ask
  - Checks 5+ Chrome paths — Program Files, Program Files (x86), AppData, both /c/ and /mnt/c/ prefixes
  - Mirrors baked in — Tsinghua for pip, npmmirror for npm, USTC for apt. Never touches googleapis.com or other blocked
  domains
  - Honest when stuck — if a tool can't be found and can't be downloaded via mirrors, it says so directly instead of
  fabricating

  GO/NO-GO logic

  - GO: Node.js + Python + Git are installed, Chrome found at at least one path
  - NO-GO: Node.js or Python completely missing, or 3+ tools gone (environment was never set up)
