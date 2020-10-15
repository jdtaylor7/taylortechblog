---
layout: post
title: "Macros in C and C++"
description: "How and Why They Work"
tags: [cpp, compilers]
---

While quite a divisive topic in the C++ community, macros are widely used in
legacy code and for niche cases in more modern code. As such, understanding them
is crucial to understanding both the preprocessor and C++ as a whole.

This article will cover how macros work and the basics of using them in your
code. Towards the end of the article I've also detailed how to use the
preprocessor directly to see the output of macro replacements, a topic rarely
discussed.

I won't be covering every tiny detail about macros or when you should/shouldn't
use them in C++. If you'd like to see some common C++ use cases, Jonathan
Boccara gives great examples in his article [*3 Types of Macros That Improve C++
Code*](https://www.fluentcpp.com/2019/05/14/3-types-of-macros-that-improve-c-code/).
The GCC preprocessor documentation also includes a section on [macro
pitfalls](https://gcc.gnu.org/onlinedocs/cpp/Macro-Pitfalls.html#Macro-Pitfalls).

In general, most of the examples I present here are not considered the best
applications of macros. That being said, they help illustrate many aspects of
macros in straightforward ways.

## What are Macros?

A macro is an identifier that represents some expression. The expression,
sometimes called a "replacement list", can be any snippet of code.

All instances of a macro are replaced (or *expanded*) by the preprocessor early
in the compilation process. This occurs specifically during [phase
4](https://en.cppreference.com/w/cpp/language/translation_phases) of compilation
in both C and C++. The `#define` preprocessor directive is used to create
macros. The basic syntax is:

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
to apply scope to macros.

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
compiler would never see the invalid definition. The definitions of all
*expanded* macros, on the other hand, will be seen by the compiler and therefore
must result in valid code.

## Phases of Translation

As it turns out, the "C preprocessor" is a separate tool which, just like a
linker, is called automatically by the compiler during the compilation process.
It is also compatible with other languages in the C family, namely C++,
Objective-C, and Fortran.

GCC's preprocessor is called "CPP" (C Preprocessor) and is packaged as a
separate binary. You can run this tool manually to see the effects of
preprocessing firsthand if you like, which I discuss
[below](#running-the-preprocessor-manually).

Without getting too far into the weeds, "compilation" as defined by the C/C++
standards includes multiple steps called *translation phases*. The most relevant
steps for this discussion of macros are when tokenization, preprocessing, and compilation all take place.

* Tokenization (of preprocessing tokens): step 3
* Preprocessing: step 4
* Compilation: step 7 (steps 7/8 in C++)

Tokenization technically occurs twice, once in step 3 and again in step 7 right
before compilation. More discussion about this
[below](#tokens-and-preprocessor-tokens).

The most important takeaway here is that initial tokenization occurs *before*
preprocessing, which occurs *before* compilation. Therefore, extraneous
whitespace and comments are all removed before the preprocessor ever sees the
code, and preprocessing occurs before the compiler ever sees the code. This is
important context which helps put preprocessing in perspective.

One last note, the specifications for C/C++ don't prescribe the implementation
of these translation phases, since that's not the goal of the specification
period. Therefore, there's no one way to apply these translation phases. One
tool could implement one phase each, one tool could implement all of the phases,
or anything in between. As long as the behavior mimics these phases, that's all
that matters. In GCC's case, the preprocessor we've been discussing (CPP), can
perform at least the first four phases, while GCC from a user's perspective
performs all of them (by calling the preprocessor, linker, and other
tools).

For more information about the translation phases, I would recommend looking
either at cppreference.com or directly at the drafts for the C/C++ standards:

* [cppreference.com for C](https://en.cppreference.com/w/c/language/translation_phases)
* [cppreference.com for C++](https://en.cppreference.com/w/cpp/language/translation_phases)
* [C99 Standard Draft N1256 (section 5.1.1.2)](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf)
* [C++11 Standard Draft N3337 (section 2.2)](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf)

## Two Types of Macros: Object- and Function-like

There are two overarching types of macros: object-like and function-like.

#### Object-like macros

Macros which take no arguments are called "object-like". These are often used to
store constants, as below:

{% highlight cpp %}
#define IMPORTANT_NUMBER 627
{% endhighlight %}

#### Function-like macros

Conversely, macros which do accept arguments are called "function-like". They
work similarly to regular functions.

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
docs](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html#Variadic-Macros):

{% highlight cpp %}
#define eprintf(...) fprintf(stderr, __VA_ARGS__)
{% endhighlight %}

The `__VA_ARGS__` identifier represents all of the arguments passed into the
macro, separated by commas.

Named arguments can be included in variadic macros. A custom variadic identifier
can also be specified in place of `__VA_ARGS__`. Here's an updated example of
`eprintf`:

{% highlight cpp %}
#define eprintf(str, args...) printf(stderr, str, args)
{% endhighlight %}

As in the examples shown, variadic macros are often used to extend `printf`.

There are some additional points involving variadic macros which we won't get
into here. See the [GCC documentation](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html#Variadic-Macros) for more details.

## Tokens and Preprocessor Tokens

Before discussing the # and ## operators in the next section, it's useful to
first explain the concepts of tokens and preprocessor tokens.

In computer science, a "token" is the smallest piece of a program which is
meaningful to a given language's compiler/interpreter. Take the following line
of C++ code:

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

So here's what happens. Recall the [above discussion](#phases-of-translation)
about translation phases. When your C/C++ code is compiled, it is first parsed
by the preprocessor. The preprocessor converts all of the code into
*preprocessor tokens*. It then applies some changes ("transformations") to the
source code based on these preprocessor tokens. After these transformations are
applied, the resulting source code is converted into *tokens* (**not**
preprocessor tokens) for actual compilation.

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
that a macro substitution should occur, so that transformation is applied.
Afterwards, the preprocessor directives are removed from the code. That leaves
us with the following preprocessed code:

{% highlight cpp %}
int x = 10 - 6;
{% endhighlight %}

As you can probably guess, the tokens resulting from this code are:

{% highlight plaintext %}
[int, x, =, 10, -, 6, ;]
{% endhighlight %}

Again, these are regular tokens, *not* preprocessing tokens.

As we can see, only the preprocessor is used to interact with preprocessor
directives, so the compiler does not need to recognize preprocessor directives
at all.

With these definitions in mind, let's now talk about the stringizing (#) and
token concatenation (\##) preprocessor operators.

## Preprocessor Operators: # and \##

In the examples we've seen above, macro substitution causes the compiler to
recognize the replaced text as a simple set of tokens. This is great for many
applications, but sometimes we may want a bit more control over this process.
The stringizing (#) and token concatenation (##) operators allow us to
manipulate how the preprocessor expands macros.

#### Stringizing Operator (#)

The # (single hash) operator allows us to *stringize* an argument in a
function-like macro. This converts the argument into a string constant. Instead
of interpreting the argument as a normal set of tokens, the entire argument is
converted into one string literal token. A simple example can be shown with
another `printf` macro:

{% highlight cpp%}
#define PRINT_RESULT(exp) printf(#exp " = ", exp)
PRINT_RESULT(2 + 3);
{% endhighlight %}

The code above would be preprocessed into the following:

{% highlight cpp %}
printf("2 + 3" " = ", 2 + 3);
{% endhighlight %}

Notice that the stringized argument ends up adjacent to the string literal that
was already present in the macro definition. This is the correct way of using
stringized arguments since adjacent string literal tokens are concatenated
before actual compilation (in translation phase 6).

#### Token Concatenation/Pasting Operator (\##)

While the single hash operator allows us to convert a token into a string
literal token, the double hash operator allows us to combine tokens. This is
called *token concatenation* (or *token pasting*).

As a simple example, token concatenation can be used to declare multiple
variables of the same type:

{% highlight cpp %}
#define THREE_INTS(name) int name##1, name##2, name##3
THREE_INTS(foo);
{% endhighlight %}

This generates the following code:

{% highlight cpp %}
int foo1, foo2, foo3;
{% endhighlight %}

Due to token pasting, `foo1`, `foo2`, and `foo3` are complete, valid tokens.

Token concatenation can also be used for more complicated tasks. One prime
example is creating [*X macros*](https://en.wikipedia.org/wiki/X_Macro), which
are macros that maintain two related lists. A common usage of this is to
reliably generate a list of strings corresponding to the enumerators in an
enumeration. Let's see this in action. Say we want to create an enumeration with
all possible color values for a given application. We also want to print these
options out to the user.

{% highlight cpp %}
#define COLOR_LIST \
    X(red), \
    X(blue), \
    X(green)

// Define the enum.
enum class colors {
#define X(name) COLOR_##name
COLOR_LIST
#undef X
};

// Define the list of strings.
std::vector<std::string> color_names = {
#define X(name) #name
COLOR_LIST
#undef X
};
{% endhighlight %}

Expanding the `COLOR_LIST` macro results in the following code:

{% highlight cpp %}
enum class colors {
#define X(name) COLOR_##name
X(red), X(blue), X(green)
#undef X
};

std::vector<std::string> color_names = {
#define X(name) #name
X(red), X(blue), X(green)
#undef X
};
{% endhighlight %}

Expanding the `X` macros would result in the final code:

{% highlight cpp %}
enum class colors {
COLOR_red, COLOR_blue, COLOR_green
};

std::vector<std::string> color_names = {
"red", "blue", "green"    
};
{% endhighlight %}

The stringization and token pasting operators can be used for many applications
of this concept.

#### Two Layers of Indirection

One important aspect of the stringization and token pasting operators is that
they don't expand macros themselves. As a result, an additional layer of
indirection must be used to handle all cases correctly. Let's see some examples.

{% highlight cpp %}
#define SIMPLE_STRINGIZE(x) #x
SIMPLE_STRINGIZE(hello)
{% endhighlight %}

As would be expected, the above yields "hello". The following example, however,
does not work as intended:

{% highlight cpp %}
#define BAD_STRINGIZE(x) #x
#define FOO hello
BAD_STRINGIZE(FOO)
{% endhighlight %}

It expands to "FOO".

This is the correct implementation which works in all cases:

{% highlight cpp %}
#define GOOD_STRINGIZE(x) ACTUAL_STRINGIZE(x)
#define ACTUAL_STRINGIZE(x) #x
#define FOO hello
GOOD_STRINGIZE(FOO)
{% endhighlight %}

This correctly expands to "hello".

`GOOD_STRINGIZE` expands the macro, while `ACTUAL_STRINGIZE` performs the
stringization. As mentioned, this same concept applies to the token pasting
operator as well.

## C/C++ Differences

For the most part, the C and C++ preprocessors are meant to work the same way.
While this isn't the case for every version of the two languages (the
preprocessor in general has evolved over time, for example with the inclusion of
variadic macros), there are equivalences between specific versions, i.e. C++03
matches C90 and C++11 lines up with C99.

Predefined macros, however, are slightly different between C and C++. The
complete list of non-optional predefined macros can be seen on the relevant
cppreference.com pages for the [C
preprocessor](https://en.cppreference.com/w/c/preprocessor/replace) and the [C++
preprocessor](https://en.cppreference.com/w/cpp/preprocessor/replace).

Additional macros are defined by the different compilers, but that's beyond the
scope of this articles. As one example, commonly predefined GCC macros can also
be found in the [GNU CPP
docs](https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html).

This all being said, the preprocessor is changing more alongside C++20. Modules
are introducing two new preprocessor directives, `import` and `export`, which
are set to revamp how C++ projects organize code, making the `#include`
directive obsolete. Other language features are being shifted away from the
preprocessor as well, as the `<source_location>` header will provide
alternatives to the `__FILE__` and `__LINE__` macros for debugging and tracing.

## Running the Preprocessor Manually

Running the preprocessor by itself is fairly straightforward. With GCC it can be
done with the following command, which stops compilation after the preprocessor
phase:

`gcc -E <input> -o <output>`

Alternatively, you can run the preprocessor (CPP) directly:

`cpp <input> -o <output>`

The resulting output file will be free of all preprocessing tokens, such as
preprocessor directives and macros. All macros will have been expanded and all
`#include` directives will have been substituted with the appropriate file.
Comments will also have been removed. Additional lines are added by the
preprocessor which specify included files and line numbers, along with some
other diagnostic information.

Because all necessary files will be included, this file will likely be quite
lengthy, with easily over 10,000 lines of code even for a simple C++ file making
use of the Standard Library. Thankfully, searching for the name of the original
source file along with the relevant line number makes finding expanded macros
quite easy.

## Conclusion

I'm hoping this article gives a good overview of not just *how* macros work, but
*why* they work and how they fit into the greater C/C++ compilation model. While
there are certainly myriad ways to misuse macros, they can still be used for
very concise and efficient solutions.

## References
* [GCC preprocessor documentation](https://gcc.gnu.org/onlinedocs/cpp/Variadic-Macros.html#Variadic-Macros)
* [cppreference.com: Preprocessor replacing text macros](https://en.cppreference.com/w/cpp/preprocessor/replace)
* [C99 Standard Draft N1256](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf)
* [C++11 Standard Draft N3337](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf)
* [Jonathan Boccara: 3 Types of Macros That Improve C++ Code](https://www.fluentcpp.com/2019/05/14/3-types-of-macros-that-improve-c-code/)
* [Eli Bendersky: Parsing C++ in Python with Clang](https://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang)
