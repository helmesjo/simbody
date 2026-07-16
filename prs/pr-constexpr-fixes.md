Title: [portability] Make NTraits/Scalar/Vec/CoordinateAxis constexpr to fix static init order

---

> **Note:** These changes were extracted from a build2 port of Simbody. All fix
> genuine issues in the upstream codebase and none is build2-specific.

## What

Adds `constexpr` to four areas of the codebase:

**NTraits.h / Scalar.cpp**
- Makes all `NTraits<T>::get*()` functions `constexpr` where the underlying
  `std::numeric_limits<T>` function is itself `constexpr`. The three
  `sqrt`/`pow`-derived getters (`getSignificant`, `getSqrtEps`, `getTiny`) are
  conditionalized on `__cpp_lib_constexpr_cmath` (C++23) via helper macros and
  fall back to function-local statics on older standards.
- Changes the definitions of `SimTK::NaN`, `SimTK::Infinity`, `SimTK::Eps`,
  `SimTK::Pi`, `SimTK::Zero`, `SimTK::One`, and all other numeric constants in
  `Scalar.cpp` from `const` to `constexpr` (the three `sqrt`/`pow`-derived
  values are conditionalized the same way).

**DecorativeGeometry.cpp**
- Changes the eleven file-scope `const Vec3` color constants (`Black`, `Gray`,
  `Red`, `Green`, `Blue`, `Yellow`, `Orange`, `Magenta`, `Purple`, `Cyan`,
  `White`) to `constexpr Vec3`.

**CoordinateAxis.h**
- Marks the `CoordinateAxis(XTypeAxis)`, `CoordinateAxis(YTypeAxis)`, and
  `CoordinateAxis(ZTypeAxis)` constructors and the corresponding constructors in
  `XCoordinateAxis`, `YCoordinateAxis`, and `ZCoordinateAxis` as `constexpr`.

**Vec.h**
- Marks the eight `Vec<M,E>` constructors that take two through nine explicit
  element arguments as `constexpr`. The constructor body is changed from
  `assert(M==N); (*this)[i]=ei;` to direct array writes `d[i*STRIDE]=ei`
  (arity is enforced by the C++ type system, so the runtime assert is
  redundant).

## Why

`SimTK::NaN`, `SimTK::Infinity`, and the other `Scalar.cpp` constants are
defined as `extern const Real` with dynamic initialization: each calls a
`NTraits<Real>` getter that returns a function-local static. Dynamic
initialization is not ordered across translation units. In a static-library
build, `RegisterTypes_osim*` translation units in opensim-core construct default
component instances (via `constructProperties()`) before `Scalar.cpp`'s
initializers run, so any property that defaults to `SimTK::NaN` or
`SimTK::Infinity` silently reads as zero. The same problem affects `Vec3` color
constants in `DecorativeGeometry.cpp` that depend on dynamic initialization.

Making the `NTraits` getters `constexpr` allows the downstream `Scalar.cpp`
constants and `DecorativeGeometry.cpp` color constants to be given constant
initialization, which the C++ standard guarantees runs before any dynamic
initialization regardless of link order. Marking the `CoordinateAxis` and `Vec`
constructors `constexpr` is a prerequisite for `constexpr Vec3` aggregates to
be valid.

The changes have no observable effect in shared-library builds, where
initialization order is well-defined.

## How to test

Build Simbody from source and run its test suite:

    cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
    cmake --build build -j4
    ctest --test-dir build -C Release --output-on-failure

All tests should pass. To exercise the static-initialization fix specifically,
build with `-DSIMBODY_BUILD_SHARED_LIBS=OFF` and repeat. Then build opensim-core
as a static library against this patched Simbody and confirm that no component
property defaults are silently zero.

---

**Note for the opensim-core static-build PR:** Once this patch is merged into
upstream Simbody, the twelve opensim-core call-site workaround patches (patches
38-49 in the build2 opensim port, tracked in the PR
`patch/static-simtk-siof`) become unnecessary. The preferred fix is this
upstream Simbody change. If this Simbody PR is merged first, the opensim-core
workaround PR can be dropped entirely.
