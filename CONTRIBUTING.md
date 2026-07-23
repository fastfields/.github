# Contributing to fastfields

fastfields is a set of **layered C++/CUDA libraries** for dense scalar and
vector fields, plus thin **Python bindings**. It is a rewrite of the
JIT-compiled `jitfields`, dropping the cppyy/torchlib dependency: data crosses
backends via **DLPack**, and the Python layer binds with **nanobind**.

The project is spread across several repositories, each one layer of the stack,
connected by git submodules:

```
fastfields-kernels        voxelwise math (header-only, backend-agnostic, templated)
  ├─ fastfields-cpu-impl   CPU loops over elements  ─ fastfields-cpu-lib   (libfastfields-cpu, dtype dispatch)  ┐
  └─ fastfields-cuda-impl  CUDA kernels + launchers ─ fastfields-cuda-lib  (libfastfields-cuda, nvcc)           ┴─ fastfields-lib (libfastfields, DLTensor, device dispatch)
                                                                                                                   └─ fastfields-bind-py (fastfields.dlpack, nanobind)
                                                                                                                        ├─ fastfields-numpy / -cupy / -torch (fastfields.{numpy,cupy,torch})
                                                                                                                        └─ fastfields             (fastfields.any)
```

Each repo has a **`CLAUDE.md`** describing its role, layout, and conventions;
the hub repo **`fastfields-lib/MIGRATION.md`** holds the module status matrix,
the per-module porting pattern (`distance.{h,cpp}` is the template), and the
running list of bugs fixed. Read the relevant one before changing core code.

Keep each layer doing its one job. **Prefer deleting code to adding it**, and
when in doubt push complexity *down* the stack (into the kernels) rather than
duplicating it across the lib layers.

---

## Issue-based workflow

Work is tracked in issues. Before writing code:

1. **File an issue** describing the change (or claim an existing one). One issue
   = one coherent change.
2. **Break up big work into sub-issues.** A multi-repo or multi-module effort
   gets an umbrella issue with a checklist, and each part its own issue that
   references the parent (`part of #NN`). Land the parts as separate PRs; close
   the umbrella when the last one merges.
3. **Triage with a label** (see below) — every issue should carry at least one.
4. **Reference the issue** from your branch and PR, and close it from the PR
   body with `Closes #NN`. Use `part of #NN` when a PR advances but doesn't
   finish an issue.

Because the stack is layered, say **which repo(s)** an issue touches, and note
when a change needs a matching submodule bump in a parent repo.

### Labels (triage)

| label | for |
|---|---|
| `bug` | incorrect behaviour, UB, crashes |
| `enhancement` | new feature or API (e.g. porting a module to a new layer) |
| `perf` | efficiency / codegen, correctness unaffected |
| `maintainability` | refactors, dedupe, internal cleanup |
| `documentation` | docs, comments, CLAUDE.md, MIGRATION.md |
| `cuda` | needs a CUDA toolchain and/or a GPU to build or validate |
| `blocked` | needs hardware, a design decision, or human input to proceed |
| `good first issue` | small, well-scoped, low-context |

`bug` / `enhancement` / `documentation` are GitHub defaults; add the rest once.

---

## Branches, commits, PRs

- **One branch and one PR per task — never bundle.** A branch fixes exactly ONE
  thing (one issue, one feature, one refactor). If you notice an unrelated bug
  while working, resist the urge to fix it here — file an issue or note it and
  give it its OWN branch. A PR that touches several unrelated concerns is hard to
  review, revert, bisect, and reason about; split it. Do this even when a
  workflow hands you a shared branch: rebase each concern onto its own branch
  rather than stacking them.
- **Name the branch for the task** under your author prefix (`claude/…` for
  Claude-authored work, `<user>/…` otherwise), descriptive and referencing the
  issue: `claude/fix-42-dt-l1-dispatch`, `claude/port-posdef-cuda-launchers`,
  `claude/ci-nvcc-compile`. (The `fix:`/`feat:`/… prefix belongs on the commit
  *subject*.) **Never commit straight to `main`.**
- **Commit subjects** use a conventional prefix, then a concise imperative
  summary:

  ```
  fix:      a bug / UB / crash
  feat:     a new feature or API surface (e.g. a newly-ported module/op)
  refactor: behaviour-preserving restructuring (dedupe, extract, rename)
  perf:     an efficiency change (behaviour unchanged)
  docs:     docs / comments / CLAUDE.md / MIGRATION.md / this file
  test:     tests only
  chore:    build, CI, tooling, Makefiles, submodule bumps
  ```

  Explain the *why* in the body, not just the *what*, and reference the issue.
- **One PR per branch/task.** Title mirrors the single change; body says what
  changed, why, and **how it was verified** (paste the passing test line / the
  clean nvcc compile), and links the issue. If a repo has a
  `.github/pull_request_template.md`, fill it in.
- **Squash-merge**, then delete the branch. A change spanning layers is a series
  of small PRs (one per repo) plus the submodule-pointer bump — not one giant PR.

---

## Build & test gate (must pass before you open a PR)

