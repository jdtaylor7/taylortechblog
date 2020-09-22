---
layout: post
title: "Macros in C and C++"
description: "TODO"
tags: [cpp, compilers]
---

While a quite divisive topic in C++, macros are still widely used in legacy code
and for niche cases in modern code. As such, understanding them is still
necessary in C++.

This article will cover how macros work and the basics of using them in your
code. How macros have evolved with C++ will also be discussed. I've also
included a short nugget at the end detailing how to use the preprocessor
directly to see the output of macro replacements yourself, a topic rarely
discussed.

I won't be talking about some of the finer details of macros or when you
should/shouldn't use them. If you'd like to see some common C++ use cases,
Jonathan Boccara gives great examples in his article [*3 Types of Macros That
Improve C++
Code*](https://www.fluentcpp.com/2019/05/14/3-types-of-macros-that-improve-c-code/).

In general, most of the examples I present in this article are not considered
the best applications of macros. That being said,

## What are Macros?

A macro is an identifier that represents some expression. The expression,
sometimes called a "replacement list", can be any snippet of code.

All instances of a macro are replaced (expanded) by the preprocessor early in
the compilation process (specifically during [phase
4](https://en.cppreference.com/w/cpp/language/translation_phases) in both C and
C++). The `#define` preprocessor directive is used to create macros. The basic
syntax is:

`#define my_macro replacement list`

The replacement list includes all content until the end of the line. This can be
extended to multiple lines by placing a backslash at the end of the line:

{% highlight plaintext %}
#define MY_FRUIT PINEAPPLE, \
                 PEACH, \
                 MANGO
{% endhighlight %}

Note that when the macro is expanded, the replacement list will all be put on
one line.

In addition to being *defined*, macros can be *undefined*. This is the only way
to apply to scope to macros.

{% highlight cpp %}
std::string x;
x = FANCY_MACRO;
#define FANCY_MACRO "chartreuse"
x = FANCY_MACRO;
#undef FANCY_MACRO
x = FANCY_MACRO;
{% endhighlight %}

The above code would be expanded as follows:

{% highlight cpp %}
std::string x;
x = FANCY_MACRO;
x = "chartreuse";
x = FANCY_MACRO;
{% endhighlight %}

This wouldn't compile since `FANCY_MACRO` is an undefined symbol in the first
and third instances.

The replacement list doesn't even have to be valid code, technically. Macros are
expanded during preprocessing, before the compiler sees the code. Therefore, if
a macro with an invalid definition isn't actually used after being defined, the
compiler would never see the invalid definition. The definitions of all expanded
macros, on the other hand, will be seen by the compiler.

## C/C++ Compilation Process

As it turns out, the "C preprocessor" is a separate tool which, just like a
linker, is called automatically by the compiler during the compilation process.
It is also compatible with multiple languages in the C family: namely C, C++,
Objective-C, and Fortran.

GCC's preprocessor is called "CPP" (confusingly enough) and is packaged as a
separate binary. You can run this tool manually to see the effects of
preprocessing firsthand if you like, which I discuss
[below](#running-the-preprocessor-manually).

Tokens vs. preprocessing tokens

Stage 4 of the compilation process

Link to [cppreference here](https://en.cppreference.com/w/c/language/translation_phases)

## Two Types of Macros: Object- and Function-like

There are two overarching types of macro: object-like and function-like.

#### Object-like macros

Macros which take no arguments are called "object-like". These are often used to
store constants, as below:

{% highlight cpp %}
#define IMPORTANT_NUMBER 627
{% endhighlight %}

#### Function-like macros

Conversely, macros which do accept arguments are called "function-like". They
work similarly to regular functions, as below.

{% highlight cpp %}
#define ADD(x, y) x + y
int z = ADD(2, 3);
{% endhighlight %}

The above code snippet would be converted into:

{% highlight cpp %}
int z = 2 + 3;
{% endhighlight %}

Note that the semicolon is excluded in the macro definition and thus must be
included in the call to the macro.

*Variadic macros* are function-like macros which accept a variable number of
arguments. Here's the example used by the [GNU CPP
page](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html#Variadic-Macros):

{% highlight cpp %}
#define eprintf(...) fprintf(stderr, __VA_ARGS__)
{% endhighlight %}

The `__VA_ARGS__` identifier represents all of the arguments passed into the
macro.

Named arguments can be included in variadic macros. A custom variadic identifier
can be specified in place of `__VA_ARGS__` as well. Here's an updated example of
`eprintf`:

{% highlight cpp %}
#define eprintf(format, args...) printf(stderr, format, args)
{% endhighlight %}

As in the examples shown, variadic macros are often used to extend `printf`.

There are some additional points involving variadic macros which we won't get
into here. See the [GCC documentation](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html#Variadic-Macros) for more details.

## Tokens and Preprocessor Tokens

Before discussing the # and ## operators in the next section, it's useful to
first explain the concepts of tokens and preprocessor tokens.

In computer science, a "token" is the smallest piece of a program which is
meaningful to a given language's compiler. Take the following line of C++ code:

{% highlight cpp %}
int foo = (5 + 8) * 3;  // foo is an original name
{% endhighlight %}

After this line is converted into tokens (tokenized), the resulting array of
tokens would look as follows:

{% highlight plaintext %}
[int, foo, =, (, 5, +, *, ), *, 3, ;]
{% endhighlight %}

This set of tokens is passed to the next stage of the compiler for further
processing. Note that, while used for demarcating tokens, whitespace characters
themselves are not tokens. Comments are also not tokens.

Now, while each language has its own set of tokens, C and C++ also have sets of
*preprocessor tokens*. This means that the preprocessor interprets source code
differently than the actual compiler.

So here's what happens. When your C/C++ code is compiled, it is first parsed by
the preprocessor. The preprocessor converts all of the code into preprocessor
tokens. It then applies some changes ("transformations") to the source code
based on these preprocessor tokens. After these transformations are applied, the
resulting source code is converted into tokens for further processing.

To make this crystal clear, let's run through an example.

{% highlight cpp %}
#define MY_NUM 6
int x = 10 - MY_NUM;
{% endhighlight %}

The preprocessor would parse this code into the following preprocessor tokens:

{% highlight plaintext %}
[#define, MY_NUM, 6, int, x, =, 10, -, MY_NUM, ;]
{% endhighlight %}

Now that the preprocessor has generated preprocessing tokens, it must transform
the code based on the preprocessor directives. The #define directives indicate
that macro substitution should occur, so that transformation is applied.
Afterwards, the preprocessor directives are removed from the code. That leaves
us with the following preprocessed code:

{% highlight cpp %}
int x = 10 - 6;
{% endhighlight %}

As you can probably guess, the tokens resulting from this code are:

{% highlight plaintext %}
[int, x, =, 10, -, 6, ;]
{% endhighlight %}

Again, these are regular tokens ("tokens"), *not* preprocessing tokens.

As we can see, only the preprocessor is used to interact with preprocessor
directives, so the code for processing tokens does not need to recognize
preprocessor directives at all.

With these definitions in mind, let's now talk about the preprocessor operators
\# and \#\#.

## Preprocessor Operators: # and \##

In the examples we've seen above, macro substitution causes the compiler to
recognize the replaced text as a set of tokens, without any extra complexity.
This is great for many applications, but sometimes we may want a bit more
control over this process. The single hash (#) and double hash (##) operators
allow us to manipulate how the preprocessor turns the replacement text into
tokens.

#### Single hash operator #: Stringizing



#### Double hash operator \##: Token concatenating



## C++ Differences: TODO keep?

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
