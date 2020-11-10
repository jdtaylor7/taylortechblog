---
layout: post
title: "ELF File Format"
description: "How Compiled C Binaries are Structured"
tags: [cpp, compilers]
---

Intro

## What is an Executable File Format?

Executable file formats:

* [COFF](https://en.wikipedia.org/wiki/COFF)
* [Mach-O](https://en.wikipedia.org/wiki/Mach-O) (macOS)
* [PE](https://en.wikipedia.org/wiki/Portable_Executable) (Windows)
* [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) (Linux, many mobile phones and game consoles)

## ELF Format File Layout

* ELF header: Stores general information about the executable (endianness,
target OS ABI, entry point address, etc.)
* Program header table: Specifies how and where to load memory segments
* Section header table: Defines the sections
* Sections: Store different parts of the process image

## Common Section Names

* .text: Program code
* .data: Initialized read-write data
* .rodata: Initialized read-only data
* .bss: Zero-initialized data

## How are ELF Files Built?

In general, desktop application programmers don't have to worry about how
binary files are built. That's the whole purpose of the compiler toolchain! That
being said, some environments may require extra work from the developer.
Embedded environments are one such instance.

The linker is responsible for the layout of the final executable file. The
linker follows the directives in a *linker script* to accomplish this.

Linker scripts will be the focus of our next post.

## Tools for Parsing ELF Files

* `readelf`
* `objdump`
* `nm`

## References
* [Executable and Linkable Format 101 - Part 1 Sections and Segments (Ignacio Sanmillan)](https://www.intezer.com/blog/research/executable-linkable-format-101-part1-sections-segments/)
* [Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [Object File](https://en.wikipedia.org/wiki/Object_file)
* [Comparison of executable file formats](https://en.wikipedia.org/wiki/Comparison_of_executable_file_formats)
* [Itanium C++ ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html)
* [System V Application Binary Interface](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)
* [ELF for the Arm Architecture](https://developer.arm.com/documentation/ihi0044/h/)
* [Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)
* [ELF101 a Linux executable walkthrough (Ange Albertini)](https://upload.wikimedia.org/wikipedia/commons/e/e4/ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)
