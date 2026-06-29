---
title: CMake out-of-source build
date: 2026-06-29T00:30:00+00:00
tags:
  - cmake
  - build
  - c
categories:
  - Build
  - Tooling

---

Keep generated build files out of your source tree with an out-of-source build.

```bash
# Configure: create a build/ dir containing CMakeCache.txt and generated files
cmake -B build

cd build

# Build, using all available cores
cmake --build . --parallel
```

Need to change a CMake variable (e.g. `PYTHON_EXECUTABLE`)? Pass it at
configure time with `-D`:

```bash
cmake -B build -DPYTHON_EXECUTABLE=/usr/bin/python3
```
