# Simbody upstream patches

## Context

These patches were extracted from a build2 port of Simbody
(https://github.com/simbody/simbody). Each patch fixes a real upstream issue.
None is build2-specific. The goal is to apply each patch (or a logical group)
to a fresh branch of this fork and open a pull request.

The upstream repo is at: https://github.com/simbody/simbody

## How to apply a patch

From the repo root:

    git apply patches/NN-Name.patch

Patches 01-04 apply cleanly to HEAD. Patch 05 does not apply cleanly because
its changes are already present in master (see the patch 05 entry below).

## PR status

| Patch | Status |
|---|---|
| 01-04 (constexpr fixes) | Not yet submitted |
| 05 (CableSpan) | Already in master -- no PR needed |

---

## Patch inventory

### Constexpr fixes (static-initialization-order fiasco)

**01-constexpr-ntraits.patch**
Makes all `NTraits<T>::get*()` functions `constexpr` where the underlying
`std::numeric_limits<T>` function is itself `constexpr` (everything except
`sqrt`/`pow`-derived values, which are guarded by `__cpp_lib_constexpr_cmath`
for C++23). The `imag()` function, which previously returned `getZero()`, is
updated to use a local static directly so it does not depend on the `constexpr`
getter. In `Scalar.cpp`, changes the definitions of `SimTK::NaN`,
`SimTK::Infinity`, `SimTK::Eps`, `SimTK::Pi`, `SimTK::Zero`, `SimTK::One`,
and all other numeric constants from `const` to `constexpr` (those backed by
`sqrt`/`pow` are conditionalized on `__cpp_lib_constexpr_cmath`).

The existing definitions use `NTraits<Real>::getNaN()` etc., which return
function-local statics. That makes `SimTK::NaN` and friends dynamically
initialized, meaning they are not guaranteed to have their correct values before
another translation unit's static initializers run. In a static-library build
this causes a static-initialization-order fiasco (SIOF): `RegisterTypes_osim*`
translation units in opensim-core construct default component instances before
`Scalar.cpp`'s initializers have run, so properties that default to `SimTK::NaN`
silently read as 0. Making the getters `constexpr` gives the constants constant
initialization, which the standard guarantees runs before any dynamic
initialization.

Files: `SimTKcommon/Scalar/include/SimTKcommon/internal/NTraits.h`,
`SimTKcommon/Scalar/src/Scalar.cpp`

**02-constexpr-color-constants.patch**
Changes the eleven file-scope `const Vec3` color constants in
`DecorativeGeometry.cpp` (`Black`, `Gray`, `Red`, `Green`, `Blue`, `Yellow`,
`Orange`, `Magenta`, `Cyan`, `White`, `DarkBrown`) to `constexpr Vec3`.
`Vec3` is an aggregate of `double` values initialized from literals, so
`constexpr` is valid once the `Vec` constructors in patch 04 are marked
`constexpr`. Same SIOF root cause as patch 01.

File: `SimTKcommon/Geometry/src/DecorativeGeometry.cpp`

**03-constexpr-coordinate-axis.patch**
Marks the three `CoordinateAxis(XTypeAxis/YTypeAxis/ZTypeAxis)` constructors
and several related constructors in `XCoordinateAxis`, `YCoordinateAxis`, and
`ZCoordinateAxis` as `constexpr`. Also removes trailing whitespace on affected
lines. The constructors initialize a single integer member from a compile-time
constant, making the `constexpr` annotation straightforward. This allows
`CoordinateAxis` values to participate in constant expressions, which is a
prerequisite for making downstream aggregate types fully `constexpr`.

File: `SimTKcommon/Mechanics/include/SimTKcommon/internal/CoordinateAxis.h`

**04-constexpr-vec-constructors.patch**
Marks the eight `Vec<M,E>` constructors that take two through nine explicit
element arguments as `constexpr`. The body is changed from the runtime-checked
form `assert(M==N); (*this)[i]=ei;` (which uses `operator[]` and would be
non-constexpr) to direct array writes `d[i*STRIDE]=ei`. The runtime `assert`
is dropped because arity is enforced by the C++ type system -- each overload is
instantiated only when M equals the overload's argument count.

File: `SimTKcommon/SmallMatrix/include/SimTKcommon/internal/Vec.h`

---

### CableSpan DLL-export compatibility

**05-cable-span-out-of-line-defaults.patch** *(already in master -- no PR needed)*
Removes `= default` for `CableSubsystemTestHelper`'s move constructor and
move-assignment operator from the class definition in `CableSpan.h` and
provides them explicitly out-of-line (as `= default`) in
`CableSpan_SubsystemTestHelper_Impl.cpp`. Also fixes the operand order in one
`assert()` in `CablePath.cpp`: `assert(mapSurfaceToObstacle.size() == next)`
becomes `assert(next == mapSurfaceToObstacle.size())`, removing a
signed/unsigned comparison warning.

MSVC and MinGW emit warnings (or errors under `-Werror`) when a `= default`
special member is defined inline in a class annotated for DLL export, because
the compiler cannot guarantee that the defaulted body is identical across
translation units when the class has members visible only through a
forward-declared `Impl*`. Moving the definition out of the header resolves this.

**Status:** Both changes are already applied in this fork's `master` branch.
The out-of-line move operations appear in `CableSpan_SubsystemTestHelper_Impl.cpp`
(lines 601 and 604 as of the time of writing). The `CablePath.cpp` assert was
fixed via a cast approach in commit `9a960545` ("Disambiguate comparison in
CablePath assert"). The patch file is retained as a historical record of what
the build2 port changed.

Files: `Simbody/include/simbody/internal/CableSpan.h`,
`Simbody/src/CableSpan_SubsystemTestHelper_Impl.cpp`,
`Simbody/src/CablePath.cpp`

---

## Suggested PR grouping

| PR | Patches | Branch(es) |
|---|---|---|
| constexpr fixes (SIOF + value types) | 01, 02, 03, 04 | `patch/constexpr-ntraits-scalar`, `patch/constexpr-color-constants`, `patch/constexpr-coordinate-axis`, `patch/constexpr-vec-constructors` |
| CableSpan DLL-export compatibility | 05 | already in master, no branch needed |