There is **no GPU in CI**, so the automated gate is the **CPU C++ path** and the
**Python** suites; CUDA is validated by **compile + link** only. Runtime CUDA
correctness (atomics, races, the flagged scratch-buffer sizings) needs real
hardware — label those `cuda` / `blocked` and add them to the GPU-CI list in
MIGRATION.md rather than claiming them tested.

Submodules must be checked out (or symlinked) so the include nesting resolves:
`cpu-lib/impl -> cpu-impl`, `cpu-impl/kernels -> kernels`, `lib/cpu -> cpu-lib`
(and the cuda mirror).

**C++ (CPU) — the source of truth:**
```sh
make -C fastfields-cpu-lib test CXX=clang++   # builds libfastfields-cpu.so + runs tests/test_*.cpp
make -C fastfields-lib      all  CXX=clang++   # builds libfastfields.so (links the cpu lib)
```
Every module ships a `tests/test_<module>.cpp` that validates the CPU path
against a **brute-force / analytic reference** (identity, adjointness,
`solve∘matvec ≈ id`, finite-difference stencils, …) across dtypes and sizes,
wired into the Makefile `MODULES`/test discovery. Sanitize host code you touch:
```sh
clang++ -std=c++11 -O1 -g -fsanitize=address,undefined -I. tests/test_<m>.cpp <m>.cpp -o /tmp/t && /tmp/t
```

**CUDA — compile + link gate (needs nvcc; e.g. Ubuntu `nvidia-cuda-toolkit`):**
```sh
nvcc -std=c++14 -c -arch=sm_70 -x cu -I. <module>.cpp -o /tmp/<m>.o   # in fastfields-cuda-lib
make -C fastfields-cuda-lib libcuda                                    # links libfastfields-cuda.so
```
A module joins the cuda-lib `MODULES` only once its `cuda-impl` **host launchers
compile** (device-alloc/copy the shape+stride arrays, launch the `CUGLOB` kernel
over the grid, forward the CUDA `stream`, free) — see `distance_euclidean.h` for
the reference launcher. Missing launchers are honest `throw` stubs, not silent
gaps.

**Python:**
```sh
pip install <repo>                       # regular install (native-namespace + editable don't mix)
cd /tmp && python -m pytest <repo>/tests -q   # run from a neutral cwd
ruff check <repo> && ruff format --check <repo>
```
The torch wrapper's autograd is checked with `torch.autograd.gradcheck`; cupy's
runtime tests skip cleanly without a GPU.

CI (`.github/workflows/`) runs the CPU C++ matrix, the Python suites + ruff, and
the nvcc compile gate on every PR.

---

## Coding style

### C++ / CUDA

- **C++11** for the libraries (the Makefiles use clang-style flags); the
  nanobind extension is C++17. No C++14/17 features in kernels/impl/lib.
- **Respect the layer boundaries.** Kernels are header-only, `static`/`inline`,
  templated, and operate on a *single element* — no device loops there. `impl`
  owns the loops. The exported `lib` ABIs take **unsafe pointers** (`cpu`/`cuda`
  lib) or **`DLTensor`** (`lib`) — **no templates in an exported signature**;
  dispatch dtype/dim/spline/bound to the templated impl.
- **Backend-agnostic kernels.** Use the `FF_NAMESPACE_BEGIN(FF)/(FF_DEVICE)/…`
  macros and `CUDEV`/`CUGLOB`/`CUHOST` from `cuda_switch.h`; never hard-code
  `ff::cpu` — the same source must compile for CPU and CUDA. Keep the
  NVRTC-vs-AOT split (`__CUDACC_RTC__`) and the `__CUDA_ARCH__` atomic guards
  intact.
- **Add a module by analogy.** Copy the `distance` module through the layers
  (impl entry → `cpu/cuda-lib/<m>.{h,cpp}` dispatch → `lib/<m>.{h,cpp}` device
  dispatch), add it to the three `MODULES` lists, and ship a CPU test. Narrow
  pointer/stride arrays with `copy_if_needed` using the **correct array
  lengths** (a length bug there is a heap OOB — it has bitten us before).
- **Single source of truth.** If the same logic must hold in two places (a CPU
  loop and its CUDA launcher, a dispatch check and the kernel's real
  expectation), keep them in agreement — a silent divergence is how UB and
  segfaults creep in.

### Python

- **PEP 420 namespace package** — no repo ships a `fastfields/__init__.py`; each
  ships only its `fastfields/<sub>/` subpackage. Keep the module layout
  consistent across wrappers (`_util`/`_dt`/`_sym`/`_resample`/`__init__`).
- **NumPy-style docstrings** and **type annotations** on every public function;
  `from __future__ import annotations` at the top of every module; optional
  backend types (`torch`, `cupy`) imported under `if TYPE_CHECKING:`.
- **Ruff** (`line-length = 79`, `select = ["B","E","F","I","W"]`) — `ruff check`
  clean and `ruff format`-ed.
- The library is **stride-aware**: pass arrays through with their native strides
  (zero-copy, incl. `broadcast_to`/`expand` views); never force a contiguous
  copy on a read-only or in-place input unless an op genuinely requires it — and
  if it does, say why in a comment. Outputs are real contiguous buffers.

