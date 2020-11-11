---
layout: post
title: "ELF Executable File Format"
description: "How Compiled C Binaries are Structured"
tags: [cpp, compilers]
---

As part of my [autonomous drone
project](https://www.taylortechblog.com/posts/drone-project-intro), I am writing
custom C++ firmware to run the drone itself. One of my goals with this firmware
is to rely on very few external and vendor-provided libraries. This necessitates
diving deep into the startup sequence and peripheral details of the
microcontroller I'll be using (a STM32F103C8T6). This, in turn, requires some
knowledge about ELF, the Executable and Linkable Format.

## What is ELF?

ELF is an executable (or object) file format. It defines a common specification
for how executable files should be organized such that a processor can
understand them properly. To be more specific, the compiled binary emitted at
the end of a C++ compilation process is an ELF file.

[ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) is one of
many such object file formats. It is part of the System V ABI, which is the
standard application binary interface used by most Unix-like operating systems
today (Linux, BSD, etc.). It is widely adopted, finding additional use in
Android mobile devices and many popular video game consoles. Other object file
formats include [PE](https://en.wikipedia.org/wiki/Portable_Executable) on
Windows, [Mach-O](https://en.wikipedia.org/wiki/Mach-O) on macOS, and
[COFF](https://en.wikipedia.org/wiki/COFF) as an alternative on Unix-like
systems.

## ELF Format File Layout

Let's discuss how ELF files are actually laid out. At a high level, ELF files
have four main sections:

* ELF header: Stores metadata about the executable (endianness, target hardware architecture, entry point address, etc.)
* Program header table: Specifies memory segments
* Sections: Store different parts of the process image
* Section header table: Defines sections

#### Sections

* .text: Program code
* .data: Initialized data
* .rodata: Initialized read-only data
* .bss: Zero-initialized data
* .got:
* .symtab:
* .dynsym:

#### Segments

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

## Example: Building and Examining a Simple ELF File

## References
* [Executable and Linkable Format (ELF)](http://www.skyfree.org/linux/references/ELF_Format.pdf)
* [Executable and Linkable Format 101 - Part 1 Sections and Segments (Ignacio Sanmillan)](https://www.intezer.com/blog/research/executable-linkable-format-101-part1-sections-segments/)
* [Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [Object File](https://en.wikipedia.org/wiki/Object_file)
* [Comparison of executable file formats](https://en.wikipedia.org/wiki/Comparison_of_executable_file_formats)
* [Itanium C++ ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html)
* [System V Application Binary Interface](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)
* [ELF for the Arm Architecture](https://developer.arm.com/documentation/ihi0044/h/)
* [Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)
* [ELF101 a Linux executable walkthrough (Ange Albertini)](https://upload.wikimedia.org/wikipedia/commons/e/e4/ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)
* [Tool Interface Standard (TIS) Executably and Linking Format(ELF) Specification](https://refspecs.linuxbase.org/elf/elf.pdf)
* [System V Application Binary Interface](http://www.sco.com/developers/devspecs/gabi41.pdf)
* [System V ABI](https://wiki.osdev.org/System_V_ABI)
