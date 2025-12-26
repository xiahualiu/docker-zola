+++
title = "The Art of Flattening: How to NOT Nest Your Code"
date = 2024-05-10
draft = false
[taxonomies]
  tags = ["Design Patterns", "C++", "Clean Code"]
[extra]
  toc = true
  keywords = "Code, C++, Design Patterns, Nesting, Guard Clauses"
+++

At the very beginning of the [Linux kernel coding style](https://www.kernel.org/doc/html/v4.10/process/coding-style.html) document, it bluntly states:

> if you need more than 3 levels of indentation, you’re screwed anyway, and should fix your program.

Code nesting—often called "Arrow Code" because the indentation shapes look like an arrow pointing to the right—exists in almost all programming languages. While indentation is necessary for structure, deep nesting dramatically increases cognitive load.

Have you ever stared at a block of code indented five tabs deep and wondered how to flatten it? This article explores design patterns and techniques specifically tailored to avoid nesting in C++.

## Table-Driven Methods

Sometimes, the cleanest logic is no logic at all—it's just data. Instead of writing complex `switch-case` or chained `if-else` statements, you can often use a **lookup table**.

For example, imagine a function that takes an integer input `n` (1 to 12) and returns the number of days in that month. A novice approach might use a massive switch statement. A table-driven approach is $O(1)$ and readable:

```cpp
int getDaysOfTheMonth(const int n)
{
    // A simple look-up table
    static const int daysInMonths[12] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
    return daysInMonths[n-1];
}
```

To make this production-ready, we need to handle leap years and validate input. Notice how we can handle the logic without creating deep branches:

```cpp
int getDaysOfTheMonth(const int n, const int year)
{
    if (n < 1 || n > 12 || year < 0) {
        throw std::runtime_error("Input out of range!");
    }

    // Leap year logic: divisible by 4, unless divisible by 100 but not 400
    bool isLeap = (year % 4 == 0) && (year % 100 != 0 || year % 400 == 0);
    
    // Feb is index 1. If it's a leap year, add 1 day.
    static const int daysInMonths[12] = {31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
    
    int days = daysInMonths[n-1];
    if (n == 2 && isLeap) {
        days += 1;
    }
    
    return days;
}
```

This concept extends beyond arrays. You can use std::map or std::unordered_map to map inputs to values, or even inputs to functions (function pointers or lambdas), effectively replacing complex control structures with hash lookups.

## Early Return (Guard Clauses)

One of the easiest ways to reduce nesting is the Early Return pattern, also known as Guard Clauses. Instead of wrapping your "happy path" logic inside an if block, you invert the condition and return (or throw) immediately.

### The Nested Way

Here, the core logic is buried inside error checking:

```cpp
void processUser(User* user) {
    if (user != nullptr) {
        if (user->isValid()) {
            // Actual logic happens here
            saveToDatabase(user);
        } else {
            throw std::exception("Invalid user");
        }
    } else {
        throw std::exception("Null user");
    }
}
```

### The Flattened Way

By handling the edge cases first, the "happy path" remains at the root indentation level:

```cpp
void processUser(User* user) {
    if (user == nullptr) {
         throw std::exception("Null user");
    }
    
    if (!user->isValid()) {
        throw std::exception("Invalid user");
    }

    // Actual logic happens here, un-nested
    saveToDatabase(user);
}
```

This aligns with the mental model of most developers: handle the preconditions, then do the work.

## State Design Pattern

Finite State Machines (FSM) are often implemented with massive switch-case blocks that check a state variable. As the complexity grows, this becomes unreadable.

The State Pattern leverages polymorphism to flatten this logic. Instead of if (state == ON), the code delegates behavior to an object representing the current state.

Imagine a ceiling fan where the buttons do different things depending on the current speed.

### The Implementation

We define an interface FanState and concrete states.

```cpp
struct Fan; // Forward declaration

struct FanState {
    virtual void increaseSpeed(Fan& fan) = 0;
    virtual void stop(Fan& fan) = 0;
    virtual ~FanState() = default;
};

struct Fan {
    std::unique_ptr<FanState> state;
    int speed = 0;

    void clickIncrease() {
        state->increaseSpeed(*this);
    }
};

// Concrete State: Stopped
struct StoppedState : FanState {
    void increaseSpeed(Fan& fan) override {
        fan.speed = 1;
        std::cout << "Fan started. Speed: 1\n";
        // Transition to next state (simplified logic)
        // In real code, you would assign a new 'RunningState' here
    }
    void stop(Fan& fan) override { /* Already stopped */ }
};
```

In the usage code, there are no conditionals. The logic is encapsulated within the classes:

```cpp
int main() {
    Fan myFan;
    myFan.state = std::make_unique<StoppedState>();
    
    // No if-else needed; the logic is polymorphic
    myFan.clickIncrease(); 
}
```

This removes deeply nested logic regarding state transitions from your main controller class. You can read more about this pattern here.

## Factory Method

Similar to the State pattern, the Factory Method allows you to encapsulate the creation logic. If you find yourself writing if (type == "A") return new A(); else if (type == "B")..., a Factory pattern can hide that branching complexity, keeping your main business logic flat.

## Use Higher-Order Functions

Modern C++ (C++11 and beyond) provides powerful tools in the <algorithm> library that act as higher-order functions. While C++ doesn't have a method named map on arrays like Python or JavaScript, it has std::transform.

## The Loop Way

loops often lead to nesting, especially when combined with filtering (if inside for).

```cpp
std::vector<int> input = {1, 2, 3, 4, 5};
std::vector<int> output;

for (int i : input) {
    if (i % 2 == 0) { // Nesting level 1
        output.push_back(i * i);
    }
}
```

## The Algorithm Way

Using standard algorithms flattens the logic and makes the intent (Filter -> Map) clear.

```cpp
std::vector<int> input = {1, 2, 3, 4, 5};
std::vector<int> output;

// Copy only if even
std::copy_if(input.begin(), input.end(), std::back_inserter(output), 
             [](int i){ return i % 2 == 0; });

// Transform (Map) in place to square them
std::transform(output.begin(), output.end(), output.begin(), 
               [](int i){ return i * i; });
```

With C++20 Ranges, this becomes even more declarative and completely flat, resembling the functional style of Rust or Python.

## Conclusion

Nesting isn't inherently bad, but deep nesting hides bugs and hurts readability. By using Table-Driven methods for static data, Guard Clauses for error checking, and Design Patterns for complex state behavior, you can keep your code as flat as the logic allows.