### Naming

- `snake_case` for functions, members, variables; module file names match the op
  family (`distance`, `posdef`, `pushpull`, `reg_field`, `reg_flow`, `resize`,
  `restrict`, `splinc`). `_`-prefixed names are internal.
- Op renames that avoid a namespace/function clash are canonical:
  `resample` (resize), `restriction` (restrict), `spline_coeff` (splinc).
  `restriction` **accumulates** into its output — pre-zero it.
- Match the surrounding code's comment density and idiom. Comments explain *why*
  and how the tricky bits work — not what a line obviously does.

### Review

Correctness-critical changes — dispatch layers, `copy_if_needed` lengths, kernel
math, CUDA launchers — get a **skeptical diff review** that verifies behaviour
against a reference (the brute-force probe the tests use, or fold-vs-runtime for
the UB-sensitive paths) before merge. **Correctness beats brevity beats
cleverness.**

---

## Python packaging, CI & docs

The Python repos follow the conventions of the **bagofseeds** packages
(`bagof-*`, `fiery-*`) — study one (e.g. `bagof-magic`) before adding a repo.

### pyproject

- **PEP 420 native namespace.** No repo ships `fastfields/__init__.py`; each
  ships only its `fastfields/<sub>/` subpackage with
  `[tool.setuptools.packages.find] namespaces = true` so the distributions
  merge into one `fastfields` import root.
- **Dynamic version via `versioningit`** (`dynamic = ["version"]`; setuptools +
  versioningit + wheel build backend). The version comes from git tags; a
  `_version.py` is written into the package and git-ignored. `default-tag` keeps
  a tag-less checkout building.
- **Rich `[project]` metadata**: `authors`/`maintainers`, `keywords`,
  trove `classifiers` (incl. `License :: OSI Approved :: MIT License`, the
  supported `Programming Language :: Python :: 3.x`, `Typing :: Typed`), and a
  `[project.urls]` block (Homepage / Documentation / Issues / Repository).
- **Tooling config lives in pyproject**: `ruff` (`line-length = 79`,
  `select = ["B","E","F","I","W"]`, `format.quote-style = "preserve"`),
  `codespell`, `coverage` (`omit` the generated `_version.py`), and
  `[tool.pytest.ini_options] testpaths = ["tests"]`.

### Workflows

Each repo carries **thin, self-contained** workflows under `.github/workflows/`
(bagofseeds factors the real steps into a central `bagofseeds/actions` repo and
keeps one-line callers per package; we inline the steps for now — extract a
`fastfields/actions` repo later if the duplication bites):

- `lint.yaml` — `ruff check`, `ruff format --check`, `codespell` (push + PR).
- `test.yaml` — a Python-version matrix: checkout `submodules: recursive`,
  install the package `.[test]`, run `pytest` from a neutral cwd. Path-filter on
  `fastfields/**`, `tests/**`, `pyproject.toml`. Wrapper repos need
  `fastfields-dlpack`; until it is on PyPI / the wheel index they build it from
  the sibling repo.
- `docs.yaml` — build the site and deploy to Pages (`permissions: pages: write,
  id-token: write`, `concurrency: {group: pages}`).
- C++ repos get a `test.yaml` too: the **cpu-lib** one (`make test
  CXX=clang++`) is the real gate; **cuda-lib** installs the CUDA toolkit and
  runs the nvcc **compile+link** gate (no GPU).

Trigger on push to `main` + `pull_request`; **enable Pages with the "GitHub
Actions" source** once per repo (Settings → Pages) or `deploy-pages` fails.

### Docs

- **[zensical]** (a mkdocs-material successor) configured in `zensical.toml`,
  with the `mkdocstrings` python handler set to `docstring_style = "numpy"` so
  the API pages render from the same NumPy-style docstrings the code already
  carries. `docs/` holds `index.md`, `api/`, and `requirements.txt`.
- Each package publishes to `https://fastfields.github.io/<repo>/`; the C++
  `fastfields-lib` publishes a prose/architecture site (no mkdocstrings). The
  org **landing page** at `https://fastfields.github.io/` (the
  `fastfields/fastfields.github.io` repo) introduces the project and links out
  to each package's API site and to the wheel index; every package's nav links
  back to it.

### Wheel distribution

Follow **PyTorch's model**, served from the `fastfields/whl` repo
(<https://fastfields.github.io/whl/>):

- The compute backend is the wheel's **local version label**
  (`…+cpu`, `…+cu124`); the index has one PEP 503 folder per backend
  (`cpu/`, `cu118/`, `cu124/`), installed with `--extra-index-url`.
- **PyPI** ships the broad default — a CUDA wheel (`cu124`) on Linux/Windows, a
  CPU wheel on macOS; the index additionally offers CPU and other CUDA lines.
- CUDA wheels are **fat** (multi-`sm_*` SASS + forward-compatible PTX) to run on
  as many GPUs as possible, trading binary size and build time for coverage.

[zensical]: https://github.com/mkdocs/zensical

---

Thanks for keeping fastfields fast.
