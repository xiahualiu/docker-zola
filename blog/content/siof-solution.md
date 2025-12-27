+++
title = "Solving the Static Initialization Order Fiasco (SIOF) with Meyer's Singleton"
date = 2025-01-14
draft = false
[taxonomies]
  tags = ["C++"]
[extra]
  toc = true
  keywords = "C++"
+++

## What is SIOF?

The **Static Initialization Order Fiasco (SIOF)** is a bug that appears when one translation unit initializes a global object that depends on a global object from another translation unit. Within a single file, globals initialize top to bottom. Across files, the order is undefined, so a dependency may run before its prerequisite exists.

## Example: Logger vs Database

Two files each define a global: a logger and a database. The database logs during construction, so it relies on the logger being ready.

```cpp
// logger.h
struct Logger {
    Logger() { std::cout << "Logger Initialized\n"; }
    void log(const char* msg) { std::cout << "Log: " << msg << "\n"; }
};

extern Logger globalLogger;
```

```cpp
// logger.cpp
#include "logger.h"

Logger globalLogger;  // global instance
```

```cpp
// database.cpp
#include "logger.h"

struct Database {
    Database() {
        // Risky: depends on another global's constructor having run
        globalLogger.log("Database starting up...");
    }
};

static Database globalDB;  // SIOF may strike here
```

If the runtime initializes `globalDB` before `globalLogger`, the database constructor calls `log()` on an unconstructed object, often causing a crash.

## Startup Phases

- Static initialization (safe)
  - Zero initialization: all global storage is zeroed.
  - Constant initialization: e.g., `const int x = 42;` baked into the binary.
- Dynamic initialization (risky)
  - Global constructors run; order across translation units is unspecified.

## Fix: Meyer's Singleton

Move the static object inside a function so it initializes on first use, not at link time.

```cpp
// logger.h
class Logger {
public:
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;

    static Logger& get() {
        static Logger instance;  // magic static: initialized on first call
        return instance;
    }

    void log(const char* msg) { std::cout << "Log: " << msg << "\n"; }

private:
    Logger() { std::cout << "Logger Initialized\n"; }
};
```

```cpp
// database.cpp
#include "logger.h"

struct Database {
    Database() { Logger::get().log("Database starting up..."); }
};

static Database globalDB;
```

## Why It Works

- Lazy initialization: `Logger` constructs exactly when `get()` is first called.
- Deterministic dependency: Even if `globalDB` starts first, it triggers `Logger::get()` and the logger is built before use.

## Thread Safety

Since C++11, function-local statics are initialized in a thread-safe manner: concurrent first calls to `get()` still construct `Logger` exactly once.

## Takeaway

When globals depend on each other, replace them with function-local statics (Meyer's Singleton) to avoid SIOF and guarantee safe, deterministic startup.