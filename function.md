# Functions

## `const` arguments <a id='using-const'></a>

Where possible, arguments should be `const`. 
This reduces the risk of accidental modification, especially from inadvertent references.
Consider:

```cpp
template<typename Bar_>
void foo(Bar_ x) {
    Bar_ y = x;
    ++y;
}
```

If `Bar_` is defined to be a reference, e.g., from `decltype()`, then `foo()` would modify the input value, which is probably not what we intended.
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

## Default arguments

Functions should not have default arguments.
Instead, functions should accept an `Options` struct that contains all arguments with their default values.
This avoids confusion when a caller needs to specify a subset of those arguments or if we need to add more options.
Users willing to accept all default arguments can simply call the function with `{}`, which is convenient enough that no overload is needed. 

If an argument is optional but has no sensible default, it should be wrapped in a `std::optional`.
This is more explicit than using a sentinel value (e.g., -1 for unsigned integers) and more concise than passing in a separate `bool` for this purpose.

Default values for integer arguments should consider using `sanisizer::cap()` to ensure that the value is below the maximum guaranteed by the standard for that integer type.
For example, a `std::size_t` is only guaranteed to store up to 65535, so setting a greater default value could result in compiler warnings on some unusual implementations.
This scenario is not uncommon for setting buffer sizes where a larger default value is preferred for performance on typical systems but lies beyond the standard-guaranteed upper limit.

## Templating <a id='function-templating'></a>

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

## Lambdas

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

Keep in mind that IIFEs use `auto` type deduction rules for the return type.
This means that they will never return a reference type, which can be unexpected when using an IIFE to replace ternary expressions,
e.g., for multi-line expressions or compile-time switches.
In these (relatively rare) cases, we should instead write out a full lambda and invoke it:

```
const auto& x = [...]() -> const auto& {
    /* body */
}();
```
