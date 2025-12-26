+++
title = "The C++ RVO Trap: When Deleted Move Constructors Break Your Code"
date = 2025-01-14
draft = false
[taxonomies]
  tags = ["C++", "Programming", "Linux"]
[extra]
  toc = true
  keywords = "C++, RVO, NRVO, Move Semantics, Copy Elision"
+++

## The Surprising Case

You write a simple C++ class. You want to be explicit, so you define a copy constructor but delete the move constructor. You expect **RVO** (Return Value Optimization) to make the return efficient.

Instead, you get a compile error.

```cpp
// filepath: rvo_trap.cpp
class Widget {
public:
    Widget() = default;
    Widget(const Widget&) = default;   // Copy constructor
    Widget(Widget&&) = delete;         // Deleted move constructor

    int value{42};
};

Widget createWidget() {
    Widget w;
    return w;  // Error: call to deleted move constructor
}
```

Why does the compiler complain about a deleted move constructor if it's supposed to elide the copy anyway?

## Why This Happens

We need to distinguish between two cases:

- Guaranteed copy elision (RVO): returning a temporary (`return Widget();`). Since C++17, this works even if all constructors are deleted.
- NRVO (named return value optimization): returning a named variable (`return w;`). This is an optimization, not a guarantee.

Because NRVO is optional, the standard requires a valid fallback plan if the compiler chooses not to optimize. That fallback is chosen via overload resolution.

## The Overload-Resolution Trap

When you return the local variable `w`, the compiler treats it as an rvalue and considers these candidates:

- `Widget(const Widget&)` — copy constructor
- `Widget(Widget&&)` — move constructor (marked `delete`)

Deleted functions still participate in overload resolution. The rvalue `w` matches `Widget&&` better than `const Widget&`, so the move constructor “wins.” Only after selecting it does the compiler notice it is deleted, triggering the error. It never falls back to the copy constructor because the better match was chosen first.

## The Fix

Do not explicitly delete the move constructor. If you define a copy constructor, the compiler will not implicitly declare a move constructor—the move simply will not exist.

```cpp
// filepath: solution.cpp
class Widget {
public:
    Widget() = default;
    Widget(const Widget&) = default;
    // Widget(Widget&&) = delete;  // Remove this line

    int value{42};
};

Widget createWidget() {
    Widget w;
    return w; // OK: copy is the only candidate; NRVO usually applies
}
```

Why this works:
- `w` is treated as an rvalue.
- The move constructor does not exist, so the only candidate is `Widget(const Widget&)`.
- The compiler has a valid fallback plan (copy), and will typically perform NRVO so no call happens at all.

## Important Note on Side Effects

Copy elision can skip constructor calls entirely. Do not rely on side effects inside copy or move constructors.

```cpp
Widget(const Widget& other) {
    global_counter++; // Risky: might be optimized away by RVO/NRVO
}
```

Design classes assuming copy/move construction may be elided.