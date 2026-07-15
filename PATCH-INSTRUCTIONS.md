# Simbody patch extraction and upstream PR setup

This document describes how to go from the build2 port of Simbody
(`~/git/b2/simbody/libsimbody/`) to a set of numbered patch files, a
`PATCHES.md` catalog, PR description files, and `patch/` branches in the
Simbody GitHub fork -- following the same approach used for opensim-core in
`~/git/b2/opensim/`.

The analogous process for opensim-core is already complete in
`~/git/b2/opensim/` (see `patches/PATCHES.md` and `prs/` there).
This document covers the equivalent work for Simbody, tracked here in
`~/git/b2/upstream-simbody/`.

---

## Repository layout

| Path | Role |
|---|---|
| `~/git/b2/simbody/upstream/` | Upstream Simbody source (build2 tree layout) |
| `~/git/b2/simbody/libsimbody/libsimbody/` | Patched Simbody source (build2 port) |
| `~/git/b2/upstream-simbody/` | Git fork of Simbody for upstream PRs (`helmesjo/simbody`, branch `master`) -- this repo |
| `patches/` | To be created: numbered patch files (in this repo) |
| `prs/` | To be created: PR description files (in this repo) |
| `patches/PATCHES.md` | To be created: patch catalog (in this repo) |

`upstream/` and `libsimbody/libsimbody/` share the same directory layout as
the CMake Simbody repo, so a recursive diff between them is the canonical
source of truth for all patches:

```sh
diff -rq \
  ~/git/b2/simbody/upstream/ \
  ~/git/b2/simbody/libsimbody/libsimbody/ \
  --exclude=".gitattributes" \
  2>/dev/null | grep "^Files"
```

Ignore `Only in upstream/` lines (CMake build system, docs, tests -- absent in
the build2 port by design) and `Only in libsimbody/` lines that end in `.orig`
(build2 patching artefacts).

---

## Patches to extract

Eight source files differ, grouped into five logical patches:

### 01 -- constexpr NTraits getters and Scalar constants (SIOF fix)

**Files:**
- `SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h`
- `SimTKcommon/Scalar/src/Scalar.cpp`

**What:** Makes all `NTraits<T>::get*()` functions `constexpr` (where the
underlying `std::numeric_limits<T>` function is `constexpr`, i.e. everything
except `sqrt`/`pow`-derived values, which are guarded by
`__cpp_lib_constexpr_cmath` for C++23). Changes the definitions of
`SimTK::NaN`, `SimTK::Infinity`, `SimTK::Pi`, `SimTK::Zero`, `SimTK::One`,
and all other numeric constants in `Scalar.cpp` from `const` to `constexpr`.

**Why:** The existing definitions use `NTraits<Real>::getNaN()` etc. which
return function-local statics. That makes `SimTK::NaN` and friends
*dynamically initialized*, meaning they are not guaranteed to have their
correct values before another translation unit's static initializers run. In a
static-library build this causes a static-initialization-order fiasco (SIOF):
`RegisterTypes_osim*` (opensim-core) constructs default component instances
before `Scalar.cpp`'s initializers have run, so properties that default to
`SimTK::NaN` silently become 0. Making the getters `constexpr` gives the
constants *constant initialization*, which the standard guarantees runs before
any dynamic initialization.

**Note:** Once this patch is merged into upstream Simbody, the twelve
opensim-core call-site workaround patches (38-49) become unnecessary. See
`~/git/b2/opensim/patches/PATCHES.md` for details.

---

### 02 -- constexpr DecorativeGeometry color constants (SIOF fix)

**Files:**
- `SimTKcommon/Geometry/src/DecorativeGeometry.cpp`

**What:** Changes the eleven file-scope `const Vec3` color constants
(`Black`, `Gray`, `Red`, etc.) to `constexpr Vec3`.

**Why:** Same SIOF pattern as patch 01. `Vec3` is an aggregate of `double`
values initialized from literals, so `constexpr` is valid and gives constant
initialization.

