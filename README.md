# Developer guide for tatami libraries

## Overview

This document contains some guidelines for developing **tatami** libraries.
I made this up as I went along so not all libraries are fully compliant, though I try my best to update things to meet these (shifting) standards.

## General naming

Types should be named with `PascalCase`.

Variables and functions should be named with `snake_case`.

Template parameters should have a trailing underscore to distinguish them from non-template parameters.
This occurs regardless of whether a template parameter is a type.

## Classes

### `class` vs `struct`

Classes should not have any public members.
Access to members should be granted via getters or setters.
All members should be prefixed by `my_` to avoid confusion with arguments in the constructor and initialization list.

Structs should have public members only, with no private members or methods.
These are treated as passive data carriers with no additional functionality.
A non-default constructor is allowed, though.

(Some of these ideas were taken from Google's C++ style guide.)

### Templating

Template parameters should only contain types or flags that influence the type, see the logic described [below](#function-templating).

Class templates are more limited than their function counterparts in terms of argument deduction and the positioning of defaults (at least as of C++17).
To avoid an overly frustrating user experience, we allow default arguments for template parameters that could have been deduced from a constructor.
This is a rather reluctant compromise, as the presence of defaults increases the risk of strange type mismatch errors or implicit conversions. 
However, it is more palatable when the default is the base class of all possible input types, where mismatches and conversions are no longer a concern.
This pattern is occasionally used to offer users a choice between run-time and compile-time polymorphism.

```cpp
template<typename Thing_ = Base>
class Foo {
public:
    Foo(Thing_ x) {}
};

Foo<> x(Base());
Foo<> y(Derived1());
Foo<> z(Derived2());
Foo<Derived2> z(Derived2()); // potential devirtualization.
```

## Functions

### Arguments

Functions should not have default arguments.
Instead, functions should accept an `Options` struct that contains all arguments with their default values.
This avoids confusion when a caller needs to specify a subset of those arguments or if we need to add more options.
Users willing to accept all default arguments can simply call the function with `{}`, which is convenient enough that no overload is needed. 

If an argument is optional but has no sensible default, it should be wrapped in a `std::optional`.
This is more explicit than using a sentinel value (e.g., -1 for unsigned integers) and more concise than passing in a separate `bool` for this purpose.

Default values for integer arguments should consider using `sanisizer::cap()` to ensure that the value is below the maximum guaranteed by the standard for that integer type.
For example, a `std::size_t` is only guaranteed to store up to 65535, so setting a greater default value could result in compiler warnings on some unusual implementations.
This scenario is not uncommon for setting buffer sizes where a larger default value is preferred for performance on typical systems but lies beyond the standard-guaranteed upper limit.

### Templating <a id='function-templating'></a>

Template parameters should only contain types or flags that influence the type, e.g., `sparse_`, `oracle_`.
It is tempting to move other parameters into the template arguments to convert run-time checks into compile-time checks for efficiency.
However, this is not ergonomic if those parameters are only known at run-time, as callers must manually write `if` statements to switch between instances of the template function.
I also suspect that this will bloat the binaries, especially if many combinations of arguments must be considered at compile-time.
Thus, non-type-related template parameters should be used sparingly - usually only in performance-critical loops.

Template functions should refrain from adding defaults for their template parameters.
Most template arguments can be deduced at the call site so the corresponding defaults would not be used anyway.
If deduction fails, the presence of defaults can result in confusing errors when the compiler attempts to match the supplied type to the default (or worse, silently perform a conversion).
The obvious exception is that of a template parameter that defines an non-deducible output type, in which case a "sensible" default may be convenient and self-documenting.

In the template parameter list, non-deducible template parameters without defaults should always appear first;
followed by the non-deducible parameters with defaults;
and finally, all deducible parameters, in the order in which they appear in the types of the function arguments.
This approach ensures that the non-deducible parameters can be easily specified by the user. 

### Lambdas

Lambdas should be written in their full form, as shown below:

```
[...] (...) -> return_type {
    /* body */
}
```

For lambdas that have no return value, a `void` return should be explicitly specified.
Similarly, lambdas that accept no arguments should still be written with `()`.
The goal is to more clearly distinguish lambda definitions from IIFEs.

Speaking of which, IIFEs should be written with their "canonical" form:

```
[...]{
    /* body */
}()
```

Developers keep in mind that IIFEs use `auto` type deduction rules for the return type.
This means that they will never return a reference type, which can be unexpected when using an IIFE to replace ternary expressions, e.g., for multi-line expressions or compile-time switches.
In these (relatively rare) cases, we should instead write out a full lambda and invoke it:

```
const auto& x = [...]() -> const auto& {
    /* body */
}();
```

## Using `const` <a id='using-const'></a>

Local variables and function arguments should be declared as `const` unless they need to be modified.
Similarly, class methods should consider being `const` wherever possible.
This reduces the risk of accidental modification, especially from inadvertent references.
Consider:

```cpp
template<typename Bar_>
void foo(Bar_ x) {
    Bar_ y = x;
    ++y;
}
```

If `Bar_` is defined to be a reference, then `foo()` would modify the input value, which is probably not what we intended.
By comparison, setting the function argument to be `const` guarantees that any reference will be a compile-time error.

```cpp
template<typename Bar_>
void foo2(const Bar_ x) {
    Bar_ y = x; // need to modify this, so can't be const. 
    ++y;
}
```

Some might say that it is bad form to expose the implementation detail of `x`'s `const`-ness to the user if we're passing it by value anyway.
Which is a fair point (see discussion [here](https://stackoverflow.com/questions/117293/does-using-const-on-function-parameters-have-any-effect-why-does-it-not-affect)) -
but for header-only libraries, there isn't a clear separation between the function declaration from the definition, so just adding a `const` is the most pragmatic solution here. 
We might be able to achieve this for virtual methods, e.g., the following seems to work:

```cpp
class Base {
public:
    Base() = default;
    virtual ~Base() = default;
    virtual void foo(int) = 0;
};

class Derived : public Base {
public:
    Derived() = default;
    void foo(const int) {}
};
```

## Using `decltype`

One feature of `decltype()` is that it will preserve the reference in the type.
This can be dangerous - for example, the code below will modify `start` by its reference `i`:

```cpp
void loop_body(int& start, int& end) {
    for (decltype(start) i = start; i < end; ++i) { // 'i' is a 'int&' alias for 'start'.
        // Do something
    }
}
```

A simple solution is to define a small copying utility that clears out all qualifiers and references.
This is only "called" within a `decltype()` statement to obtain the desired type:

```cpp
#include <type_traits>

template<typename Input_>
std::remove_cv_t<std::remove_reference_t<Input_> > I(Input_ x) { // i.e., identity().
    return x;
}

// Could also achieve the same effect with std::remove_cvref_t in C++20,
// or even auto to remove references/const/etc. during template deduction.

void loop_body2(int& start, int& end) {
    for (decltype(I(start)) i = start; i < end; ++i) { // 'i' is now an 'int'.
        // Do something
    }
}
```

Given how simple it is to do, I'd suggest just adding `I()` to all `decltype()` statements that should return a non-reference type.
This eliminates any concern about references without manual checking of the `decltype()`'d expression, especially those involving a function call.
It also avoids unintended propagation of `const`-ness when we follow the [previous section's advice](#using-const):

```cpp
void foo3(const int x) {
    decltype(I(x)) y = x; // 'I()' removes constness so that we can modify 'y'.
    ++y;
}
```

Personally, I would prefer to use `auto` over `decltype()` in most situations where a variable is defined.
`auto` doesn't need the `I()` trick to deduce a non-reference, it's shorter to write, and it doesn't require updating upon changes to variable names. 
`decltype()` only needs to be used when the type needs to be more explicit, especially when storing the results of expressions of unknown type.
For example:

```cpp
void loop_body3(const int& start, const int& end) {
    for (auto i = start; i < end; ++i) { // 'i' is an 'int' after deduction.
        // Do something.
    }
}

template<typename Index_>
void loop_body4(const Index_ start, const Index_ length) {
    // start + length is not of a known type after integer promotion,
    // so we force it to be the same as 'start' via decltype().
    for (decltype(I(start)) i = start, end = start + length; i < end; ++i) {
        // Do something.
    }
}
```

## Accessing two-dimensional arrays

### Overflow from integer products

Two-dimensional accesses typically involve the following pattern, e.g., for row-major matrices:

```cpp
ptr[r * nc + c]; // gets matrix entry (r, c)
```

If the integer type (typically promoted to `int`) is not large enough to hold the product, our code will be incorrect due to the overflow.
Developers should coerce all operands to the array's size type before any arithmetic to ensure that no overflow occurs.
This is most easily done with [`sanisizer::nd_offset()`](https://github.com/LTLA/sanisizer):

```cpp
// Equivalent to ptr[c + nc * r] after casting all values to std::size_t.
ptr[sanisizer::nd_offset<std::size_t>(c, nc, r)]; 
```

If we are allocating contiguous memory to hold a high-dimensional array, we should use `sanisizer::product()` to check that the product does not overflow the allocation's size type.
Subsequent accesses into this allocation (e.g., via `sanisizer::nd_offset()` or friends) can omit this check as long as they are referencing valid positions.

```cpp
typedef std::vector<double> Buffer;
Buffer buffer(sanisizer::product<typename Buffer::size_type>(nr, nc));
```

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
A slightly better approach is to use offsets in the loop body to ensure that an invalid pointer is never constructed.

```cpp
int col = 5;
std::size_t offset = col; // Make sure it's size_t to avoid overflow
for (int r = 0; r < nr; ++r) {
    row_contents[r] = ptr[offset];
    offset += nc;
}
```

Unfortunately, the addition in the last iteration could overflow the offset type.
This is technically fine for `size_t` but could be undefined behavior otherwise, even though the offset is not used after the loop.
So, the safest approach is to just compute the 2-dimensional index directly in the loop body.
This ensures that only valid offsets are ever computed, assuming that the loop stops at the end of the matrix.

```cpp
int col = 5;
for (int r = 0; r < nr; ++r) {
    // Compute offset as 'col + nc * r' inside the loop.
    row_contents[r] = ptr[sanisizer::nd_offset<std::size_t>(col, nc, r)];
}
```

Modern compilers (well, Clang and GCC, at least) can optimize out the multiplication so there is no performance penalty compared to incrementing an offset.
This approach is easier to reason about and is more amenable to vectorization as there are no dependencies in the loop body.

## Zeroing arrays 

Some functions store their output in a user-supplied pointer to a pre-allocated array.
Such functions should not assume that the array has been zeroed.
Incorrectly making this assumption can lead to subtle bugs when the functions are called with non-zeroed buffers.

To test that this assumption is not present in function `X()`, my test suites fill the output buffer with a non-zero initial value from the `initial_value()` function below.
After calling `X()` with `output.data()`, we can compare `output` against the expected output to verify that the function ignores the non-zero initial value.
If the expected output is not readily available, we could instead initialize a separate vector with another call to `initial_value()` and call `X()` on it.
Any inconsistency in the results would indicate that the variable `initial_value()` is affecting the results. 

```cpp
// Taken from https://github.com/libscran/scran_tests:
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

False sharing can be mostly avoided by writing code in a manner that uses automatic variables allocated on each thread's stack.
This is good practice regardless of whether we're in a multi-threaded context.

If threads need to write to the heap, false sharing can be mitigated by creating separate heap allocations within each thread, e.g., by constructing a thread-specific `std::vector`.
This gives `malloc` a chance to use different memory arenas (depending on the implementation) that separates each thread's allocations.
We could guarantee protection against false sharing by aligning all heap allocations to `hardware_destructive_interference_size`;
but then we would need to override `std::allocator` in all STL containers, which seems too intrusive for general use.
Obviously, writing to contiguous memory in the heap across multiple threads should be done sparingly, typically to the output buffer once all calculations are complete.

An interesting sidenote is that each thread will typically create its own `tatami::Extractor` instance on the heap.
Modifications to data members of each `tatami::Extractor` instance during `fetch()` calls could be another source of false sharing.
This might be avoidable by adding an `alignas(hardware_destructive_interference_size)` to the class definition,
but this is only supported by very recent compilers and I'm not even sure if it's legal when the object's natural alignment is stricter;
so, we'll just stick to our ostrich strategy of hoping that `malloc` takes care of it.

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
   using fast math in general is obviously a less palatable option and I can't cajole GCC into using the (unexported) vectorized `log` function without it.
   So, the solution is to just write a variant of a delayed operation helper that uses a vectorized `log` implementation from an external library like **Eigen**.
   Then, users can choose between the standard helper or the vectorized variant that requires an extra dependency.

## Choosing integer types

### General comments

The standard only provides weak guarantees on the size of each integer, e.g., `int` is only guaranteed to be no less than 16 bits.
If a larger minimum size is needed, consider using `long` or `long long`.
If a precise integer size is needed, consider using some of the types from `<cstdint>`.
(A precise type is more predictable across platforms but may always not be available.)

If any of the fixed-size integers or `size_t` are used, they should be namespaced with `std::`.
The `<cstddef>` header should also be imported when using `std::size_t`, so as to avoid relying on a compiler-dependent alias.

`std::size_t` is the most appropriate type for indexing arrays via pointers as it is the size of the largest array.
That said, we should try to avoid spamming `std::size_t` everywhere as it is not guaranteed to be able to represent other integers.
For example, an implementation could define a 16-bit `std::size_t` and 32-bit `int`.
In general, the most appropriate integer type should be chosen to avoid casts and the associated risks of overflow/wrap-around.

### On `Index_`

In **tatami**, the `Index_` template parameter is used to represent the row and column indices.
We expect that the user's choice of `Index_` is large enough to store the extents of both dimensions, i.e., the number of rows/columns.
This also means that `Index_` is often a good choice for internal calculations that involve a single dimension,
as the results of such calculations are usually non-negative and less than the extent.
It can also be safely used to represent 1-based indices as all indices will still be no greater than the dimension extent after incrementing.

Additionally, we expect that the dimension extents will fit into a `std::size_t`.
This is because `tatami::fetch()` accepts a pointer to an array in which the matrix values might be stored, and addressing the elements of an array requires a `std::size_t` .
Under this assumption, indices and extents that are typically stored as `Index_` can be safely converted to `std::size_t` without wrap-around, even if `Index_` is of a larger type.

### Container sizes

Indexed iteration over an arbitrary container `x` should use `decltype(I(x.size()))`.
This ensures that the index type is large enough to span the size of the container.
Of course, this is only relevant if the size of the container is not subject to other constraints -
for example, if we know that the container has length equal to the number of rows, and the number of rows fits into an `int`, it is obviously safe to use `int` to iterate over the container.

```cpp
const auto n = x.size();
for (decltype(I(n)) i = 0; i < n; ++i) {
    // do something here.
}
```

When allocating or resizing a container, we must consider the possibility that the new length is greater than the container's size type.
If the implicit cast wraps around, we would silently create a vector that is smaller than intended.
This can be prevented by using **sanisizer** functions to check for wrap-around:

```cpp
sanisizer::resize(x, new_length);

// For new objects:
auto x2 = sanisizer::create<std::vector<double> >(new_length);
```

In the specific case of resizing to a dimension extent in **tatami**, we could instead use some specialized wrappers around the **sanisizer** functions.
These implement some optimizations by exploiting the assumption that extents can be safely cast to `std::size_t`.

```cpp
tatami::resize_container_to_Index_size(x, new_length);
auto x2 = tatami::create_container_of_Index_size(x, new_length);
```

This protection may be omitted for calls to a container's `reserve()` method, if one exists.
A smaller-than-expected reservation from wrap-around is mostly harmless, as it will be expanded upon insertion.
(Nonetheless, performance degradation can be avoided by using `sanisizer::cap()` to attempt to reserve up to the maximum value of the size type.)
Insertions beyond the container's size type will eventually result in `std::bad_alloc`.

### On `std::vector::size_type`

For STL vectors with the default allocator, we can be fairly confident that `size_type` is no greater than `std::size_t` in any hypothetical implementation.
The requirements of `std::vector::data()` demand that the vector holds its data in an underlying array, which must have a maximum size representable by `std::size_t`.
Moreover, a call to the default allocator must yield an array of length that fits inside `std::size_t`,
and it would be rather unusual - perhaps impossible? - for a `std::vector` implementation to perform multiple allocations to form a region of contiguous storage greater than `std::size_t`.

In addition, it is most pragmatic to assume that `std::size_t` is sufficient to represent the length of STL vectors as it simplifies the overloads.
Many of my functions will accept an arbitrary pointer of input/output data so that they can be used with pointers from `new` or arrays from other langauges' FFIs.
If we assume that `std::size_t` is enough, the same function can be used with `std::vector::data` and the `std::vector` overload can be a simple wrapper.

### On iterator differences

An annoying quirk of C++ is that distances between pointers for the same array (and possibly iterator differences in general) are not guaranteed to fit into the difference type.
For example, `std::ptrdiff_t` is a typically the signed counterpart of `std::size_t`, and an array size that fits in the latter may not fit in the former.
This annoyance is compounded by the fact that many of the STL algorithms return iterators, e.g., `std::max_element`, `std::lower_bound`.
If we need a positional index, we need to convert the returned iterator back to an index via subtraction.
We can do so safely using the `sanisizer::ptrdiff()` function:

```cpp
std::vector<double> x(vec_length);
sanisizer::can_ptrdiff<decltype(I(x.begin()))>(vec_length);
auto max_it = std::max_element(x.begin(), x.end());
auto max_idx = max_it - x.begin(); // known to be safe after can_ptrdiff().
```

In practice, this may not be necessary as glibc 2.30 prohibits allocations beyond the maximum size of `PTRDIFF_MAX`.
(Presumably the same restrictions apply for `std::vector`.)
However, this is a implementation choice, and the standard explicitly mentions that the pointer difference may not fit in `std::ptrdiff_t` in a conforming implementation!
So, better to be safe than sorry.
