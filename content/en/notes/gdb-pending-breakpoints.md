---
title: Set pending breakpoints in shared libraries (GDB)
date: 2026-06-29T00:05:00+00:00
tags:
  - gdb
  - debugging
  - shared-libraries
categories:
  - Debugging
  - Tooling

---

When the code you want to break on lives in a shared library that hasn't been
loaded yet, GDB can't resolve the breakpoint right away. Turn on *pending*
breakpoints so GDB sets it once the library loads.

```bash
set breakpoint pending on
break <source file name>:<line number>
```

Reference:
[How to set breakpoints on future shared libraries](https://stackoverflow.com/questions/100444/how-to-set-breakpoints-on-future-shared-libraries-with-a-command-flag)
