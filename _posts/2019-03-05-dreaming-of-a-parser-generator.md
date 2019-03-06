---
layout: post
title: "Dreaming of a Parser Generator for Language Design"
date: 2019-03-05 21:30:00 -0500
tags: ["Compiler Tools", "Parsing", "Performance"]
author: "Jeff Walker"
---
Trying to design [a new programming language](https://adamant-lang.org/), I'm faced with the question of how to implement a parser for it. Various parser generators are available and support a soup of parsing algorithms including LL, LR, LALR, ALL(*), GLR, and others. Ideally, I could look at the parser generators for the language I'm working in and pick the one that suited my use case. Unfortunately, despite a lot of searching, I've never found one that works well for language design or for that matter production compilers. Instead, I've ended up implementing a recursive descent parser that switches to precedence climbing for expression parsing. Approaches like that are prevalent in both the programming language design community and production compilers. If computer science has extensively studied parsing algorithms and many parser generators have been written, how come they aren't used? Indeed, parsing is often treated as if it is a "solved problem." What gives?

I'm not the first person to notice this contradiction. In ["Parsing: The Solved Problem That Isn't"](https://tratt.net/laurie/blog/entries/parsing_the_solved_problem_that_isnt.html) (2011), Laurence Tratt survey's the algorithms available and explains why he finds them inadequate. Likewise, Parr and Fisher open their paper ["LL(\*): The Foundation of the ANTLR Parser Generator"](https://www.antlr.org/papers/LL-star-PLDI11.pdf) (2011) by saying "Parsing is not a solved problem, despite its importance and long history of academic study." Then again in ["Adaptive LL(*) Parsing: The Power of Dynamic Analysis"](https://www.antlr.org/papers/allstar-techreport.pdf) (2014) Parr, Harwell, and Fisher begins "Computer language parsing is still not a solved problem in practice, despite the sophistication of modern parsing strategies and long history of academic study." They identify some of the same issues I'll be discussing. However, I have a few additional concerns. In particular, I believe compiler error handling needs to be significantly improved. That is discussed at the end of this post.

As I said, I'm approaching the problem of parsing from the perspective of a programming language designer. There isn't a single best parsing algorithm or parser generator for all use cases. My purpose is to lay out what I think a language designer needs from a parser generator. To a lesser extent, I also discuss DSLs. This means I'm not concerned with any of the following:

* parsing binary files
* parsing all existing programming languages
* parsing network streams
* parsing every strange grammar construction that might be possible
* parsing markup languages

## Nice to Have

Before getting into what I see as the requirements for a parser generator for a language designer, let's get out of the way those features that are useful but not necessary.

### Composability

Many authors focus on the issue of composing grammars to form new composite languages. For example, by embedding SQL statements as queries in another language like Java. Alternatively, composability can be used for intermixing languages as when languages are combined with HTML to form webpage templates. It can also be useful for embedding DSLs with different syntax into a host language. Parser generators often have one or both of two problems with this. First, combining two context-free grammars that conform to the limitations of the grammars accepted by the tool may not produce a grammar that does. Indeed, in the general case, combining two unambiguous grammars may produce an ambiguous grammar. Second, generators dependent on a separate lexer aren't able to combine their lexical specifications because the lexer often doesn't have enough information to switch between the two languages as needed. The scannerless parser generators often tout the ability to handle combining grammars, but still suffer from the first issue.

While having a tool that supports combining grammars would be handy, I don't see it as a must-have. In practice, languages are not combined very often. I don't think that is because of challenges with the tools, but rather it is not a problem that comes up very often. When it does arise, it is not as much of a problem as claimed. If the languages being combined are radically different, then for the sake of the programmer, there will probably need to be unambiguous delimiters at the transitions between the languages. These can be used to write melded lexer and grammar specifications easily. More often what is being done is adding features similar to another language, as was done when adding LINQ to C#. It simply doesn't make sense to combine languages with different rules for numeric  or other tokens.

As I'll discuss later, I find having a separate lexer valuable and don't see the benefits to composability outweighing that. However, if it is possible to design tools with distinct lexing and parsing phases that enable or ease combining grammars, that would be wonderful.

### Incremental Lexing and Parsing

As compilers and source code have grown in length and complexity, one response has been to adopt more incremental compilation. The ultimate in incremental compilation is to support incremental lexing and parsing. These enable advanced use cases like re-lexing and parsing as the developer types to provide real-time compiler errors. Out of the box support for incremental lexing and parsing would enable newly designed languages to offer these sophisticated features more easily. However, programmers have not yet come to see these features as mandatory for their work. New languages can afford to wait until they are mature to offer these.

### Control of Error Parsing

I'll discuss error handling in detail later. Here, I'd like to discuss a feature that might enable even better error handling but is definitely not required. In most compilers, the phases of compilation are independent so that information from later stages can't influence earlier phases. For correct programs, this works fine. However, when source code contains errors, early phases are forced to attempt to diagnose and recover from errors without access to analysis from later steps. For example, a parse error might be resolvable by inserting tokens in one of two different ways. The parser must make an arbitrary decision between these two resolutions. It might be the case though that one of the two produces a well-typed syntax tree while the other does not. If it were possible to easily control the resolution of errors in the parser, it might be possible to use the type information to make the correct decision.

### Token Value Generation

["Some Strategies For Fast Lexical Analysis when Parsing Programming Languages"](http://nothings.org/computer/lexing.html) discusses optimizing a lexer by generating token values during lexing. That is, since the characters are already being read once, create the value of a token during the initial lexing phase rather than reprocessing the token text after the fact. For example, compute the value of a numeric constant during lexing or generate the value of a string constant accounting for escape sequences. I've never seen a lexer generator that directly supports this (some can be fudged with lexer actions). The handling of escape sequences in strings has been particularly irksome to me. The lexer has already correctly identified each escape sequence, yet I end up writing a separate pass through the token value to generate the string constant which must also recognize the escape sequences. A similar operation is to compute the hash of an identifier during lexing and use it to perform string interning during lexing. This could entirely avoid the creation of many string objects.

## Requirements

Having gotten some nice to have features out of the way, let's look at what I consider to be the required features for a parser generator for programming language design. Throughout these, there are two recurring themes. First, languages that are being designed are in flux, and their grammars evolve over time. This is not always limited to simple additions and the designer may change to a radically different syntax or make fundamental semantic changes to the language. Second, error handling is essential. Besides generating correct, performant machine code, a compiler's most crucial task is to provide high-quality error messages to the developer. The quality of error messages impacts language adoption and is the majority of the "user interface" of a compiler.

## Separate Lexer

Scannerless parsing has grown in popularity as the increased performance of computers has made it feasible. As previously discussed, this is currently required for grammar composability. The lack of a separate lexer specification and attended specification language is often perceived as simplifying the use of the generator. However, I believe this is a tempting trap. As grammars grow, having a separate lexical specification begins to pay dividends. It gathers together the complete set of legal tokens rather than having them scattered throughout a grammar. Having a language designed specifically for lexing simplifies the specification. Without this, the somewhat distinct needs of lexing must be shoehorned into the formalisms of the parsing technology. Also, separating the lexer and parser doesn't mean that fixed tokens like keywords and operators can't be named by their text within the parser specification. Having a separate lexer is also valuable in implementing a number of the other requirements I will discuss.

Ultimately though, the argument for having a separate lexer is that it matches human language processing. People reading a text first separate it into tokens and then process those. Reflecting this two-layer scheme in the design of the language produces languages which are easy to read and simple to parse. Of course, there are instances where humans use top-down processing to adjust how they perceive individual tokens, but these are relatively rare. Likewise, there are instances where the parser providing input to the lexer is useful, but they are rare. These are better handled through specific lexer features. Many such cases can be dealt with through lexer modes and custom lexer actions. In other instances, this is not possible, but the parsing tool could be modified to better support it. Contextual keywords are a common example where separating lexing and parsing causes problems. However, this could be easily handled if a parser rule could specify that it matched an identifier with a particular value. Thus the lexer could emit contextual keywords as identifiers in all instances, but the grammar could express that certain words were expected in certain situations. Special handling for cases like the ambiguity between Java's `>>` operator and generics could also be developed. String interpolation is another common situation that should be accounted for.

## Unicode Support

Unicode support should go without saying. However, many parsing tools were developed before the widespread adoption of Unicode or support was left out to simplify the tool. Modern languages must provide Unicode support in strings if not also in identifiers. The challenge for parser generators is the vast number of states that Unicode support can produce. It can be a difficult performance issue for scannerless parsers. This is a situation where the separation of the lexer and parser can be advantages so that the complexity of Unicode can be isolated to the lexer.

## Unambiguous Grammars

The limitations of LL and LR grammars have led to newer tools adopting algorithms that embrace ambiguity. That is they accept ambiguous grammars and either produce parse forests or report and resolve ambiguity at runtime. They trade the problems of limited grammars for the uncertainty of ambiguous grammars. As a language designer, one wants to know that their language parses unambiguously. There has been work on identifying ambiguity in fully general context-free grammars. However since detecting ambiguity is an undecidable problem, these approaches must by necessity be approximate. Besides which, most tools don't offer any form of compile-time ambiguity detection anyway.

Since we can't detect arbitrary ambiguity, what we need is a new class of grammars which are unambiguous, but flexible enough to include the kinds of grammars we would naturally want to write for programming languages. Perhaps, by setting aside the problem of efficient parsing and looking only at building up unambiguous grammars, we could find such a class of grammars. That way, we could verify the grammar to be in that unambiguous class. Then we could use an algorithm like [Marpa](http://savage.net.au/Marpa.html), which while accepting ambiguous grammars claims to parse all reasonable unambiguous grammars in linear time, to implement the parser.

## Flexible Grammars and Disambiguation

For a fully known and static language, adapting a grammar to the limitations of LL or LR parsing is painful, but doable. For an open-ended, continually changing language, it is too much work. Simple modifications to a grammar can break the limitations. For me, lack of support for left-recursion is a nonstarter. What a language designer needs is support for a relatively flexible set of grammars that allow them to worry about their language instead of satisfying the parser generator. Likewise, when operators may be added and removed, rewriting a grammar to encode precedence levels is onerous. The parser generator should provide simple ways of specifying operator precedence and associativity as disambiguation rules on top of the grammar.

## Support Intransitive Operator Precedence

I've written before about how languages need to adopt [intransitive operator precedence]({% post_url 2019-02-26-operator-precedence %}). When specifying disambiguation rules for operator precedence and associativity, this should be supported.

## AST Generation and Control

When implementing the compiler for a new language, rapid development is vital. To support that, parser generator tools should provide automatic generation of abstract syntax trees. While not all designers may choose to use these, they can be an invaluable aid to those who do. To enable widespread adoption of these ASTs, they should be flexible in two ways. First in the structure of the generated tree and second in the actual code generated.

When using current parser generators, the AST often doesn't match the structure of the grammar. Thus the grammar isn't a reasonable basis from which to generate the AST. Support for flexible grammars and disambiguation should go a long way to mitigating this. However, more control over the generated AST could be invaluable. The [Hime](https://cenotelie.fr/projects/hime/) parser generator has a unique feature in this regard that more tools should adopt. Grammars can be augmented with annotations termed [tree actions](https://cenotelie.fr/projects/hime/reference/lang-tree-actions.html) which allow for the modification of the AST produced. Additional grammar features like lists (which may be token separated) enable ASTs to reflect the desired structure rather than the limitations of BNF. They can also enable optimizations. For example, lists can improve on the performance of right-recursive grammars for lists in LR parsers. It should also be possible to control which tokens and location information is included in the AST.

Getting AST nodes written in a compatible style can be just as important as getting the right AST structure. Tools that do generate ASTs frequently provide little to no control over the code generated for those ASTs. This can lead developers to abandon the use of those ASTs or the tool altogether. Full AST node customization may not be in the cards, but a few options should be available. In particular, I'd like to see control over node mutability so that immutable, partially mutable, or fully mutable nodes could be used. It should also be possible to easily add properties to all or a given subset of nodes. For example, to add a data type property to all expression nodes which will later be set to the expression's data type by the type checker.

## Support Concrete Syntax Trees

Increasingly, the compiler is not the only tool that needs to lex and parse source code in a given language. Yet these tools are often forced to implement their own lexers and parsers rather than reusing the ones used by the compiler. The problem is that they need access not merely to the AST but to the concrete syntax tree. That is the syntax tree with every token and all whitespace and comments included. The concrete syntax tree enables tools like automated refactoring, pretty-printing, code linters and document comment generators. Parser generators should support the reuse of a single grammar in both the compiler and these tools.

## Usability

As is too often the case with open-source tools, parser generators are often lacking in usability. Tools syntax, features, and limitations need to be clearly documented. They should be designed to be easy to learn and the grammars to be easily read. Remember that people using a parser generator are often using it for the first time and users referring to the grammar may not be familiar with the tool. Additionally, grammar errors should be clearly reported. Too frequently the error messages of parser generators are incomprehensible without detailed knowledge of the parsing algorithm being used. Sometimes, even that is not enough, and one must understand the particular implementation of the parser generator. Ambiguities and unsupported forms in the grammar should be clearly reported with an indication of which rules are the problem and where in the rule the issue occurs. The exact nature of the issue should be clearly explained. Ideally, an example string which will cause the parsing problem would be provided and if there is an ambiguity the different possible parse trees offered.

## Performance

Performance still matters for generated parser even with today's computers being multiple orders of magnitude faster than those available when parsing algorithms were first being developed. Developers still frequently complain about slow compilation times. The generated lexer and parser should be opportunities for easy performance wins as optimizations can be shared by every program using the tool. As one example, check out ["Some Strategies For Fast Lexical Analysis when Parsing Programming Languages"](http://nothings.org/computer/lexing.html) which describes a detailed optimization of a lexer that achieves significant performance improvements over generated lexers.

While performance is important, there is an important caveat to that. For *correct code* lexing and parsing should be *fast*. That is, they should be linear with a low constant, ideally on par with LL and LR algorithms. The vast majority of code compiled parses without error. However, for code with parsing errors, performance is much less of a concern. In most situations, there is a single or a few files with parsing errors. In those cases, producing good compiler errors is more important than fast parsing. So much so that it may not even be unreasonable to parse the file again with a different algorithm that handles errors better.

## Compiler Error Handling

I've saved the most important requirement for last. As I wrote above, generating good compiler error messages is one of the core responsibilities of a compiler. Yet, compiler generator tools give this short shrift. They often default to producing nothing but the first parse error and failing. That parse error is often confusing and poorly written. It refers to the particular convoluted grammar the tool happened to support. Reading the documentation on error handling often gives a short discussion of panicking (throwing away tokens) and of error rules. There is little to no discussion of how to generate the kind of good compiler errors developers expect from their compilers. Often the sample grammars have no error handling support. Many compiler tools provide virtually no information to use when trying to generate high-quality error messages. Furthermore, the very concept of compiler errors often seems to be a foreign concept to the tool, being totally omitted from the parser API.

Parser generators should make compiler errors a focus. They should provide lots of features for handling parse errors and detailed documentation on how to make the best use of those features. The default error messages generated by the compiler should be as good as possible for the programmer, not the grammar writer. Rather than offering minimal recovery strategies and hoping that the rest of the file will parse, the tool should fallback to a more sophisticated parsing strategy in the face of errors. One that can take into account parsing after the error to select the best recovery choice. This is an area where parser generators could offer a great deal of value over hand-written parsers. Very few hand-written parsers can afford a second parsing strategy optimized for error recovery. A parser generator can feed the single grammar into two different algorithms to offer this functionality with little to no impact to the compiler writer.

## Enabling Language Growth

All of the requirements I've laid out here can be summed up by one goal: enabling language growth. That is supporting the life cycle of new languages by providing value in each phase of their development and building on past stages with each new one. Initially, a new language needs a quick and dirty way to get lexing and parsing working for a small grammar. Existing parser generators do ok at this but would benefit from AST generation and improved usability. As the language grows and evolves, support for flexible grammars and disambiguation enables rapid design iteration. Additionally, having a separate lexer and unambiguous grammars guide the language development toward good designs while support for intransitive operator precedence provides design freedom. As the language nears v1.0, Unicode support, performance and error handling become important. Then as the ecosystem matures, further improvements to error handling and the development of additional tools for the language ecosystem enabled by concrete syntax trees bring the language on par with the mainstream languages with which it is competing.

### Requirements Summary:

* Separate Lexer
* Unicode Support
* Unambiguous Grammars
* Flexible Grammars and Disambiguation
* Support Intransitive Operator Precedence
* AST Generation and Control
* Support Concrete Syntax Trees
* Usability
* Performance
* Compiler Error Handling

---

Additional Reading:

* [Parsing: a timeline](https://jeffreykegler.github.io/personal/timeline_v3) by Jeffrey Kegler author of [Marpa](http://savage.net.au/Marpa.html) is a good history of parsing algorithms with a bias toward those leading to the algorithm used in Marpa. In particular, it omits GLL and GLR.
* [What are the reasonable computer languages?](https://jeffreykegler.github.io/Ocean-of-Awareness-blog/individual/2016/01/lrr.html) by Jeffrey Kegler
* [Parsing Expressions by Recursive Descent](https://www.engr.mun.ca/~theo/Misc/exp_parsing.htm) by Theodore Norvell
* [Generating Good Syntax Errors](https://research.swtch.com/yyerror) (in LR parsers) by Russ Cox
* [Parsing list comprehensions is hard](http://www.rntz.net/post/2018-07-10-parsing-list-comprehensions.html) by Michael Arntzenius
* [A Haskell challenge](https://jeffreykegler.github.io/Ocean-of-Awareness-blog/individual/2018/08/rntz.html) by Jeffrey Kegler responds to "Parsing list comprehensions is hard" in the context of [Marpa](http://savage.net.au/Marpa.html).

{% comment %}
More links including grammar examples:

* http://lambda-the-ultimate.org/node/4489
* http://www.cs.utsa.edu/~wagner/CS3723/grammar/examples.html
* https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form
* https://jeffreykegler.github.io/Ocean-of-Awareness-blog/individual/2016/01/lrr.html
* https://jeffreykegler.github.io/Ocean-of-Awareness-blog/individual/2012/self_parse.html
* https://jeffreykegler.github.io/personal/timeline_v3#g-basic-op
* http://www.rntz.net/post/2018-07-10-parsing-list-comprehensions.html
* https://jeffreykegler.github.io/Ocean-of-Awareness-blog/individual/2018/08/rntz.html
* https://jeffreykegler.github.io/Ocean-of-Awareness-blog/individual/2014/02/semantic_ws.html
* https://jeffreykegler.github.io/Marpa-web-site/
* http://savage.net.au/Marpa.html
{% endcomment  %}

*[DSLs]: Domain Specific Languages
*[AST]: Abstract Syntax Tree
*[ASTs]: Abstract Syntax Trees
*[BNF]: Backus–Naur form
