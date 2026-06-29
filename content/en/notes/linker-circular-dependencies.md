---
title: Resolve circular dependencies between static libraries
date: 2026-06-29T00:20:00+00:00
tags:
  - linker
  - c
  - gcc
  - static-library
categories:
  - Build
  - Linking

---

The linker processes libraries in order and only pulls in symbols it needs *so
far*. When two (or more) static libraries depend on each other, that single
pass isn't enough. Wrap the mutually-dependent libraries in a group so the
linker scans them repeatedly until all symbols resolve.

```bash
-Wl,--start-group -lmy_lib -lyour_lib -lhis_lib -Wl,--end-group -ltheir_lib
```

`-Wl,<option>` in GCC passes `<option>` straight through to the linker (`ld`).
So `--start-group` / `--end-group` are linker options, not compiler ones.
