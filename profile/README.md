## Welcome to fastfields 👋

Fast tools for dense scalar and vector fields (implemented in C++/CUDA).

### Project hierarchy

C++/CUDA sources
```
└─ fastfields-kernels           # "Kernels" that operate at the single-element level
   │                            # * header-only
   │                            # * all functions are static (= hidden to linker)
   │                            # * data and pointer types are templated
   │                            # * sizes can be either statically or dynamically defined
   |
   ├─ fastfields-csrc-cppyy     # Functions that aim to be JIT-compiled by cppyy
   │                            # * header-only
   │                            # * implement parallel loops across elements
   │                            # * data and pointer types are templated
   │                            # * sizes are statically defined
   |
   ├─ fastfields-csrc-cupy      # CUDA kernels that aim to be JIT-compiled by cupy
   │                            # * header-only
   │                            # * operate on a subset of element
   │                            # * data and pointer types are templated
   │                            # * sizes are statically defined
   |
   ├─ fastfields-cpu-impl       # Header-only low level implementation that runs on the CPU
   │  │                         # * header-only
   │  │                         # * all functions are static (= hidden to linker)
   │  │                         # * implement parallel loops across elements
   │  │                         # * data and pointer types are templated
   │  │                         # * sizes are dynamically defined
   |  |
   │  └─ fastfields-cpu-lib  ─┐ # Header + source files that end up in "fastfields-cpu.{ext}"
   │                          │ # * all functions are exported
   │                          │ # * inputs are dlpack tensors with CPU memory (no template)
   │                          │ # * dispatches dlpack dtype to correct template implementation
   │                          │ # * compiled by gcc/lvmm
   |                          │
   └─ fastfields-cuda-impl    │ # Header-only low level implementation that runs on the CPU
      │                       │ # * header-only
      │                       │ # * all functions are static (= hidden to linker)
      │                       │ # * implement parallel loops across elements
      │                       │ # * data and pointer types are templated
      │                       │ # * sizes are dynamically defined
      │                       │
      └─ fastfields-cuda-lib ─┤ # Header + source files that end up in "fastfields-cuda.{ext}"
                              │ # * all functions are exported
                              │ # * inputs are dlpack tensors with CUDA memory(no template)
                              │ # * dispatches dlpack dtype to correct template implementation
                              │ # * compiled by nvcc
                              │
                              └ fastfields-lib   # Header + souce files that end up in "fastfields.{ext}"
                                                 # * all functions are exported
                                                 # * inputs are dlpack tensors (no template)
                                                 # * dispatches dlpack device to correct lib (cpu or cuda)
                                                 # * links against fastfields-cpu and fastfields-cuda
                                                 # * compiled by gcc/lvmm
```

### ⚠️ Local dev-tree gotcha: symlinks, not live submodules

Every repo above pins its cross-repo dependency via a **git submodule
gitlink** (e.g. `fastfields-cpu-impl`'s `kernels` pointing at a specific
`fastfields-kernels` commit), but the **local dev tree** wires those same
paths as plain **symlinks to sibling checkouts** instead (`cpu-impl/kernels
-> ../fastfields-kernels`, `cpu-lib/impl -> ../fastfields-cpu-impl`, etc. —
see each repo's own CLAUDE.md for its exact symlink list). This is
convenient for iterating across layers without re-pinning constantly, but it
means **a build in this dev tree compiles against whatever commit/branch
each sibling checkout happens to be on right now — not the pinned gitlink,
and not necessarily `origin/main`.**

**Before concluding a compile/link failure is a real cross-repo API bug**,
check every sibling repo reached via a symlink is actually on (or at least
not behind) the branch you mean to test against — usually `origin/main`.
A sibling checkout sitting on a stale feature branch produces a compile
error that looks exactly like a real incompatibility (wrong arity, "call to
non-static member function", missing symbol, etc.), and is easy to
mistake for one.

This has already produced at least one false-positive bug report:
`fastfields-cpu-impl#40` (closed `not_planned`) diagnosed a "static vs.
instance method" API break between `fastfields-kernels` and
`fastfields-cpu-impl` that had, in fact, already been fixed on both repos'
`main` two days earlier (PR `fastfields-cpu-impl#37`) — the build that
"failed" was against a `fastfields-cpu-impl` checkout still sitting on an
older feature branch. `git -C <sibling> fetch && git -C <sibling> log
origin/main -1` (or just diffing against a fresh clone) is a cheap
pre-check before filing a "cross-repo incompatibility" issue.

Python packages
```
├─ fastfields-cpu-impl   ─┐
├─ fastfields-csrc-cupy  ─┤
│                         └─ jitfields       # Python bindings using JIT compilation (cppyy and cupy)
│
└─ fastfields-lib
   └─ fastfields-binds                       # Python bindings from dlpack to fastfields-lib, using nanobinds
      ├─ fastfields-numpy   ─┐               # User-friendly numpy interface (with checks)
      ├─ fastfields-cupy    ─┤               # User-friendly cupy  interface (with checks)
      └─ fastfields-torch   ─┤               # User-friendly torch interface (with checks and autodiff)
                             └─ fastfields   # Meta package
```
