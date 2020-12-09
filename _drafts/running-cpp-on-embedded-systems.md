---
layout: post
title: "Running C++ on Embedded Systems"
description: "C++ in a Bare Bones Environment"
tags: [cpp, compilers, embedded, hardware]
---

Intro

## How C++ Binaries are Structured

## Hosted and Freestanding Execution Environments

## Linker Scripts

## Bootloaders

## Strategies to Get C++ Working
* Heap setup
    * Ban heap usage
    * Allow normal heap usage - can lead to heap fragmentation down the line
    * Use non-fragmenting allocator
    * Allow normal heap usage at initialization, then ban during runtime
    * Custom heap
        * Free lists
        * Periodic reset
        * Pool allocator
        * etc.
* Utilize ROM where possible
    * Use const static data and static initializers
    * Classes can achieve this with some magic
* Evaluate memory footprint of libraries before committing to them
    * Esp. std::string, containers, std::ostream, etc.
    * Also things that throw exceptions
* Can use some libraries without issue
    * std::algorithms

## C++ Features to Use
* Function overloading
* References
* Namespaces
* Inlining
* Operator overloading
* auto
* decltype
* override, final, =delete, =default
* Range-based for loops
* Suffix return types
* std::initializer_list
* Move constructors
* noexcept
* Digit separators and binary literals
* Function return type deduction (auto return type)

## C++ Features to Consider
* Simple classes (if constructors/destructors are figured out)
* New/delete (if heap management utilized)
* Single, non-virtual inheritance
* Virtual functions
* Templates
* Lambdas
* constexpr (since expression may not be evaluated at compile time)

## C++ Features to Avoid
* Exceptions
* Runtime type information

## C++17/20 Features?
Haven't looked into these for embedded yet

## Useful Compiler Flags

[C++ On Embedded Systems](https://bitbashing.io/embedded-cpp.html)


## References
* [Main function - cppreference](https://en.cppreference.com/w/cpp/language/main_function)
* [Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)
* [stm32-from-scratch](https://github.com/textshell/stm32-from-scratch)
* [C++ On Embedded Systems - Matt Kline](https://bitbashing.io/embedded-cpp.html)
* [From Zero to main(): Bare metal C - François Baldassari](https://interrupt.memfault.com/blog/zero-to-main-1)
* [Modern C++ in embedded systems – Part 1: Myth and Reality - Dominic Herity](https://www.embedded.com/modern-c-in-embedded-systems-part-1-myth-and-reality/)
* [Modern C++ in embedded systems - Part 2: Evaluating C++](https://www.embedded.com/modern-c-embedded-systems-part-2-evaluating-c/)
* [C99 Standard Draft N1256](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf)
* [C++11 Standard Draft N3337](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf)
* [Effective C++ in an Embedded Environment](https://www.artima.com/shop/effective_cpp_in_an_embedded_environment)
