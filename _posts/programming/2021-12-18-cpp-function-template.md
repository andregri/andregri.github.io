---
layout: single
author: andrea
title: Where to implement C++ function templates
tags: cpp template generics
---

I was implementing a **template function** inside a class. As usual, I wrote the
prototype in .h file and the definition in a .cpp file. This templated method
is then called in other part of the software. When I hit the compile button,
the compiler complained about an **undefined reference** where I called the
template function.

Since the function was called from a pointer of the class, my first try was to
force the compilation of the the template function by defining a specialization
of it in the .cpp file. I compiled again but the error remained.

After a quick search on web I found the solution: defining the template function
immediately in the .h file because the Stackoverflow user's told that template
function declaration and definitions must stay in the same **translation unit**.

A translation unit is the result of the preprocessing stage and so the very first
for the compiler. A translation unit consists of a source file where `#include`
directives are literally included and macro are expanded. Now it is clear why it
was not working. In my `main.cpp` I was including just the .h file of the class
so the translation unit of `main.cpp` did not contain the definition of the
function template.

However, I wanted a deeper insight about function templates and indeed
[isocpp FAQ](https://isocpp.org/wiki/faq/templates#templates-defn-vs-decl) has a
huge section focused only for this kind of error.
It says that templates are not functions or classes but just a *pattern* used by
the compiler to generate a family of functions and classes.
C++ compiler is based on the **separate compilation model**, that is
while it is compiling one .cpp file, it is not required to remember the details
of another .cpp file. This is why it complained when I tried to separate the
template function from its definition.

Lesson learned. I conclude with these two useful rules:
1. Define template functions immediately after their declaration.
2. When C++ compiler is compiling a .cpp file, it is not required to use details
of another .cpp file.

## Resources

1. [Wikepedia definition of translation unit](https://en.wikipedia.org/wiki/Translation_unit_(programming))
2. [tutorialspoint brief definition of translation unit](https://www.tutorialspoint.com/What-is-a-translation-unit-in-Cplusplus)
3. [isocpp FAQ](https://isocpp.org/wiki/faq/templates)
4. [Official C++ reference for function templates](https://en.cppreference.com/w/cpp/language/function_template)
