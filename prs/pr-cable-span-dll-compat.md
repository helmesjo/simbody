Title: [portability] Move CableSubsystemTestHelper defaulted move operations out of line

---

> **Series note:** This PR is one of a set of portability and correctness fixes
> extracted from a build2 port of Simbody. All changes fix genuine issues in the
> upstream codebase and none is build2-specific. Each PR is self-contained and
> can be reviewed and merged independently.

## What

In `CableSpan.h`, removes `= default` from the inline declarations of
`CableSubsystemTestHelper`'s move constructor and move-assignment operator and
replaces them with plain declarations:

    CableSubsystemTestHelper(CableSubsystemTestHelper&&) noexcept;
    CableSubsystemTestHelper& operator=(CableSubsystemTestHelper&&) noexcept;

The definitions (still `= default`) are added out-of-line in
`CableSpan_SubsystemTestHelper_Impl.cpp`.

Also fixes the operand order in one `assert()` in `CablePath.cpp`:

    assert(mapSurfaceToObstacle.size() == next);

becomes:

    assert(next == mapSurfaceToObstacle.size());

This places the unsigned value (`size_t`) on the right and the signed loop
counter on the left, matching the conventional form and silencing the
signed/unsigned comparison warning.

## Why

MSVC and MinGW emit warnings (and errors under `-Werror`) when a `= default`
special member function is defined inline inside a class that is annotated for
DLL export (`SimTK_SIMBODY_EXPORT`). The compiler cannot guarantee that the
defaulted body is identical across translation units when the class has
non-trivially-movable members visible only through a forward-declared `Impl*`
pointer. Moving the definition out of the header eliminates this warning cleanly
without changing behavior.

## How to test

Build Simbody from source and run its test suite:

    cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
    cmake --build build -j4
    ctest --test-dir build -C Release --output-on-failure

On Windows with MSVC or MinGW, confirm that no warnings are emitted for
`CableSubsystemTestHelper`'s move operations. All tests should pass.
