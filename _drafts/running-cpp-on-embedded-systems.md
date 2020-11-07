---
layout: post
title: "Running C++ on Embedded Systems"
description: "C++ in a Bare Bones Environment"
tags: [cpp, embedded, hardware]
---

Intro

## How C++ Binaries are Structured

Executable (object) file formats:
* [COFF](https://en.wikipedia.org/wiki/COFF)
* [Mach-O](https://en.wikipedia.org/wiki/Mach-O) (macOS)
* [PE](https://en.wikipedia.org/wiki/Portable_Executable) (Windows)
* [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) (Linux, many other systems)

## ELF Format File Layout

## Hosted and Freestanding Execution Environments

## Linker Scripts

## Bootloaders

## References
* [Main function - cppreference](https://en.cppreference.com/w/cpp/language/main_function)
* [Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)
* [stm32-from-scratch](https://github.com/textshell/stm32-from-scratch)
* [C++ On Embedded Systems - Matt Kline](https://bitbashing.io/embedded-cpp.html)
* [From Zero to main(): Bare metal C - François Baldassari](https://interrupt.memfault.com/blog/zero-to-main-1)
* [Modern C++ in embedded systems – Part 1: Myth and Reality - Dominic Herity](https://www.embedded.com/modern-c-in-embedded-systems-part-1-myth-and-reality/)
