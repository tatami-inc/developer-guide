# Miscellaneous

## Naming

Types should be named with `PascalCase`.

Variables and functions should be named with `snake_case`.

Template parameters should have a trailing underscore to distinguish them from non-template parameters.
This occurs regardless of whether a template parameter is a type.

## Using `auto`

We use `auto` liberally to define types for the return values of methods/functions or to create copies/references to other variables.
This is convenient and avoids inadvertent casts. 

We prefer not to use `auto` for defining the types of arithmetic results, as it can be difficult to predict the type of integer promotions. 
`auto` should not be used at all in frameworks like **Eigen** that make heavy use of expression templating.

`auto` should also not be used when assigning a literal. 
A specific type is preferred, even if it is implicitly defined by `decltype`:

```cpp
for (decltype(total) i = 0; i < total; i++) {
    // See comments below about decltype().
}
```

## Using `const` <a id='using-const'></a>

Where possible, variables shouhld be `const`.
This reduces the risk of accidental modification.

## Using `decltype`

### Stripping qualifiers

One feature of `decltype()` is that it will preserve the reference in the type.
This can be dangerous - for example, the code below will modify `start` by its reference `i`:

```cpp
void loop_body(int& start, int& end) {
    for (decltype(start) i = start; i < end; ++i) { // 'i' is a 'int&' alias for 'start'.
        // Do something
    }
}
```

A simple solution is to define a small alias that clears out all qualifiers and references.

```cpp
#include <type_traits>

template<typename Input_>
using I = std::remove_cv_t<std::remove_reference_t<Input_> >; // i.e., identity().

// Could also achieve the same effect with std::remove_cvref_t in C++20,
// or even auto to remove references/const/etc. during template deduction.

void loop_body2(int& start, int& end) {
    for (I<decltype(start)> i = start; i < end; ++i) { // 'i' is now an 'int'.
        // Do something
    }
}
```

Given how simple it is to do, I'd suggest just adding `I<>` around all `decltype()` statements that should return a non-reference type.
This eliminates any concern about references without manual checking of the `decltype()`'d expression, especially those involving a function call.
It also avoids unintended propagation of `const`-ness when we follow the [previous section's advice](#using-const):

```cpp
void foo3(const int x) {
    I<decltype(x)> y = x; // removes constness so that we can modify 'y'.
    ++y;
}
```

### Prefer `auto`

Personally, I would prefer to use `auto` over `decltype()` in most situations where a variable is defined.
`auto` doesn't need the `I<>` trick to deduce a non-reference, it's shorter to write, and it doesn't require updating upon changes to variable names. 
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
    // 'start + length' may not be the same as 'start' after integer promotion,
    // so we force it to be the same via 'decltype()'.
    for (I<decltype(start)> i = start, end = start + length; i < end; ++i) {
        // Do something.
    }
}
```
