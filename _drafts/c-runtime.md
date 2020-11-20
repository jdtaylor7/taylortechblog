---
layout: post
title: "The C Runtime"
description: "What's Behind the Curtain?"
tags: [cpp, compilers]
---

Intro

## ABI and EABI

* [C++ ABI for the Arm Architecture](https://developer.arm.com/documentation/ihi0041/g/?lang=en)
* [Itanium C++ ABI](https://itanium-cxx-abi.github.io/cxx-abi/abi.html#dso-dtor-motivation)

## The Actual C Runtime (CRT) Code

[How Initialization Functions Are Handled](https://gcc.gnu.org/onlinedocs/gccint/Initialization.html)

[GCC's crtstuff.c](https://github.com/gcc-mirror/gcc/blob/master/libgcc/crtstuff.c)

`crtstuff.c` gets compiled into various object files, which are then packaged
for each environment. The object files usually have names like `crtbegin*.o`,
`crtend*.o`, and `crtfastmath.o`.

Can see different CRT object files by searching for "crt" in the package lists
for various Linux distributions ([Arch
Linux](https://www.archlinux.org/packages/core/x86_64/gcc/) and
[Ubuntu](https://packages.ubuntu.com/bionic/amd64/libc6-dev/filelist), for
example).

Can inspect `crt*.o` files with `objdump` (`-d`, `-D`, and `-t` options
helpful).

On my Cygwin install, `crt*.o` files located at
`/usr/lib/gcc/x86_64-pc-cygwin/<version>/`

## Program Entry Point: `main`

The intro of the [Wikipedia page](https://en.wikipedia.org/wiki/Entry_point)
entry is actually quite descriptive.

Essentially an entry point is where a program starts. A Python program starts at
the first line of the Python file. In C and C++, a program starts with a `main`
function. In both languages, though, execution technically starts a bit earlier.
This extra code is called a language's "runtime".

## What About the Standard Library?

The GNU C++ Standard library, libstc++, is distinct from the C runtime.

## References
* [Using the GNU Compiler Collection (CGG)](https://gcc.gnu.org/onlinedocs/gcc/)
* [CNU Compiler Collection (GCC) Internals](https://gcc.gnu.org/onlinedocs/gccint/index.html#Top)
* [The GNU C++ Library](https://gcc.gnu.org/onlinedocs/libstdc++/)
