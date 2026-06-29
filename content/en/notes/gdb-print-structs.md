---
title: Pretty-print structs in GDB
date: 2026-06-29T00:10:00+00:00
tags:
  - gdb
  - debugging
  - c
categories:
  - Debugging
  - Tooling

---

To inspect what fields a struct has and print it readably:

```bash
# Turn on pretty printing for nicer, multi-line output
> set print pretty on

# Show the type/fields of a variable or pointer
> ptype p

# Pretty-print the value a pointer points to
> print -pretty -- *pointer
```

Reference:
[How do I show what fields a struct has in GDB](https://stackoverflow.com/questions/1768620/how-do-i-show-what-fields-a-struct-has-in-gdb/55234381)
