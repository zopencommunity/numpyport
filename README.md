# numpyport

z/OS port of [numpy](https://github.com/numpy/numpy) — the fundamental package
for array computing in Python.

## Status

Built and tested against Python 3.12, 3.13 and 3.14.

## Installation

```sh
zopen install numpy
```

Or from the wheel index:

```sh
pip install numpy --extra-index-url https://repo.zopen.community/pypi/wheels/simple/
```

## Linear algebra is the slow fallback

This builds with `-Dallow-noblas=true`, so `numpy.linalg` uses the `lapack_lite`
routines bundled with numpy rather than an optimised BLAS. Everything works —
`solve`, `det`, `inv`, `eigvalsh`, `qr`, `svd` are all exercised by the check —
but large matrix operations are considerably slower than on a platform with a
tuned BLAS.

There is no OpenBLAS on z/OS. The intended follow-on is BLIS (already ported,
BLAS only) paired with a reference LAPACK built by XL Fortran, which would be a
change to `NUMPY_MESON_ARGS` plus the corresponding dependencies. A working
numpy unblocks far more than a fast one, which is why that came second.

## What this port has to deal with

Five z/OS problems, none of them obvious from the failure it produces. They are
documented at length in `buildenv`; in brief:

**numpy vendors a fork of meson.** `vendored-meson/meson` is a submodule of
numpy's *own* meson repository, carrying a `features` module for CPU dispatch
that upstream meson does not have. So the ported system meson cannot be
substituted — doing that fails at `meson_cpu` with `Module "features" does not
exist`. `mesonport`'s z/OS support is applied to the vendored copy instead, from
`meson-patches/`. Without it the build stops at `ERROR: Unable to detect linker`.

Those patches are applied with `gpatch` in `zopen_pre_build` rather than from
`patches/`, because zopen-build applies that directory with `git apply` from the
source root and git refuses paths inside a submodule.

**String literals must be ASCII.** IBM's clang emits EBCDIC literals by default
while the interpreter is an ASCII build, so extensions compile, link and install
perfectly and then every one of them fails to import with `UnicodeDecodeError` —
their module names come back as EBCDIC. `-fzos-le-char-mode=ascii` fixes it, and
it must be that flag rather than `-fexec-charset`: the latter changes the
literals but not the Language Environment, so `printf` stops working and meson's
configure probes start returning `-1`. setuptools passes these flags for
extension builds; meson does not.

**Extensions must bind against libpython's side deck**, or every `Py*` symbol
comes back `UNRESOLVED` from the binder.

**Sources are untagged ASCII**, so z/OS tools read them as EBCDIC and the
compiler reports `unexpected character <U+0080>`.

**The interpreters ship unrelocated paths.** Their pkg-config files name the
machine they were built on rather than where they are installed, so meson
concludes `Python.h` is missing; and `sysconfig`'s `LIBDIR` is wrong for some
interpreters and not others. Both are recomputed from `sys.base_prefix` and
verified before use. See "Upstream issues" below.

Two patches go to numpy itself, in `patches/`. Both are small guards on standard
predefined macros that land on paths numpy already maintains, and both would be
reasonable upstream:

| patch | why |
| --- | --- |
| `npy_cpu_features_zos.patch` | numpy treats `__s390x__` as implying Linux on Z and includes `<sys/auxv.h>`; z/OS is s390x with no auxiliary vector |
| `npy_tls_zos.patch` | a thread-local symbol cannot be resolved across the z/OS DLL boundary; numpy already has an empty `NPY_TLS` for platforms without TLS |

## Upstream issues worth fixing at the source

Two of the accommodations above are working around defects outside this port:

1. **IBM Open Enterprise Python ships unrelocated build metadata.** Every
   interpreter's `python-<version>.pc` has
   `prefix=/home/pyzbld/buildbot/worker/...`, and `sysconfig`'s `LIBDIR` points
   at paths that exist for some interpreters and not others — 3.14's names a
   different product directory entirely. setuptools-based builds never notice
   because they use `sysconfig` paths that are correct; anything meson-based asks
   pkg-config and breaks. Relocating at packaging time would fix it for everyone.

2. **zopen's `PYTHONPATH` gains a leading empty entry**, because ports build it
   as `PYTHONPATH="${PYTHONPATH}:..."` from an unset variable. Python expands the
   empty entry to the working directory at interpreter startup, which puts the
   *source tree* on `sys.path` during a port's check. Most ports never notice;
   numpy's package sits at its repository root, so it shadowed the installed
   wheel and failed with `cannot import name 'version' from partially
   initialized module 'numpy'`. The fix is
   `PYTHONPATH="${PYTHONPATH:+${PYTHONPATH}:}..."` in the `zopen_append_to_env`
   snippets.

## What the check covers

Importing numpy proves very little here — the failure this port exists to
prevent is an extension that builds and installs perfectly and imports as
mojibake. So the check exercises arithmetic, matmul and reductions; the whole of
`linalg` (`solve`, `det`, `inv`, `eigvalsh`, `qr`, `svd`) because that is the
bundled fallback and the part most likely to be wrong without a BLAS; FFT;
random; sorting; and an explicit big-endian layout and byteswap assertion, since
z/OS is big-endian and few numpy builds exercise that path.
