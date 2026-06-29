---
title: VS Code C/C++ debug config (lldb)
date: 2026-06-29T00:35:00+00:00
tags:
  - vscode
  - debugging
  - lldb
  - c
categories:
  - Debugging
  - Tooling

---

A minimal `launch.json` configuration for debugging a C/C++ binary in VS Code
with LLDB via the C/C++ extension (`cppdbg`).

```json
"configurations": [
  {
    "name": "Launch (lldb)",
    "type": "cppdbg",
    "request": "launch",
    "program": "${workspaceFolder}/a.out",
    "args": [],
    "stopAtEntry": false,
    "cwd": "${workspaceFolder}",
    "environment": [],
    "externalConsole": false
  }
]
```

Reference: [Debug C++ in VS Code with LLDB](https://code.visualstudio.com/docs/cpp/lldb-mi)
