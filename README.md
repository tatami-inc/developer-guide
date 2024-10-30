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
