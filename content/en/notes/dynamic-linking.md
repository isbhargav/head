---
title: Dynamic linking
date: 2026-06-29T00:25:00+00:00
tags:
  - linker
  - c
  - dynamic-linking
  - resources
categories:
  - Linking
  - C

---

Unlike static libraries, dynamically-linked libraries (`.so` / `.dylib` /
`.dll`) are resolved at load/run time rather than baked into the binary. The
binary just records which shared objects it needs, and the dynamic linker
loads and resolves them when the program starts.

A great walkthrough of how dynamic linking actually works:
[How dynamic linking works (video)](https://youtu.be/zJ8Dp3nYmpM?si=6UXgGgNplz2n6apT)