---

### 03 -- constexpr CoordinateAxis constructors

**Files:**
- `SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h`

**What:** Marks the three `CoordinateAxis(XTypeAxis/YTypeAxis/ZTypeAxis)`
constructors and several related constructors in `XCoordinateAxis`,
`YCoordinateAxis`, `ZCoordinateAxis` as `constexpr`. Also trims trailing
whitespace.

**Why:** The constructors initialize only a single integer member from a
compile-time constant. Making them `constexpr` allows `CoordinateAxis` values
to be used in constant expressions, which is a prerequisite for making
downstream types (e.g. `Vec`) fully `constexpr`.

---

### 04 -- constexpr Vec element-list constructors

**Files:**
- `SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h`

**What:** Marks the `Vec<M,E>` constructors that take two through seven
explicit element arguments as `constexpr`. The body changes from
`assert(M==N); (*this)[i]=ei;` to a delegating `constexpr` form.

**Why:** `Vec<3,double>` is the type of the color constants in patch 02 and of
spatial vectors throughout SimTK. `constexpr` constructors allow `Vec` values
to be constant-initialized.

---

### 05 -- CableSpan out-of-line defaulted move operations

**Files:**
- `Simbody/include/simbody/internal/CableSpan.h`
- `Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp`
- `Simbody/src/CablePath.cpp`

**What:**
- Removes `= default` for `CableSubsystemTestHelper`'s move constructor and
  move-assignment operator from the class definition in `CableSpan.h` and
  provides them explicitly out-of-line in `CableSpan_SubsystemTestHelper_Impl.cpp`.
- Fixes the operand order in one `assert()` in `CablePath.cpp`
  (`assert(mapSurfaceToObstacle.size() == next)` to
  `assert(next == mapSurfaceToObstacle.size())`).

**Why:** MSVC and MinGW both emit warnings (or errors under `-Werror`) when a
`= default` special member is defined inline in a class that is marked for
DLL export, because the compiler cannot guarantee that the defaulted body will
be the same across translation units when the class has non-trivially-movable
members visible only through a forward-declared `Impl*`. Moving the definition
out of the header resolves this cleanly.

---

## Step 1: create the output directories

```sh
cd ~/git/b2/upstream-simbody
mkdir -p patches prs
```

---

## Step 2: generate patch files

For each patch, diff the relevant files between `upstream/` and
`libsimbody/libsimbody/` and write the result to a numbered file:

```sh
UP=~/git/b2/simbody/upstream
LIB=~/git/b2/simbody/libsimbody/libsimbody
OUT=~/git/b2/upstream-simbody/patches

# diff exits 1 when files differ (always true here) -- the "|| true" prevents
# the shell from aborting. The --label flags produce a/b-prefixed paths so
# "git apply" works with its default -p1 strip level.

# 01
diff -u \
  --label a/SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h \
  --label b/SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h \
  $UP/SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h \
  $LIB/SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h \
  > $OUT/01-constexpr-ntraits.patch || true
diff -u \
  --label a/SimTKcommon/Scalar/src/Scalar.cpp \
  --label b/SimTKcommon/Scalar/src/Scalar.cpp \
  $UP/SimTKcommon/Scalar/src/Scalar.cpp \
  $LIB/SimTKcommon/Scalar/src/Scalar.cpp \
  >> $OUT/01-constexpr-ntraits.patch || true

# 02
diff -u \
  --label a/SimTKcommon/Geometry/src/DecorativeGeometry.cpp \
  --label b/SimTKcommon/Geometry/src/DecorativeGeometry.cpp \
  $UP/SimTKcommon/Geometry/src/DecorativeGeometry.cpp \
  $LIB/SimTKcommon/Geometry/src/DecorativeGeometry.cpp \
  > $OUT/02-constexpr-color-constants.patch || true

# 03
diff -u \
  --label a/SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h \
  --label b/SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h \
  $UP/SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h \
  $LIB/SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h \
  > $OUT/03-constexpr-coordinate-axis.patch || true

# 04
diff -u \
  --label a/SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h \
  --label b/SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h \
  $UP/SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h \
  $LIB/SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h \
  > $OUT/04-constexpr-vec-constructors.patch || true

# 05
diff -u \
  --label a/Simbody/include/simbody/internal/CableSpan.h \
  --label b/Simbody/include/simbody/internal/CableSpan.h \
  $UP/Simbody/include/simbody/internal/CableSpan.h \
  $LIB/Simbody/include/simbody/internal/CableSpan.h \
  > $OUT/05-cable-span-out-of-line-defaults.patch || true
diff -u \
  --label a/Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp \
  --label b/Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp \
  $UP/Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp \
  $LIB/Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp \
  >> $OUT/05-cable-span-out-of-line-defaults.patch || true
diff -u \
  --label a/Simbody/src/CablePath.cpp \
  --label b/Simbody/src/CablePath.cpp \
  $UP/Simbody/src/CablePath.cpp \
  $LIB/Simbody/src/CablePath.cpp \
  >> $OUT/05-cable-span-out-of-line-defaults.patch || true
```

