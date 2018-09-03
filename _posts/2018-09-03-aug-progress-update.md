---
layout: post
title: "August 2018 Progress Report"
date: 2018-09-03 08:09:10 -0400
categories: "Progress Report"
author: "Jeff Walker"
---

The Adamant language has been under development in one form or another since at least August 2016. However, it wasn't until May 2018 that development picked up. In June 2018, additional hours began to be committed to development each week. In all that time, the language, compiler, and website progressed considerably. However, that progress was not documented and shared outside of the commits on the compiler and docs. This is the first of what will hopefully be more frequent updates on progress on the Adamant language. As such, it summarizes all the work to date.

### Current Status

The Adamant language currently exists mostly as a set of design ideas. Many aspects of this are documented, but other portions remain mostly undocumented. Generally, those portions which more closely follow the precedent set by other languages are the ones that are not well documented. Someone familiar with C# should be able to infer how many of them work. For Adamant to be successful, it needs design, tools, and documentation. The current state of each of these is:

* **Language Design:**

  The language design has been relatively stable. Semantically, the most significant change has been the introduction of more compile-time code execution. Over time, there has been some drift in the syntax from Adamant’s C family heritage. These changes have been made to improve readability and consistency and make good use of the easily typed characters. The rules for borrow checking variables and references were clarified, but there is still uncertainty about how object fields should be handled. Reference counting will be supported and should be sufficient if no further borrow checking rules are added. The areas of the language that still need better design and more thought are mainly asynchronous support, effect typing, error handling, compile-time code execution and advanced generic types.
