# Classes

## `class` vs `struct`

Classes should not have any public members.
Access to members should be granted via getters or setters.
All members should be prefixed by `my_` to avoid confusion with arguments in the constructor and initialization list.

Structs should have public members only, with no private members or methods.
These are treated as passive data carriers with no additional functionality.
A non-default constructor is allowed, though.

(Some of these ideas were taken from Google's C++ style guide.)

## `const` methods

Class methods should be marked as `const` wherever possible, to protect against accidental modification of the object.
We also treat `const` as an implicit indication that the method is thread-safe.

We also recommend setting the method arguments as `const`, as described for [functions](function.md#using-const).
Note that the `const`-ness is ignored when matching function signatures,
so we can hide the implementation detail from the interface, e.g., for virtual methods:

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

## Templating

Template parameters should only contain types or flags that influence the type, see the logic described [here](function.md#function-templating).

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
