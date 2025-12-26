+++
title = "Why is std::hash a struct?"
date = 2024-08-28
draft = false
[taxonomies]
  tags = ["C++"]
[extra]
  toc = true
  keywords = "C++"
+++

If you look at the C++ standard, you will notice that `std::hash` is defined as a struct (specifically, a class template) rather than a simple function. You can verify this in the [cppreference documentation](https://en.cppreference.com/w/cpp/utility/hash).

Despite being a struct, it acts like a function because it overloads the `operator()`:

```cpp
std::size_t h = std::hash<int>{}(42); // Acts like a function call
```

Why did the standards committee design it this way? There are three main reasons:

1. Partial Specialization.
2. Statefulness.
3. Legacy Type Traits.

## Partial Template Specialization

The most significant technical reason is that **C++ does not support partial specialization for function templates**, only for class/struct templates.

If `std::hash` were a function, you could overload it for specific types (like `int` or `std::string`), but you could not write a generic implementation for a container like `std::vector<T>`.

Because `std::hash` is a struct, we can partially specialize it to handle families of types. For example, if you wanted to create a hash for any `std::vector<T>`, you would need partial specialization:

```cpp
// This is possible because std::hash is a struct
template <typename T>
struct hash<std::vector<T>> {
    std::size_t operator()(const std::vector<T>& v) const {
        // ... hash logic ...
    }
};
```

If `std::hash` were a function, this generic matching for "vector of anything" would be significantly harder to implement cleanly.

## Stateful Hashing (Salted Hash)

Using a **struct** allows the hasher to hold state. While the default specializations for standard types are stateless, users can define custom hashers that store internal data.

A common use case is a **Salted Hash**. To prevent Hash DoS attacks or to randomize iteration order, you might want a hasher that holds a random seed (salt) initialized at runtime.

```cpp
struct SaltedHash {
    std::size_t salt;
    
    SaltedHash(std::size_t s) : salt(s) {}

    std::size_t operator()(const std::string& key) const {
        return std::hash<std::string>{}(key) ^ salt; 
    }
};
```

If `std::hash` were just a function pointer or a static function, creating distinct instances with unique internal seeds would be impossible without using global variables.

## Legacy Type Information (Pre-C++17)

Historically (before C++17), function objects needed to expose meta-information to work with the standard library adaptors (like `std::not1` or `std::bind1st`).

By inheriting from `std::unary_function`, structs provided typedefs automatically:

```cpp
std::hash<T>::argument_type // T
std::hash<T>::result_type   // std::size_t
```

While C++17 rendered this obsolete via `decltype` and improved metaprogramming capabilities, the design of `std::hash` predates these improvements.

## Conclusion

`std::hash` is a struct to leverage the full power of C++ generic programming. It allows for partial template specialization (essential for generic containers), supports stateful hashing (like salts), and historically provided necessary type traits for the STL.

It gives the user the freedom to treat the hashing process as an object, not just an operation.
