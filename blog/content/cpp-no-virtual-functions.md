+++
title = "Avoid virtual functions in C++ (when possible)"
date = 2024-08-27
draft = false
[taxonomies]
  tags = ["C++"]
[extra]
  toc = true
  keywords = "C++"
+++

Virtual functions are a core feature of C++, introduced at the very beginning to enable *Run-time Polymorphism*. This gives C++ a capability that C lacks natively: the ability to treat different types as a single abstract base type.

However, many developers overlook the "dark side" of this feature. It comes with hidden costs in both memory and performance.

## The Cost of Runtime Polymorphism

Let's say you are writing a database application. You decide that every data type must have a `print()` function that returns a `std::string`.

The classic OOP approach is to define an abstract base class with a pure virtual `print()` function. Later, you implement this for specific types, like `Int1`.

*(You can find the example code [HERE](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIM6SuADJ4DJgAcj4ARpjEIADMAOykAA6oCoRODB7evv6p6ZkCIWGRLDFxSbaY9o4CQgRMxAQ5Pn4BdpgOWQ1NBCUR0bEJyQqNza15HeP9oYPlw0kAlLaoXsTI7Bzm8aHI3lgA1CbxbmPEocAn2CYaAII7eweYx6dO55is17cP9/tMCgUhwsAMwABEmI1jokrPdDvDDikvFFaHhkCAfgjDgA3PDNLxiQ5jdAgEDnS6Ii6CCBLQ5oBhjV5gw4aE6wh6JMFsn4/f6Aw4ASUEXEOIERyNRyGBoIhUJMMMxCKRKLRGLhCKFBC4EHpjNCBEOTFpYuxYi8mAgRuhVk5ivhxNJ5KMlP1NLpAkZqGxsQuR3l7KxWOImAI6wYRIIJJARAA%2Bk7gBACAg8AoALTXU3eTBLbnq%2BHyrl5l2mgjsO2B/U4s2YXMcrnxdm8gz8zVma3l5WStV3LGtnUeg2Vq0m6uW2n%2Bgvlh1kghU4Au6m03UGidFoMhsMRqOx%2BOJ5NpjPVnMN8uTospC4lstrhGVzPm2uTk/3H6VlhMUKHN2rnsa4WG7UuA0Y8Az/Ag2yYMwICAkDy0rKIRROZkMgAL0wVAqEtLhYKLeC2yQok8DQjDLTMHDf3tSNSTQLwV1OE43GOMwzCEIiXgwwVhTFcx8Po05DgQ143AYrdSVcWhayxacaLo4T%2BJ41i0MODjW245ihJEqJeLkxjp3EySEWDUNiHDVlnzrDgVloTgAFZeD8bheFQThhMsawiTWDYXh2HhSAITRLJWABrBJ4gAOniSKoui6KADZ9E4SR7IC0hnI4XgFBADQ/IClY4FgJA0BYFI6FichKCKkr6DiYApACGhaFLYhMogKIUqiUImgAT04XyOuYYguoAeSibQun8xzSCKthBCGhhaB6jgtFILAoi8YA3DEWhMsmrB3yMcQlt4fBg26b0duWzBVC6WitmW/UahS1EomIbqPCwFLZzwFhet4b1iCidJwUwfbgFRIxcr4AxgAUAA1PBMAAdyGlJGF%2BmRBBEMR2CkDH5CUNQUt0Lh9EMYxrGsfQ8CiTLIBWVAUjqBlOCc/7fUwWmaWqWoshcBh3E8No9GCOYygqPQ0gyJnJj8EnJaKBgBjF4YSc6bp6hmGW9DVpnemaJWhjiVXNcFvJjb6A2FiNlYFE8zYJCs2zkqO1LOEOVQAA5YtTWLJEOYBkClKQwrbCBcEIEgmPibDeAmrQlmC0KIpilPIvi6yOCS0gHOWtKMqynKjrymBEBANYCCRAhyv7YrSuIcJWC2T3vd9/3A8OYOzF4TB8CIX09H4THRHEXHB/xlR1Bd4nSER16Ul%2Bx2ODs7OUrSobaMr5SqHdr2fb9gOg8kEOvw8WvqqjmPC/jxPIuT1OYoSzPndzlnbALuPAsfruV5dvOr8//6GRnCSCAA%3D%3D))*

### Memory Overhead

The first issue is size. Even though `Int1` only holds a single 4-byte `int` member variable, `sizeof(Int1)` is **16 bytes** on a 64-bit machine.

In contrast, `Int2` (which has no virtual functions) takes up only **4 bytes**.

Where does this size difference come from?

If you check the assembly in the constructor of `Int1`, you'll see the program storing an additional value: `vtable for Int1+16`. This is the **vptr** (virtual pointer), which points to the `vtable` in memory.

```assembly
mov     edx, OFFSET FLAT:vtable for Int1+16
mov     rax, QWORD PTR [rbp-8]
mov     QWORD PTR [rax], rdx    # Write 8 bytes (vtable address)
```

The `Int2` constructor, however, simply writes the integer value and returns.

We need this pointer because **the vtable pointer is unrelated to the object type during runtime**. If you `static_cast` an `Int1` object to `BaseData`, the object still points to Int1's vtable. This is how the program knows to call Int1::print() even when holding a BaseData pointer.

But if you are storing billions of these integers in a database, quadrupling your memory usage (4 bytes -> 16 bytes) is a disaster.

### Performance Overhead

The second issue is speed. Calling a virtual function requires a "pointer chase":

1. Follow the object's `vptr` to the `vtable`.
2. Look up the correct function address in the table.
3. Jump to that address.

This has several side effects:

* **No Inlining:** The compiler cannot inline the function because it doesn't know which function to call until runtime.
* **Optimization Barriers:** As seen in the Godbolt example, the compiler optimized away `Int2` entirely because it saw the code was unused. It could not do the same for `Int1` because the virtual nature implies the function might be called externally.
* **Cache Misses:** You are jumping to different memory locations (object -> vtable -> code), which hurts CPU cache performance.

Even using the `final` keyword—which suggests to the compiler that it can devirtualize—is not a guaranteed fix.

## The Solution: Static Polymorphism (CRTP)

If we want the flexibility of an interface without the runtime cost, we can use the **Curiously Recurring Template Pattern (CRTP)**.

[Here is the optimized example code](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIMwBspK4AMngMmAByPgBGmMT%2BAMykAA6oCoRODB7evv5BaRmOAmER0SxxCWbJdpgOWUIETMQEOT5%2BgbaY9sUMjc0EpVGx8Um2TS1teZ0KE4PhwxWj1QCUtqhexMjsHOaJ4cjeWADUJoluTrPEmKxn2CYaAIJ7B0eYp%2BdX4cB3D89PBEwLBSBkBHzchyYCgUxwAKr8npDoccLFDMAARJhNU4AdisT2OhJSXhitDwyBAf0JhNm6BAIC%2BRmOKWI4QIEBWxzQDFmuPxj2pguO1wImwYx1mWPJAH1RLMzhCBLzYQAqO4QAgIPAKFYAWjuXiUxGlLLZHLO/MFJhx6L%2B1ttiX5fyRMIAkoIzHyqUSSWSKd7Ce6CGYOccQMcAG5iLyYCBcDSc61WG0BiUEOkMgispmmwSh7m8pPCzCi4ji2n0ojSxnACBR7yYFYW3G2p4B01RwGUgnUtmR6OYC12lOOu2IgzIoOJMPM33klFozFNBVTu5enuz0nk7sC6lT0Ph%2BsxuMJr0pjdpjM146G%2BIm7PszkFgh84ul8vpyuoatZ751gdNo6LZjruzKsp27Cpn2R6DqObYjk6TwAPRIccQisO8UJcmIZJMjEaJchOMJUF4DD1AIxwAO6EAgkZ4C0XhiMcJFkb0Ch/ICwKgu8CounCCKPBWmbZsAYFstKqAxNodTss%2BC5KEuTAKvCiT3IExxMImeKpiKYoaQAdLmj5DvBrb/I8fYsEw4ShkmqZThpiQmaBxxBp6TBmM5gpCWgXgvgqCoSngABemCoFQEBMIkibnIFQmuLQXnUj5Gz%2BbF5xBaF4WRWYMVuHFn4gAlSU0oVvlpflGVRYZD62elbiXvSxVwS5KV%2BeCgUeTVZp5QVGbNZa1Iod5ZWpR1GVGRJUkyTlvUZfFDDoIlQHDdgxDECQqZtRVgWTZJ0kOJF0XjQ1C1LSVb56Rozn2hway0JwACsvB%2BBwWikKgnD5ZY1gShsWw8dUPCkAQmh3WsADWICJIk%2Bkw/DCOI0ED0cJIL1gx9nC8AoIAaCDYNrHAsBIGgwJ0PE5CUKTKTkwkwBSGYfB0ICxA4xAMQYzE4TNAAnpwwNc8wxA8wA8tNDj87wpNsIIIsMLQfNvbwWAxF4wBuDhOPcMrQKGMA4hK6Q%2BDXPUEaYFr72YKodR%2BTs71st0GNkjExC8x4WAY7%2BLCS6QZvEDE6QYrrRi4aAStrFQBjAAoABqeCYJRIspIwPv8IIIhiOwUgyIIigqOohu6Fw%2Bh6yg1jWPoeAxDjkBrKgKRsVjH1%2B6yWA1xyXQ9FkLiLVMfjF6ECzlJUeiFJkAh96P6TjwwQzD6Mxe1ORfRzJPi/dDJDRzHPIwJIvq%2BeO0eiSi0O9LHvawKP92wSPdT3o4bn0cMcqgABwBLqASSMcwDIMgxxSH0p6CAuBCAkFOEDFYvBQbh0htDWGiNEHw2RpwNGpBXrvSftjXG%2BNw6kCJogEAqViQEEphAamtNIgYU4G/D%2BX8f5/wAZIIBvBMD4CIK3PQadhCiHENnbhec1AYyLqQSirsUiSzvhwZ66CMZPxFn5Ehxxwov3fp/b%2Bv9/6AOAR4Mm9BiAQMSFwKBuCtArDgTDOGSDEH6FQQ/TBTccZ4xgWYqRZh7G8CwaY8Gvt4gZGcJIIAA%3D)

In the CRTP example, `Int3` inherits from `BaseData<Int3>`. The base class casts this to the derived type at compile time to call the implementation:

```cpp
template<class T>
struct BaseData {
    std::string print() const {
        return static_cast<const T*>(this)->user_print();
    }
};
```

The assembly code for `Int3` is now identical to `Int2`. It consumes only 4 bytes and involves no pointer chasing.

### A Warning on Recursive Calls

You might notice that I named the implementation function `user_print()` instead of `print()`.

In CRTP, it is best practice to distinguish the **interface** (in the base class) from the **implementation** (in the derived class).

If the derived class fails to implement `user_print()`, you get a clear compile-time error. However, if both functions were named `print()`, and the derived class forgot to implement it, the base class `print()` would essentially call itself (since `static_cast<Derived*>(this)->print()` would resolve back to `BaseData::print()` via inheritance). This causes **infinite recursion**, leading to a stack overflow/segmentation fault.

## Generic Programming

CRTP is the foundation of **generic programming** in C++. Instead of relying on a common runtime type, we rely on a common interface.

We can write a function that accepts any type inheriting from `BaseData`:

```cpp
template<class T>
std::string print_object(const BaseData<T>& a) {
  return a.print();
}
```

Here, the compiler determines the type `T` at compile time. We get the safety and structure of an interface, but with zero runtime cost.
