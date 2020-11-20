---
layout: post
title: "How to Write Linker Scripts"
description: "Specifying the Memory Layout of C Programs"
tags: [cpp, compilers]
---

Intro

## What Does the Linker Do?

## Sections and Segments and Memory

Graphic representations of the sections and memory regions, and how segments
are used to bring them together

## How Linker Scripts are Used

## Linker Script Basics

## Why Don't We Always Write Them Manually?

Desktops. How to find

Default linker script part of output from `gcc -Wl,-verbose`

For embedded processors, often included in vendor IDEs. For example...

## References
* [Executable and Linkable Format 101 - Part 1 Sections and Segments (Ignacio Sanmillan)](https://www.intezer.com/blog/research/executable-linkable-format-101-part1-sections-segments/)
* [Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)
* [ELF101 a Linux executable walkthrough (Ange Albertini)](https://upload.wikimedia.org/wikipedia/commons/e/e4/ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)
* [STM32 from scratch, the bare minimum](http://tty.uchuujin.de/2016/02/stm32-from-scratch-bare-minimals/)
