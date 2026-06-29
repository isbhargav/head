---
title: Compiling and linking with a static library
date: 2026-06-29T00:15:00+00:00
tags:
  - c
  - linker
  - static-library
  - gcc
categories:
  - Build
  - C

---

A static library (`.a`) is just an archive of object files that gets baked
directly into your final binary at link time.

```bash
# 1. Compile sources to object files
gcc -c mylib.c -o mylib.o

# 2. Bundle the object files into a static library (archive)
ar rcs libmylib.a mylib.o

# 3. Link your program against it
#    -L. → look for libraries in the current dir
#    -lmylib → link libmylib.a
gcc main.c -L. -lmylib -o main
```

The library name passed to `-l` drops the `lib` prefix and `.a` suffix:
`libmylib.a` becomes `-lmylib`.