Verify each file is non-empty and contains only the expected files before
moving on.

---

## Step 3: create PATCHES.md

Create `patches/PATCHES.md` in this repo, following the same structure as
`~/git/b2/opensim/patches/PATCHES.md`: one section per logical group with a
bold patch filename, a prose description of what it changes and why, and a
suggested PR grouping table at the bottom.

Use the descriptions in the "Patches to extract" section above as the basis
for each entry.

Suggested PR grouping:

1. **Constexpr numeric constants (SIOF fix)** -- 01, 02 (can be one PR since
   both address the same root cause and are small)
2. **Constexpr SimTK value types** -- 03, 04 (prerequisite: 01 for Vec since
   Vec constructors call NTraits getters)
3. **CableSpan DLL export compatibility** -- 05

---

## Step 4: create PR description files

Create one file per PR under `prs/` in this repo, following the same format
as `~/git/b2/opensim/prs/`:

```
Title: [portability] <one-line description for GitHub title field>

---

> **Series note:** This PR is one of a set of portability and correctness
> fixes extracted from a build2 port of Simbody. All changes fix genuine
> issues in the upstream codebase and none is build2-specific. Each PR is
> self-contained and can be reviewed and merged independently.

## What
...

## Why
...

## How to test

Build Simbody from source and run its test suite:

    cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
    cmake --build build -j4
    ctest --test-dir build -C Release --output-on-failure

All tests should pass. The static-init fix (PR 1) can additionally be
verified by building opensim-core in static mode against this Simbody --
see the integration test in Step 7 for the full procedure.
```

The title is a standalone line for copy-pasting into GitHub's title field and
is not part of the PR body. Do not use semicolons in prose. See
`~/git/b2/opensim/prs/` for the established style.

Suggested filenames:

| File | Covers |
|---|---|
| `prs/pr-constexpr-siof.md` | patches 01 + 02 |
| `prs/pr-constexpr-value-types.md` | patches 03 + 04 |
| `prs/pr-cable-span-dll-compat.md` | patch 05 |

For `pr-constexpr-siof.md` add a note at the end explaining that merging this
patch into Simbody makes the twelve opensim-core call-site workaround patches
(38-49) unnecessary, and that the opensim-core static-build PR for those
patches should reference this Simbody PR as its preferred alternative.

Commit the documentation to the current branch (`temp-patching`) after all
three files are written:

```sh
cd ~/git/b2/upstream-simbody
git add patches/ prs/
git commit -m "Add patch files, PATCHES.md, and PR descriptions"
```

---

## Step 5: create patch/ branches in the Simbody fork

Each branch is a single commit on top of `master`. Run all five blocks in
sequence:

