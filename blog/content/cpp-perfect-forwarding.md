+++
title = "Perfect forwarding in C++"
date = 2024-08-28
draft = false
[taxonomies]
  tags = ["C++"]
[extra]
  toc = true
  keywords = "C++"
+++

Perfect forwarding is a powerful C++ technique used primarily in template programming to preserve the value category (lvalue vs. rvalue) of arguments passed to a function. It is essential for writing generic wrapper functions that are both efficient and correct.

You can find the official documentation on [cppreference.com](https://en.cppreference.com/w/cpp/utility/forward).

## The Problem: Losing Value Categories

Imagine you are writing a wrapper function that accepts an argument and passes it to another function `foo()`.

```cpp
void foo(int& x) { std::cout << "lvalue ref" << std::endl; }
void foo(int&& x) { std::cout << "rvalue ref" << std::endl; }

template<typename T>
void wrapper(T arg) {
    foo(arg);
}
```

If we pass an expensive object to `wrapper`, it gets copied into `arg`. To avoid copying, we might try to take a reference:

```cpp
template<typename T>
void wrapper(T& arg) {
    foo(arg);
}
```

But now we can't pass rvalues (like temporary objects `wrapper(5)`) because you cannot bind a non-const lvalue reference to an rvalue.

## The Solution: Universal References and `std::forward`

C++11 introduced "Universal References" (also known as Forwarding References), denoted by `T&&` in a template context.

```cpp
template<typename T>
void wrapper(T&& arg) {
    // 'arg' itself is an lvalue here, because it has a name.
    foo(std::forward<T>(arg)); 
}
```

### How `std::forward` Works

`std::forward` relies on **Reference Collapsing Rules**. When you pass an argument to `wrapper(T&& arg)`:

1. **If you pass an lvalue (`int x`):** `T` is deduced as `int&`. The type `T&&` becomes `int& &&`, which collapses to `int&`. `std::forward` casts it to `int&`.

2. **If you pass an rvalue (`5`):** `T` is deduced as `int`. The type `T&&` becomes `int&&`. `std::forward` casts it to `int&&`.

This conditional casting ensures that `foo` receives exactly what was passed to wrapper:

* If `wrapper` received an lvalue, `foo` sees an lvalue.
* If `wrapper` received an rvalue, `foo` sees an rvalue.

### Difference from `std::move()`

It is easy to confuse `std::forward` with `std::move`, but they serve different purposes.

* `std::move(x)`: Unconditionally casts `x` to an rvalue. You use this when you are done with an object and want to transfer ownership.
* `std::forward<T>(x)`: Conditionally casts `x` to an rvalue only if `x` was originally an rvalue.

| Feature | `std::move` | `std::forward` |
| :--- | :--- | :--- |
| Action | Always casts to rvalue | Casts to rvalue only if `T` is not an lvalue reference |
| Use Case | Moving ownership | Generic wrappers / Templates |
| Requirement | No template args needed | Requires template type `T` |

## Summary

Use `std::forward<T>()` exclusively in templates when you have a "Universal Reference" (`T&&`) and you want to pass the argument to another function exactly as it was received.

Use `std::move()` when you specifically want to trigger move semantics on a concrete object.