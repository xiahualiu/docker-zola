+++
title = "Type Conversions in C++ and Rust"
date = 2024-10-19
draft = false
[taxonomies]
  tags = ["Rust", "C++"]
[extra]
  toc = true
  keywords = "Rust, C++"
+++

Both C++ and Rust provide language-level type conversions, but they work very differently. C++ allows implicit conversions driven by constructors and conversion operators, while Rust relies on explicit traits such as `Deref`, `AsRef`, `From`, and `Into`.

## C++ Implicit Type Conversion

Implicit conversions are documented in the standard (see [cppreference](https://en.cppreference.com/w/cpp/language/implicit_conversion)). The compiler tries to build a value of the required type using constructors or conversion operators. The conversion fails if no viable candidate exists or if overload resolution finds multiple viable candidates.

There are also built-in promotions and narrowing rules for integers and floating-point types.

Something worth noting is the interaction between implicit conversions and overload resolution.

For example, two overloads might both become viable only after conversions, and the compiler will emit an ambiguity error.

However C++ also provides ways to disable implicit conversions. Besides marking constructors as `explicit`, we can use template deletion so only an exact type is accepted:

```cpp
template<typename T>
T only_double(T f) = delete;

template<>
double only_double<double>(double f) {
    return f;
}
```

So when we call this `only_double` with any parameter type other than `double` will trigger a compilation error.

```cpp
double fd=1.0;
float ff=1.0;
only_double(ff); // error: use of deleted function 'T only_double(T) [with T = float]'
only_double(fd);
```

You can also do explicit conversions via `static_cast`, `reinterpret_cast`, and `dynamic_cast` when you need to opt in consciously.

### Conclusion

In general C++ implicit type conversion is very useful and flexible.

The main pitfall is silent changes to integer and floating-point values when a narrowing conversion happens.

Flags like `-Wconversion` or `-Wnarrowing` (GCC/Clang) help catch these cases, but it is still a safety hazard, especially for less experienced programmers.

However this type of problem can be found easily with SAST tools like `clangd`, and I highly suggest any serious C++ programmer should use it while doing their projects.

## Rust Type Conversion

Rust provides two mechanisms that participate in borrowing conversions: `Deref` and `AsRef`.

* `Deref` enables deref coercions: references to a type can be coerced to references of its `Target`. It is triggered implicitly when dereferencing or when the compiler needs a reference of the target type.
* `AsRef` is an explicit trait method (`a.as_ref()`) that returns a reference to a target type chosen by the trait implementation.

There are also 2 common methods to convert the object type without referencing or dereferencing. These 2 are like `static_cast` in C++.

* `From` allows constructing a type from another.
* `Into` converts a value into another type; it is automatically available when `From` is implemented.

| Trait | Usage |
| :---: | :---: |
| Deref | &A -> &B |
| AsRef | A.as_ref() -> &B |
| From | A::from(B) -> A |
| Into | A.into() -> B |

### Conclusion

Rust has no general implicit type conversion. Conversions only occur when trait implementations exist, and except for `Deref` coercions they are opt-in through method calls.

This greatly improves safety, though the trait-based approach can feel more complex than C++ at first.