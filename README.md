# Memory Allocator & Unix Mini-Shell in C

A three-tier `malloc`/`free` implementation (free list, buddy system, `mmap`) and a Unix-style command shell, written in C on top of POSIX system calls.

![Language](https://img.shields.io/badge/language-C-blue)

> Academic project — Ensimag (Grenoble INP), 2021 — team project (2 students), Systems Programming course.

## Overview

This repository contains three systems-programming labs built from course skeletons:

| Directory | What it is |
|---|---|
| `ensimag-malloc/` | A dynamic memory allocator (`emalloc` / `efree`) built directly on `mmap`, with a different strategy per allocation size |
| `ensimag-shell/` | `ensishell`, a mini Unix shell: fork/exec, pipes, I/O redirection, background jobs |
| `ensimag-rappeldec/` | Warm-up C exercises (linked lists, bit manipulation, `qsort`) |

The allocator is the main part. It shows how a real `malloc` handles different sizes, keeps metadata next to user data, and puts freed memory back together.

## Memory allocator (`ensimag-malloc/`)

Each request is sent to one of three allocators by size (`src/mem.c`):

| Size | Strategy | File |
|---|---|---|
| ≤ 64 B | **Fixed-size chunks (96 B)** in a free list, refilled from `mmap`ed pools that double in size | `src/mem_small.c` |
| 65 B – 128 KiB | **Buddy allocator**: an array of free lists indexed by power of two (`TZL[48]`), with recursive block splitting and buddy merging | `src/mem_medium.c` |
| ≥ 128 KiB | **Direct `mmap` / `munmap`** per allocation | `src/mem_large.c` |

### Technical highlights

- **Block headers and footers** (`src/mem_internals.c`): every block is wrapped as `[size | magic | user data | magic | size]` (4 × 8 bytes). `efree` reads the header to find the block's real size and kind.
- **Kind stored in the magic number**: the magic value is computed from the block address with Knuth's MMIX linear congruential generator. Its two low bits hold the allocator kind (small / medium / large), so `efree` can route a pointer without any lookup table.
- **Buddy merging**: a block's buddy address is found with `ptr XOR size`. Free buddies are removed from their list and merged level by level.
- **Aligned pool growth**: new buddy pools are `mmap`ed at twice the size and then aligned to a multiple of their size, which the XOR buddy computation needs.
- **Testing**: GoogleTest suites (`tests/test_generic.cc`, `test_buddy.cc`, `test_mark.cc`), a Python test binding (`tests/mempymodule.c`, built as `libmempy.so`) and an interactive `memshell` for manual testing.

## Mini-shell (`ensimag-shell/`)

`src/ensishell.c` adds command execution to a provided command-line parser (`src/readcmd.c`):

- Runs commands with `fork` + `execvp`, and waits for foreground jobs with `waitpid`
- Connects commands with **pipes** (`pipe` + `dup2`)
- Supports **I/O redirection** (`<` and `>`) with `open` / `dup2`
- Runs **background jobs** (`&`) and keeps them in a linked list; the built-in `jobs` command lists running jobs and removes finished ones (`waitpid(..., WNOHANG)`)
- Ruby integration tests in `tests/` (`testForkExec.rb`, `testInOut.rb`, `testJobs.rb`, ...)

> **Status:** the shell was left unfinished. The last committed version of `ensishell.c` does not compile as-is (it contains a syntax error).

## Tech stack

C (GNU11) · POSIX (`mmap`, `fork`, `execvp`, `pipe`, `dup2`, `waitpid`) · CMake · GoogleTest · GNU Readline · Ruby (shell tests) · Valgrind

## Project structure

```
ensimag-malloc/
  src/        mem.c, mem_small.c, mem_medium.c, mem_large.c, mem_internals.c, memshell.c
  tests/      GoogleTest suites + Python module
ensimag-shell/
  src/        ensishell.c, readcmd.c
  tests/      Ruby test scripts
ensimag-rappeldec/
  src/, tests/  C refresher exercises
```

## Build & run

Each sub-project uses CMake on its own. Build it out of source in its `build/` directory:

```sh
cd ensimag-malloc/build      # or ensimag-shell/build, ensimag-rappeldec/build
cmake ..
make
make test                    # or: make check
```

- Allocator: requires GoogleTest (and optionally Python 3 headers for `libmempy`). It produces `libemalloc.so`, `alloctest` and `memshell`.
- Shell: uses GNU Readline if it is installed and a built-in fallback otherwise. It produces `ensishell`.

These projects were built on Linux (Ensimag lab machines) and depend on Linux-specific `mmap` flags.

## License

The course skeletons are GPLv3+ (see each `LICENSE`).