* [**Compiler:**](https://github.com/adamant/adamant.tools.compiler.bootstrap)

  The current compiler is implemented in C# with .NET core. This is for rapid development and because C# still allows execution on multiple platforms. The back end emits C code that can be compiled natively using [clang](https://clang.llvm.org). The compiler supports only a minimum subset of the language. The focus has been on establishing the framework to support basic functionality like type checking and borrow checking.
* **Documentation:**
  * [**Language Reference:**](https://github.com/adamant/adamant.language.reference/blob/master/book.md)

    The language reference replaces a previous introductory book on the language. It is easier to write, maintain, and update the more succinct reference. Some portions of the language are nearly completely documented. Others have only the most cursory of coverage. There is a section of ideas that are being considered. In some cases, an idea has been accepted, but the reference has yet to be updated to reflect it.
  * [**Design Notes:**](https://github.com/adamant/adamant.language.design/blob/master/book.md)

    The language reference, by necessity, leaves out information about why certain design decisions were made. The design notes describe some of these as well as laying out design principles to be followed in the design of the rest of the language. Sections are added to this document as the issues are encountered during language design.
* [**Website:** adamant-lang.org](https://adamant-lang.org)

  The website has been up for some time now. Content is minimal, but it does serve as an entry point to learn about the language. The most useful section is probably the links to the docs. The landing page and tagline need to be improved to focus the marketing message.
* [**Blog:** blog.adamant-lang.org]()

  This post is the first on the blog. With the publishing of this post, the blog has been added as an item in the menu of the main page. Further design work is needed, but the design is sufficient for now. Planned topics include progress updates, language design theory, language critique and evaluation, and discussion of Adamant.

### Deprecated Compiler Attempts

The Adamant language has gone through a number of attempts at writing a compiler before the current one was started. The goal was to find an approach that would be easy and not waste too much effort writing a bootstrap compiler that would be discarded after the language switched to a self-hosted compiler. These early compilers can be found on GitHub under the [adamant-deprecated organization](https://github.com/adamant-deprecated). The earlier attempts were:

1. The [Adamant Bootstrap Compiler](https://github.com/adamant-deprecated/AdamantBootstrapCompiler) was an attempt to create and Adamant to C# translator that did not perform type checking, borrow checking or significant code transformations. This was not possible due to the significant semantic differences between the languages.
2. The [Adamant Temporary Compiler](https://github.com/adamant-deprecated/AdamantTemporaryCompiler) was an attempt to create a full ecosystem of compiler generation tools in C# that could be easily translated to Adamant later. This was because Adamant does not have any of those tools and they will be needed eventually. This approach was too much work and prevented progress on the language itself. It did not enable rapid prototyping and iteration.
3. The [Adamant Exploratory Compiler](https://github.com/adamant-deprecated/AdamantExploratoryCompiler) was an attempt to write a compiler for the full language in C# using the [ANTLR 4](http://www.antlr.org) compiler tools. This approach was still found to be very heavyweight as each feature required changes throughout many layers and the tree produced directly by ANTLR was not ideal.
4. The [Pre-Adamant Translator](https://github.com/adamant-deprecated/PreAdamant.Translator) was an brief attempt to create a [Nanopass compiler framework](http://nanopass.org/) for C# and use it to write a compiler for a subset of Adamant.
5. The [Pre-Adamant Compiler](https://github.com/adamant-deprecated/PreAdamantCompiler) was to be a compiler in C# for a tiny subset of the Adamant language meant to minimize the time until a working compiler was complete.
6. The [adamant.tools.compiler](https://github.com/adamant-deprecated/adamant.tools.compiler) was an attempt to skip directly to a self-hosting compiler through a minimal C++ bootstrap and a scheme of simple translation to C/C++.

After the failure of the first five attempts, it was decided that creating a self-hosting compiler directly would be more beneficial. This sixth attempt would allow the developer to get experience working in a subset of Adamant from very early which would help to inform the design of the language. It would also prevent wasted effort on a bootstrap compiler written in another language. This compiler started as some C++ to perform a very basic lexing and recursive descent parsing to translate Adamant to C++. It quickly reached a point where it could translate a subset of Adamant that was equivalent to the subset of C++ it was  written in. The compiler code was re-written in Adamant, and the compiler was "self-hosting." However, it didn't do most of what a compiler should do. Rather, it was a simple translation to a subset of C++. This version of the compiler progressed rapidly at first. Eventually, a transition was made from translating to C++ to translating to C. However, it eventually reached a catch-22 point. Functionality like type checking, borrow checking, classes, enums, and other abstraction facilities needed to be implemented. However, the lack of those features was making development incredibly difficult. It was eventually decided that there didn't seem to be a way forward. That is when the current compiler was started.

### [Current Compiler](https://github.com/adamant/adamant.tools.compiler.bootstrap)

Taking the lessons from the previous attempts, the [adamant.tools.compiler.bootstrap](https://github.com/adamant/adamant.tools.compiler.bootstrap) project was started. This compiler is written in C# but focuses on a small subset of Adamant using a minimalist, simple approach similar to the "self-hosting" compiler. It does not yet support most of the language features the last compiler did. Instead the focus has been on laying the groundwork for important features like type checking and borrow checking rather than filling out the range of core features like control flow, expressions, and primitive types.

### August 2018

Progress on Adamant in August of 2018 consisted mostly of:

* Refactoring the compiler to use a more attribute grammar based approach.
* Realizing that some ideas for the borrow checker around object fields weren't going to work.
* Reading academic literature trying to find a solution. Discussion of "ownership types" was found which gets part of the way. However, the academic literature isn’t trying to solve the problem as Adamant, and it wasn’t much help. This problem was set aside to make progress on other things. Even if this issue isn't resolved, Adamant can still be a great language.
* Learning the details of how the Rust borrow checker works.
* Deciding details of how borrow checking would work in Adamant.
* Researching the implementation of the Rust borrow checker.
* Adding an intermediate representation (IR) layer to the compiler to aid in implementing borrow checking. This is similar to the MIR step in the Rust compiler.
* Starting to implement borrow checking

### Next Steps

The next focus will be implementing basic borrow checking for variables, parameters, and functions.