```sh
cd ~/git/b2/upstream-simbody
LIB=~/git/b2/simbody/libsimbody/libsimbody

# patch/constexpr-ntraits-scalar
git checkout master
git checkout -b patch/constexpr-ntraits-scalar
cp $LIB/SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h \
   SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h
cp $LIB/SimTKcommon/Scalar/src/Scalar.cpp \
   SimTKcommon/Scalar/src/Scalar.cpp
git add SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h \
        SimTKcommon/Scalar/src/Scalar.cpp
git commit -m "Make NTraits getters and Scalar constants constexpr to fix static init order"

# patch/constexpr-color-constants
git checkout master
git checkout -b patch/constexpr-color-constants
cp $LIB/SimTKcommon/Geometry/src/DecorativeGeometry.cpp \
   SimTKcommon/Geometry/src/DecorativeGeometry.cpp
git add SimTKcommon/Geometry/src/DecorativeGeometry.cpp
git commit -m "Make DecorativeGeometry color constants constexpr"

# patch/constexpr-coordinate-axis
git checkout master
git checkout -b patch/constexpr-coordinate-axis
cp $LIB/SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h \
   SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h
git add SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h
git commit -m "Mark CoordinateAxis constructors constexpr"

# patch/constexpr-vec-constructors
git checkout master
git checkout -b patch/constexpr-vec-constructors
cp $LIB/SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h \
   SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h
git add SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h
git commit -m "Mark Vec element-list constructors constexpr"

# patch/cable-span-out-of-line-defaults
git checkout master
git checkout -b patch/cable-span-out-of-line-defaults
cp $LIB/Simbody/include/simbody/internal/CableSpan.h \
   Simbody/include/simbody/internal/CableSpan.h
cp $LIB/Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp \
   Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp
cp $LIB/Simbody/src/CablePath.cpp \
   Simbody/src/CablePath.cpp
git add Simbody/include/simbody/internal/CableSpan.h \
        Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp \
        Simbody/src/CablePath.cpp
git commit -m "Move CableSubsystemTestHelper defaulted moves out of line"
```

After all five branches are created, return to the documentation branch:

```sh
git checkout temp-patching
```

---

## Build scripts (Windows)

No separate deps build is needed: Simbody bundles its own BLAS/LAPACK
(`WINDOWS_USE_EXTERNAL_LIBS=OFF` by default). Place the four scripts below in
`/tmp/claude/` (not in the repo) before running verification in Step 6.

`/tmp/claude/build-simbody-shared.bat`:
```bat
@echo off
call "C:/Program Files/Microsoft Visual Studio/18/Enterprise/VC/Auxiliary/Build/vcvars64.bat"
cmake -S "C:/Users/fho/git/b2/upstream-simbody" ^
  -B "C:/msys64/tmp/claude/simbody-build-shared" ^
  -G Ninja -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_INSTALL_PREFIX="C:/msys64/tmp/claude/simbody-install-shared" ^
  -DSIMBODY_BUILD_SHARED_LIBS=ON ^
  -DBUILD_VISUALIZER=OFF ^
  -DBUILD_EXAMPLES=OFF ^
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL
if errorlevel 1 exit /b 1
cmake --build "C:/msys64/tmp/claude/simbody-build-shared" -j4
```

`/tmp/claude/test-simbody-shared.bat`:
```bat
@echo off
call "C:/Program Files/Microsoft Visual Studio/18/Enterprise/VC/Auxiliary/Build/vcvars64.bat"
ctest --test-dir "C:/msys64/tmp/claude/simbody-build-shared" -C Release --output-on-failure -j4
```

