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
   │  └─ fastfields-cpu-lib  ─┐ # Header + source files that end up in "fastfields-cpu.{so|dylib|dll}"
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
      └─ fastfields-cuda-lib ─┤ # Header + source files that end up in "fastfields-cuda.{so|dylib|dll}"
                              │ # * all functions are exported
                              │ # * inputs are dlpack tensors with CUDA memory(no template)
                              │ # * dispatches dlpack dtype to correct template implementation
                              │ # * compiled by nvcc
                              │
                              └ fastfields-lib   # Header + souce files that end up in "fastfields.{so|dylib|dll}"
                                                 # * all functions are exported
                                                 # * inputs are dlpack tensors (no template)
                                                 # * dispatches dlpack device to correct lib (cpu or cuda)
                                                 # * links against fastfields-cpu and fastfields-cuda
                                                 # * compiled by gcc/lvmm
```

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
