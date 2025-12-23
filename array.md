# Arrays

## Zeroing

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
