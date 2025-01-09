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

## Function arguments

Functions should not have default arguments.
Instead, functions should accept an `Options` struct that contains all arguments with their default values.
This avoids confusion when a caller needs to specify a subset of those arguments or if we need to add more options.

Similarly, template functions should refrain from adding defaults for their template arguments.
The vast majority of template arguments are deduced at the call site so any defaults are confusing.
The exception is when the output type can be specified by the user, in which case a sensible default may be convenient and self-documenting.

In general, the template arguments should only contain types (or flags that influence the type, e.g., `sparse_`).
It is tempting to move other parameters into the template arguments to avoid run-time checks for efficiency, but this should be used sparingly.
Libraries typically need to be compiled to support all possibilities so such arguments will change at run time anyway.

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

## Zeroing arrays 

Some functions store their output in a user-supplied pointer to a pre-allocated array.
Such functions should not assume that the array has been zeroed.
Incorrectly making this assumption can lead to subtle bugs when the functions are called with non-zeroed buffers.

To ensure that this assumption is not present in function `X()`, my test suites fill the output buffer with a non-zero initial value from the `initial_value()` function below.
Upon calling `X()` with `output.data()`, we can compare `output` against the expected output to verify that the function is non-zero initial value is ignored.
If the expected output is not readily available, we could instead initialize a different vector with another call to `initial_value()` and call `X()` on it.
Any inconsistency in the results would indicate that the variable `initial_value()` is affecting the results. 

```cpp
inline int initial_value() {
    static int counter = 0;
    if (counter == 255) { // fits in all numeric types.
        counter = 1; 
    } else {
        ++counter;
    }
    return counter;
}

// Initializing the output buffer. 
std::vector<int> output(n, initial_value());
```

The real utility of `initial_value()` lies in its ability to be inserted into function overloads that return `std::vector` objects.
This is convenient as simply calling the vector-returning overload is sufficient to test all aspects of the base function, including independence from the initial value of the input buffer.
(Otherwise, the vector-returning overload would always zero-initialize its vector and mask potential dependencies on the input buffer in the base function.)

```cpp
// Defined for test binaries only.
#define TEST_INIT_VALUE initial_value()

// "Base" function that accepts a pointer. 
void X(int n, double* output);

// Overload that returns a std::vector.
std::vector<double> X(int n) {
    std::vector<double> output(n
#ifdef TEST_INIT_VALUE
        , TEST_INIT_VALUE
#endif
    );
    X(n, output.data());
    return output;
}
```

## Parallelization

### Floating-point calculations

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

### False sharing

False sharing can be mostly avoided by writing code in a manner that uses automatic variables allocated on each thread's stack (which would be good practice regardless).
However, the heap is still shared across threads and it is likely that each thread will need to write to the heap at some point.
False sharing can be mitigated by allocating to the heap within each thread, which gives `malloc` a chance to use different arenas, depending on the implementation.
(Guaranteed protection against false sharing requires alignment to `hardware_destructive_interference_size`, which seems too intrusive for general use.) 
Writing to contiguous memory in the heap across multiple threads should be done sparingly, typically to the output buffer once all calculations are complete.

An interesting sidenote is that each thread will typically create its own `tatami::Extractor` instance on the heap.
Modifications to data members of each `tatami::Extractor` instance during `fetch()` calls could be another source of false sharing.

## SIMD vectorization

Ah, vectorization: the policy with the most flip-flopping in this guide.

Most of the loops in **tatami** are trivially auto-vectorizable so no further work is necessary when compiled with the relevant flags, e.g., `-O3`, `-march=`.
There might be a slight inefficiency due to the need to check against aliasing pointers, but this is fine;
the check is performed outside of the loop and should not have much impact on performance.
On the rare occasion that the check is run frequently relative to the loop body, branch prediction should eliminate most of the cost for the expected case of non-aliasing pointers.
Auto-vectorization also tends to use unaligned SIMD loads and stores, but it's hard to imagine doing otherwise in library code that might take pointers from anywhere (e.g., R).
Fortunately, it seems like unaligned instructions are pretty performant on modern CPUs,
see commentary [here](https://stackoverflow.com/questions/52147378/choice-between-aligned-vs-unaligned-x86-simd-instructions)
and [here](https://stackoverflow.com/questions/20259694/is-the-sse-unaligned-load-intrinsic-any-slower-than-the-aligned-load-intrinsic-o).

That said, there are two common scenarios where auto-vectorization is not possible.

1. Accessing a sparse vector by index.
   If the compiler cannot prove that the indices are unique, different loop iterations are not guaranteed to be independent. 
   Here, I went through many possibilities to inform the compiler that vectorization can be performed.
   
   - Using OpenMP SIMD to force vectorization. 
     While this works, it is often subject to a more heavy-handed interpretation by compilers.
     Upon seeing `#pragma omp simd`, [GCC](https://developers.redhat.com/articles/2023/12/08/vectorization-optimization-gcc) will forcibly vectorize the loop,
     even if doing so would decrease performance according to its cost model.
     [MSVC](https://devblogs.microsoft.com/cppblog/simd-extension-to-c-openmp-in-visual-studio/) goes further and enables fast floating-point inside the loop, which is not generally desirable.
   - Encouraging vectorization with compiler-specific pragmas to indicate that there are no dependencies between loop iterations,
     e.g., `#pragma GCC ivdep` or `#pragma clang loop vectorize(assume_safety)`.
     This also works fairly well but it's hard to keep track of exactly how each compiler is interpreting these pragmas.
     Are all dependencies ignored? Or just the non-proven ones?
     Needless to say, these pragmas are non-portable and require careful testing on each platform.

   In the end, I suspect that it's not worth the effort to attempt to optimize this access pattern.
   Plenty of excuses here: lack of efficient gather/scatter for many CPUs, likely memory bottlenecks in sparse data, etc.
2. Calling `<cmath>` functions inside the loop, primarily for the delayed math operations.
   Vectorization is precluded by the `errno` side-effect and, for some functions like `std::log`, the need for `-ffast-math` to use vectorized replacements.
   While compiling with `-fno-math-errno` seems [fine](https://stackoverflow.com/questions/7420665/what-does-gccs-ffast-math-actually-do?noredirect=1&lq=1),
   using fast math in general is obviously a less palatable option and I can't cajole GCC into use the (unexported) vectorized `log` function without it.
   So, the solution is to just write a variant of a delayed operation helper that uses a vectorized `log` implementation from an external library like **Eigen**.
   Then, users can choose between the standard helper or the vectorized variant that requires an extra dependency.
