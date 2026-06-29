---
title: Ways to start or attach GDB
date: 2026-06-29T00:00:00+00:00
tags:
  - gdb
  - debugging
  - c
categories:
  - Debugging
  - Tooling

---

A few different ways to get a program running under GDB, depending on whether
it's already running, needs arguments, or reads from stdin.

```bash
# Attach to an already-running process by PID
$ gdb -p <pid>

# Launch a program and run a script (e.g. node)
$ gdb node
> run index.js

# Pass arguments on the command line
$ gdb --args program_name arg1 arg2

# Load a binary, set args, then run
$ gdb
> file a.out --stdio
> set args arg1 arg2
> run

# Or pass args directly to run
$ gdb
> file a.out --stdio
> run arg1 arg2
```