`/tmp/claude/build-simbody-static.bat`:
```bat
@echo off
call "C:/Program Files/Microsoft Visual Studio/18/Enterprise/VC/Auxiliary/Build/vcvars64.bat"
cmake -S "C:/Users/fho/git/b2/upstream-simbody" ^
  -B "C:/msys64/tmp/claude/simbody-build-static" ^
  -G Ninja -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_INSTALL_PREFIX="C:/msys64/tmp/claude/simbody-install-static" ^
  -DSIMBODY_BUILD_SHARED_LIBS=OFF ^
  -DBUILD_VISUALIZER=OFF ^
  -DBUILD_EXAMPLES=OFF ^
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL
if errorlevel 1 exit /b 1
cmake --build "C:/msys64/tmp/claude/simbody-build-static" -j4
```

`/tmp/claude/test-simbody-static.bat`:
```bat
@echo off
call "C:/Program Files/Microsoft Visual Studio/18/Enterprise/VC/Auxiliary/Build/vcvars64.bat"
ctest --test-dir "C:/msys64/tmp/claude/simbody-build-static" -C Release --output-on-failure -j4
```

On Linux, call cmake and ctest directly without vcvars and without
`-DCMAKE_MSVC_RUNTIME_LIBRARY`.

---

## Step 6: verify patches apply and build cleanly

First establish a baseline on `master`:

```sh
cd ~/git/b2/upstream-simbody
git checkout master
```

Run the shared build and tests (confirm the repo is in a known-good state):

```
cmd //c C:/msys64/tmp/claude/build-simbody-shared.bat
cmd //c C:/msys64/tmp/claude/test-simbody-shared.bat
```

Confirm each patch file applies cleanly:

```sh
git apply --check patches/01-constexpr-ntraits.patch
git apply --check patches/02-constexpr-color-constants.patch
git apply --check patches/03-constexpr-coordinate-axis.patch
git apply --check patches/04-constexpr-vec-constructors.patch
git apply --check patches/05-cable-span-out-of-line-defaults.patch
```

For each patch branch, also run the static build and tests (the mode that
exercises the constexpr fixes):

```sh
git checkout patch/constexpr-ntraits-scalar
cmd //c C:/msys64/tmp/claude/build-simbody-static.bat
cmd //c C:/msys64/tmp/claude/test-simbody-static.bat
git checkout master
# repeat for remaining patch branches
```

All tests should pass on both shared and static builds.

---

## Step 7: integration test

To verify that the Simbody patches make the opensim-core static build pass
without the opensim-core call-site workaround patches (38-49):

1. Copy the patched `NTraits.h` and `Scalar.cpp` into the Simbody source used
   by the opensim-core superbuild:

   ```sh
   SIMBY=~/git/b2/fork-opensim/dependencies/simbody
   cp $LIB/SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h \
      $SIMBY/SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h
   cp $LIB/SimTKcommon/Scalar/src/Scalar.cpp \
      $SIMBY/SimTKcommon/Scalar/src/Scalar.cpp
   ```

2. Force a Simbody rebuild (PowerShell on Windows, timestamps must advance):

   ```powershell
   $now = Get-Date
   (Get-Item "$SIMBY\SimTKcommon\Scalar\src\Scalar.cpp").LastWriteTime = $now
   (Get-Item "$SIMBY\SimTKcommon\Scalar\include\SimTKcommon\internal\NTraits.h").LastWriteTime = $now
   ```

   Then run `/tmp/claude/build-deps.bat` (or the equivalent Linux cmake
   command) to rebuild and reinstall Simbody.

3. In `~/git/b2/fork-opensim`, create a test branch with all opensim-core
   `patch/` branches merged except `patch/static-simtk-siof`, build static,
   and run tests. The expected result is 100% pass (92/92 run, 8 disabled),
   identical to the full patch set. This was already verified locally -- see
   the note in `~/git/b2/opensim/patches/PATCHES.md` under "Static build:
   SimTK SIOF".

---

## PR submission order

Open the Simbody PRs before the opensim-core static-build PR. In the
opensim-core PR for `patch/static-simtk-siof` (patches 38-49), add a note
that the preferred fix is the upstream Simbody change and link the Simbody PR.
If the Simbody PR is merged first, the opensim-core patch can simply be
dropped and the PR not opened.
