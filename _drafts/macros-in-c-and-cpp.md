---
layout: post
title: "Macros in C and C++"
description: "TODO"
tags: [cpp]
---

While a quite divisive topic in C++, macros are still widely used in legacy code
and for niche cases in modern code. As such, understanding them is still
necessary in C++.

This article will cover how macros work and the basic of using them in your
code. How macros have evolved with C++ will also be discussed. I've also
included a short nugget at the end detailing how to use the preprocessor
directly to see the output of macro replacements yourself, a topic rarely
discussed.

I won't be talking about some of the finer details of macros or when you
should/shouldn't use them. If you'd like to see some common C++ use cases,
Jonathan Boccara gives some great examples in his article [*3 Types of Macros
That Improve C++
Code*](https://www.fluentcpp.com/2019/05/14/3-types-of-macros-that-improve-c-code/).

## What are Macros?

A macro is an identifier that represents some expression. The expression,
sometimes called a "replacement list", can be any snippet of code.

All instances of a macro are replaced by the preprocessor early in the
compilation process (specifically during [phase
4](https://en.cppreference.com/w/cpp/language/translation_phases) in both C and
C++). The `#define` preprocessor directive is used to create macros. The basic
syntax is:

`#define my_macro replacement list`

As you can see, the replacement text can include any

## C/C++ Compilation Process

As it turns out, the "C preprocessor" is a separate tool which, just like a
linker, is called automatically by the compiler during the compilation process.
It is also compatible with multiple languages in the C family: namely C, C++,
Objective-C, and Fortran.

GCC's preprocess is called "Cpp" (confusingly enough) and is packaged as a
separate binary. You can run this tool manually to see the effects of
preprocessing firsthand if you like, which I discuss
[below](#running-the-preprocessor-manually).

Tokens vs. preprocessing tokens

Stage 4 of the compilation process

Link to [cppreference here](https://en.cppreference.com/w/c/language/translation_phases)

## Two Types of Macros: Object- and Function-like

#### Object-like macros

#### Function-like macros

Briefly mention variadic macros and than link to the [GCC
documentation](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html#Variadic-Macros)
about them.

## Preprocessor Operators: # and ##

#### Single hash sign: Stringizing

#### Double hash sign: Token concatenating

## C++ Differences

## Running the Preprocessor Manually

Can be done with either:

`gcc -E <input> -o <output>`

or

`cpp <input> -o <output>`

Input code:
{% highlight cpp %}
#include <stdio.h>

#define CUSTOM_MACRO_VAL 7

int main()
{
    printf("value = %g\n", CUSTOM_MACRO_VAL);  // comment!
    return 0;
}
{% endhighlight %}

Output code (snippet):
{% highlight cpp %}
int main()
{
    printf("value = %g\n", 7);
    return 0;
}
{% endhighlight %}

Removed preprocessor directives and comments, replaced macro with correct value.

Lots of other stuff included in the output file like all included file names,
source code from included files, diagnostic info, etc. But some of this output
can be culled and it can be useful regardless, especially since file names and
line numbers are included by default.

## Conclusion

## References
* [GCC documentation](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html#Variadic-Macros)
* [cppreference page](https://en.cppreference.com/w/cpp/preprocessor/replace)
* [Jonathan Boccara: 3 Types of Macros That Improve C++ Code](https://www.fluentcpp.com/2019/05/14/3-types-of-macros-that-improve-c-code/)
* [Eli Bendersky: Parsing C++ in Python with Clang](https://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang)
