# Developer guide for tatami libraries

## Overview

This document contains some guidelines for developing **tatami** libraries.
I made this up as I went along so not all libraries are fully compliant, though I try my best to update things to meet these (shifting) standards.

## General naming

Types should be named with `PascalCase`.

Variables and functions should be named with `snake_case`.

Template parameters should have a trailing underscore to distinguish them from non-template parameters.
This occurs regardless of whether a template parameter is a type.

## `class` vs `struct`

Classes should not have any public members.
Access to members should be granted via getters or setters.
All members should be prefixed by `my_` to avoid confusion with arguments in the constructor and initialization list.

Structs should have public members only, with no private members or methods.
These are treated as passive data carriers with no additional functionality.
A non-default constructor is allowed, though.

(Some of these ideas were taken from Google's C++ style guide.)

## Accessing two-dimensional arrays

### Overflow from integer products

Two-dimensional accesses typically involve the following pattern, e.g., for row-major matrices:

```cpp
ptr[r * nc + c]; // gets matrix entry (r, c)
```

If the integer type (typically promoted to `int`) is not large enough to hold the product, our code will be incorrect due to the overflow.
Developers should coerce all operands to `size_t` before any arithmetic to ensure that no overflow occurs. 

### Invalid pointer construction

Merely constructing (not dereferencing!) of a pointer beyond one-past-the-end of an array is undefined behavior.
Such a pointer can be inadvertently constructed when performing strided access in the middle of the matrix.
For example, given a row-major matrix:

```cpp
int col = 5;
auto adj_ptr = ptr + col;
for (int r = 0; r < nr; ++r) {
    row_contents[r] = *adj_ptr;
    adj_ptr += nc;
}
```

After the last iteration, `adj_ptr` will be equal to `ptr + nr * nc + col`, which is not valid.
Developers should instead use offsets in the loop body to ensure that an invalid pointer is never constructed.

```cpp
int col = 5;
size_t offset = col; // Make sure it's size_t to avoid overflow!
for (int r = 0; r < nr; ++r) {
    row_contents[r] = ptr[offset];
    offset += nc;
}
```

An even better approach is to just compute the 2-dimensional index directly in the loop body.
Modern compilers (well, Clang and GCC, at least) can optimize out the multiplication so there is no performance penalty.
This approach is easier to reason about and is more amenable to vectorization as there are no dependencies in the loop body.

```cpp
// Make sure it's all size_t to avoid overflow!
size_t col = 5;
size_t long_nc = nc, long_nr = nr;
for (size_t r = 0; r < long_nr; ++r) {
    row_contents[r] = ptr[col + r * long_nc];
}
```

## Parallelization and floating-point calculations

Historically, floating-point calculations were parallelized in a manner that ensures that the number of workers does not affect the result.
This requires some care in designing algorithms to ensure the exact same result was obtained during floating-point calculations.
The aim was to allow users to freely alter the number of threads without compromising reproducibility.
This is most relevant when dealing with downstream algorithms based on ranks, where slight floating-point errors can lead to discrete changes in rank.

With the benefit of hindsight, this policy is probably unnecessary.

- Most users will not change the number of threads after their analysis,
  so they won't even know that there is a difference due to the number of threads.
- The main reason to change the number of threads is to account for differences in hardware.
  At that point, it is likely that a different CPU is involved, at which point exact floating-point reproducibility is no longer guaranteed.
- This policy is inconsistent with the acceptance of different results between matrix representations due to sparsity or row/column preferences.
  The obvious candidate is that of the per-row/column variances, and no one has complained so far. 

All future algorithmic choices in **tatami** can disregard this policy.
This allows us to use more efficient methods for partitioning work between cores. 
Note that the differences in results due to parallelization should be limited to floating-point error, i.e., the result should be the same under exact math.

## SIMD vectorization

Most of the loops in **tatami** are trivially auto-vectorizable so no further work is really necessary.
In other cases where non-aliasing cannot be proven, I find it useful to decorate loops with the 
[`SUBPAR_VECTORIZABLE`](https://github.com/LTLA/subpar?tab=readme-ov-file#encouraging-vectorization) macro.
This assures the compiler that pointers do not alias and thus vectorization can be performed.
Even in obviously vectorizable loops, this macro can still be useful by eliminating the need for aliasing checks.

Another approach for encouraging auto-vectorization is to use OpenMP SIMD.
This is slightly more portable in that compilers should support it off the bat,
rather than requiring me to update **subpar** with new compiler-specific definitions for `SUBPAR_VECTORIZABLE`.
However, it is often subject to a more heavy-handed interpretation by compilers.
Upon seeing `#pragma omp simd`, [GCC](https://developers.redhat.com/articles/2023/12/08/vectorization-optimization-gcc) will forcibly vectorize the loop,
even if doing so would decrease performance according to its cost model.
[MSVC](https://devblogs.microsoft.com/cppblog/simd-extension-to-c-openmp-in-visual-studio/) goes further and enables fast floating-point inside the loop, which is not generally desirable..
In the end, I wanted a lighter touch that would respect the compiler's knowledge about the target system.

Yet another choice is to just use a third-party library like **highway** or **Eigen**.
I don't want to do this as it violates my general policy against dependencies and I don't think the benefits outweight the cost here.
The experimental `<simd>` standard might be an option in the future, but then I'd have to write all the code to handle the looping, I'd prefer that the compiler just figure that out itself.
