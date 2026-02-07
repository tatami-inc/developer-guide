# Choosing integer types

## General comments

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

## On `Index_`

In **tatami**, the `Index_` template parameter is used to represent the row and column indices.
We expect that the user's choice of `Index_` is large enough to store the extents of both dimensions, i.e., the number of rows/columns.
This also means that `Index_` is often a good choice for internal calculations that involve a single dimension,
as the results of such calculations are usually non-negative and less than the extent.
It can also be safely used to represent 1-based indices as all indices will still be no greater than the dimension extent after incrementing.

Additionally, we expect that the dimension extents will fit into a `std::size_t`.
This is because `tatami::fetch()` accepts a pointer to an array in which the matrix values might be stored, and addressing the elements of an array requires a `std::size_t` .
Under this assumption, indices and extents that are typically stored as `Index_` can be safely converted to `std::size_t` without wrap-around, even if `Index_` is of a larger type.

## Container sizes

Indexed iteration over an arbitrary container `x` should use the container's size type, i.e., `I<decltype(x.size()))>`.
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
A smaller-than-expected reservation from wrap-around is usually harmless, as it will be expanded upon insertion.
Insertions beyond the container's size type will eventually result in `std::bad_alloc`.
Nonetheless, if the requested size is really necessary, `sanisizer::reserve()` can be used to ensure that it was correctly reserved.

## On `std::vector::size_type`

For STL vectors with the default allocator, we can be fairly confident that `size_type` is no greater than `std::size_t` in any hypothetical implementation.
The requirements of `std::vector::data()` demand that the vector holds its data in an underlying array, which must have a maximum size representable by `std::size_t`.
Moreover, a call to the default allocator must yield an array of length that fits inside `std::size_t`,
and it would be rather unusual - perhaps impossible? - for a `std::vector` implementation to perform multiple allocations to form a region of contiguous storage greater than `std::size_t`.

In addition, it is most pragmatic to assume that `std::size_t` is sufficient to represent the length of STL vectors as it simplifies the overloads.
Many of my functions will accept an arbitrary pointer of input/output data so that they can be used with pointers from `new` or arrays from other langauges' FFIs.
If we assume that `std::size_t` is enough, the same function can be used with `std::vector::data` and the `std::vector` overload can be a simple wrapper.

## On iterator differences

An annoying quirk of C++ is that distances between pointers for the same array (and possibly iterator differences in general) are not guaranteed to fit into the difference type.
For example, `std::ptrdiff_t` is a typically the signed counterpart of `std::size_t`, and an array size that fits in the latter may not fit in the former.
Similarly, for `std::vector`, we could end up in a situation where `vec[i]` will work while `*(vec.begin() + i)` will not when `i` is between the maximum sizes of `std::ptrdiff_t` and `std::size_t`.
(`std::vector` iterator arithmetic will cast its integer inputs to the vector's difference type.)
This issue is compounded by the fact that many of the STL algorithms return iterators, e.g., `std::max_element`, `std::lower_bound`.
If we need a positional index, we need to convert the returned iterator back to an index via subtraction.

Some implementations protect against this discrepancy by limiting allocations to `PTRDIFF_MAX`.
See [here](https://lists.gnu.org/archive/html/info-gnu/2019-08/msg00000.html) for glibc's behavior with `malloc` -
presumably the same restrictions apply for implementations of `std::vector`, as discussed [here](https://discourse.llvm.org/t/libc-max-size-of-a-std-vector/35449).
For the sake of sanity, we will assume that such protection is already present when writing our code. 
Otherwise, the issue is too pervasive to cheaply defend against - for example, every `size_t` would have to be checked for overflow at run time when used in iterator arithmetic. 
We think it's fair to expect a good implementation to produce an array/vector that doesn't suddenly exhibit undefined behavior past a certain size.

In practice, this shouldn't matter too much.
For typical implementations where `std::ptrdiff_t` is the signed counterpart of `std::size_t`,
the discrepancy is only relevant when the size of each array element is no greater than 1. 
Then there's the matter of actually requesting an allocation that is large enough to exceed `PTRDIFF_MAX`,
which is very unlikely for the vast majority of 64-bit systems.

## Converting to/from floating-point

When converting an integer to floating-point, we assume that the implementation's floating-point numbers are IEEE-754 compliant.
This allows out-of-range conversions to safely overflow to the signed infinities.
Note that we already assume IEEE-754 compliance to guarantee the safety of delayed arithmetic operations in **tatami**;
it would much be too tedious to manually check for potential overflow and prevent undefined behavior in a non-compliant implementation.
In practice, overflow should be very rare given that a single-precision float can store all 64-bit integers without overflow.

In the unusual case where we need to convert a floating-point number to an integer, we can use `sanisizer::from_float()` to protect against undefined behavior due to overflow.
This is more robust than checking the `FE_INVALID` flag, which seems to be set for non-finite conversions but is not reliably set for overflow.
For example, when casting doubles to a smaller integer type, GCC on x86-64 uses the same instruction (`cvttsd2si`) to first cast the double to a 64-bit integer and then truncate the result.
As no overflow actually occurs in the floating-point-related instruction, the exception flag is never set by the hardware. 

## Thoughts on `-Wconversion`

It's tempting to set `-Wconversion` and related flags (e.g., `-Wsign-conversion`) to identify all implicit conversions.
This would force us to check that all conversions are either known to be safe or need to be wrapped in a `sanisizer::cast`.
In practice, this is too arduous as such casts become very verbose:

```cpp
Index_ x = 10;
auto x = sanisizer::create<std::vector<double> >(x);

for (Index_ i = 0; i < x; ++i) { // all accesses are known-safe.
    x[i]; // not okay, needs cast to size_type.
    x.begin() + i; // still not okay, needs cast to the iterator's difference_type

    x[static_cast<decltype(x.size())>(i)]; // correct
    x.begin() + static_cast<typename std::iterator_traits<decltype(x.begin())>::difference_type>(i); // correct
}
```
