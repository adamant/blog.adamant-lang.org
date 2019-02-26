---
layout: post
title: "Operator Precedence: We can do better"
date: 2019-02-26 10:20:00 -0500
tags: ["Language Design", Parsing, "Compiler Tools"]
author: "Jeff Walker"
---
The longer I've thought about how to handle [operator precedence](https://en.wikipedia.org/wiki/Order_of_operations#Programming_languages) and [associatively](https://en.wikipedia.org/wiki/Operator_associativity) in a programming language, the more convinced I've become that languages have fallen short. Because it was simple, easy and efficient, language designers have generally provided a [total order](https://en.wikipedia.org/wiki/Total_order) for operator precedence and made all operators associative. This is typically expressed as a set of operator precedence levels and associativity for each operator. However, this often leads to unexpected or even confusing precedence between operators. Languages allowing programmers to define new operators from combinations of symbols are particularly hurt by forcing all operators to be placed in one of a few precedence levels. In reaction, some designers eschew operator precedence entirely. While simple, that violates deep-seated programmer intuitions opening the way for mistakes and surprise. I believe future languages should adopt intransitive operator precedence instead.

*Note:* I am focused here only on language with infix operators. Languages using [prefix notation](https://en.wikipedia.org/wiki/Polish_notation), such as Lisp variants, and languages using [postfix notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation), such as Forth, can be unambiguous without operator precedence.

## Existing Practise

Most programming languages with infix operators fall into one of four categories:

1. **Total Order Precedence and Total Associativity:** Every operator has a precedence relative to every other operator. Every operator is either left- or right-associative. \\
    *Example Languages:* C, C++, C♯, Java, Go, Lua, Kotlin
2. **Total Order Precedence with Partial Associativity:** Every operator has a precedence relative to every other operator. Some operators are neither left- nor right-associative. In some languages, there are non-associative operators. For example, in Rust <span class="nowrap">`x <= y == z`</span> is illegal and would need to have parentheses added. In other languages, chained operators are interpreted differently. For example, in Python <span class="nowrap">`x < y < z`</span> is equivalent to <span class="nowrap">`x < y and y < z`</span>. \\
    *Example Languages:* Python, Rust, Prolog, Haskell, Perl
3. **Single Precedence and Associativity:** Every infix operator has the same precedence and associativity. Unary operators may or may not have higher precedence than binary operators. \\
    *Example Languages:* Smalltalk, APL, [Mary](https://en.wikipedia.org/wiki/Mary_(programming_language))
4. **Single Precedence and Non-associative:** Every infix operator has the same precedence and is non-associative. Thus all expressions must be fully disambiguated with parentheses. Unary operators may or may not have higher precedence than binary operators. \\
    *Example Languages:* [occam](https://en.wikipedia.org/wiki/Occam_(programming_language)), [Pony](https://www.ponylang.io), [RELAX NG](https://www.oasis-open.org/committees/relax-ng/compact-20021121.html#syntax)

## Faults

Unfortunately, each these options has shortcomings. A set of test expressions best illustrates this.

* `x + y * z` is almost universally read as `x + (y * z)` because this is the convention everyone is taught from elementary school onward. Breaking this convention will only lead to confusion and frustration. Requiring explicit parentheses, in this case, isn't as bad, but is still annoying.
* `x < y < z` is probably either a bug or meant to mean `x < y` and `y < z`. Treating relational operators as left-associative has led to hard to spot bugs in C code.
* By mathematical convention, logical-and has higher precedence than logical-or, so <span class="nowrap">`a or b and c`</span> should be parsed as <span class="nowrap">`a or (b and c)`</span>. However, there is no convention for the relative precedence of logical-xor. Any precedence assigned to it will be arbitrary. Yet, all logical connective should have lower precedence than equality. Thus we need an operator that has no precedence relative to some operators, but precedence relative to others so that <span class="nowrap">`a xor x == y`</span> parses as <span class="nowrap">`a xor (x == y)`</span>, but <span class="nowrap">`a xor b or c`</span> is an error.

Let's consider how each of the approaches fairs on our test cases. Of course, we don't want to evaluate a single language, but an idealized version of each approach. Single precedence and associativity requires that all operators be either left- or right-associative; which should we pick? Regardless of which is chosen, it will be easy to construct examples where it is incorrect for the operators involved. To simplify the test, I've always assumed the worst case for the given test.

<table class="table text-center">
    <colgroup span="1"></colgroup>
    <colgroup span="2"></colgroup>
    <colgroup span="2"></colgroup>
    <tr>
        <td></td>
        <th colspan="2" class="text-center" scope="colgroup">Total Order</th>
        <th colspan="2" class="text-center" scope="colgroup">Single Precedence</th>
    </tr>
    <tr>
        <th>Test Case</th>
        <th>Total Associativity</th>
        <th>Partial Associativity</th>
        <th>Single Associativity</th>
        <th class="nowrap">Non-associative</th>
    </tr>
    <tr>
        <th scope="row"><code>x + y * z</code></th>
        <td>✓</td>
        <td>✓</td>
        <td>✗</td>
        <td>✗</td>
    </tr>
    <tr>
        <th scope="row"><code>x < y < z</code></th>
        <td>✗</td>
        <td>✓</td>
        <td>✗</td>
        <td>✓</td>
    </tr>
    <tr>
        <th scope="row" class="nowrap"><code>a xor x == y</code></th>
        <td>✓</td>
        <td>✓</td>
        <td>✗</td>
        <td>✗</td>
    </tr>
    <tr>
        <th scope="row" class="nowrap"><code>a xor b or c</code></th>
        <td>✗</td>
        <td>✗</td>
        <td>✗</td>
        <td>✓</td>
    </tr>
</table>

## Partial Order

Of the existing options, total order precedence with partial associativity scores the best. However, it fails to treat `a xor b or c` as an error. How can we fix this? Well, we could make operator precedence a [partial order](https://en.wikipedia.org/wiki/Partially_ordered_set#Formal_definition) instead of a total order. We could then include in our precedence <span class="nowrap">`or` ≺ `and`</span>, <span class="nowrap">`xor` ≺ `==`</span>, <span class="nowrap">`or` ≺ `==`</span>, and <span class="nowrap">`and` ≺ `==`</span>. That would correctly handle both <span class="nowrap">`a xor x == y`</span> and <span class="nowrap">`a xor b or c`</span>.

However, using a partial order for operator precedence can still lead to problems. Consider the expression <span class="nowrap">`x and y + z`</span>. Since this mixes logical and arithmetic operators, there isn't an obvious precedence. We want to force the developer to add parentheses. One might think this is not a problem for a partial order. Yet, logical operators are lower precedence than equality <span class="nowrap">(`and` ≺ `==`)</span> and equality is lower precedence than arithmetic <span class="nowrap">(`==` ≺ `+`)</span>. Since partial order relations are transitive, those imply that <span class="nowrap">`and` ≺ `+`</span>. That isn't what we want, so we need a precedence relation that is [intransitive](https://en.wikipedia.org/wiki/Intransitivity).

## Intransitive Precedence

Let's define the kind of precedence we want. I'll call this an *intransitive operator precedence*. We'll define both an equivalence relation "≐" for operators at the same precedence and a compatible order relation "⋖" for operators with different precedence. However, our precedence relation will be intransitive. Additionally, we'll require that the precedence form a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph). We can then use them to define the precedence relationships between our operators. Associativity will be specified separately.

For the mathematically inclined, the relations have the following properties:

1. ≐ is an [equivalence relation](https://en.wikipedia.org/wiki/Equivalence_relation):
   * *a* ≐ *a* ([reflexivity](https://en.wikipedia.org/wiki/Reflexive_relation))
   * if *a* ≐ *b* then *b* ≐ *a* ([symmetry](https://en.wikipedia.org/wiki/Symmetric_relation))
   * if *a* ≐ *b* and *b* ≐ *c* then *a* ≐ *c* ([transitivity](https://en.wikipedia.org/wiki/Transitive_relation))
2. ⋖ is a strict intransitive order compatible with the equivalence relation
   * It is never the case that *a* ⋖ *a* ([irreflexivity](https://en.wikipedia.org/wiki/Reflexive_relation#Related_terms))
   * If *a* ⋖ *b*, then it is not the case that <span class="nowrap">*b* ⋖ *a*</span> ([asymmetry](https://en.wikipedia.org/wiki/Asymmetric_relation))
   * If *a* ⋖ *b* and *b* ⋖ *c*, it does ***not*** follow that <span class="nowrap">*a* ⋖ *c*</span> (but it could be the case) ([intransitivity](https://en.wikipedia.org/wiki/Intransitivity))
   * There does not exist <span class="nowrap">*a*<sub>0</sub> , ... , *a*<sub>*n*</sub></span> such that <span class="nowrap">*a*<sub>0</sub> ⋖ *a*<sub>1</sub> , ... , *a*<sub>*n*-1</sub> ⋖ *a*<sub>*n*</sub></span> and <span class="nowrap">*a*<sub>*n*</sub> ⋖ *a*<sub>0</sub></span> ([acyclic](https://en.wikipedia.org/wiki/Directed_acyclic_graph))
   * If <span class="nowrap">*a* ≐ *b*</span> and <span class="nowrap">*a* ⋖ *c*</span>, then <span class="nowrap">*b* ⋖ *c*</span>. Likewise if <span class="nowrap">*a* ≐ *b*</span> and <span class="nowrap">*d* ⋖ *a*</span>, then <span class="nowrap">*d* ⋖ *b*</span>.

This allows us to declare our desired precedence reasonably easily. First, we declare which operators have equal precedence, for example `*` ≐ `/`. Then we declare the relative precedence of operators, for example `or` ⋖ `and`. Operators of equal precedence share in the precedence we define. However, because precedence is intransitive, there can still be a lot of relations to specify. To simplify, we adopt two notational conveniences. First, that a precedence chain relates every operator to every other operator before and after it so that <span class="nowrap">`or` ⋖ `and` ⋖ `not`</span> states that <span class="nowrap">`or` ⋖ `not`</span> as well and second, that groups of operators can be related by using sets. For example, <span class="nowrap">{`and`, `or`, `not`} ⋖ `==`</span> relates all the boolean operators to the equality operator.

## An Example

It's easy to get lost in the math and notation. Let's look at a concrete example to see how this might play out in a real language. Below I've defined a simple expression language over integers and booleans. To be clear, I'm not arguing for this particular set of operator precedences. Other language designers may prefer slightly different ones. I am arguing that languages should use this kind of flexible precedence to avoid undesirable precedence relationships.

I've used a form of [EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) augmented with additional notation to represent associativity and intransitive operator precedence. Without these additional annotations, the grammar would be an [ambiguous expression grammar](https://en.wikipedia.org/wiki/Ambiguous_grammar#Addition_and_subtraction). The intent is that a hypothetical parser generator could directly use this grammar. The [grammar notation](#added-grammar-notation) section below gives a detailed explanation of the additional notation used.

```ebnf
(E) = "(" (E) ")"   #Parens
    (* Arithmetic Operators *)
    | (E) "+" E     #Add
    | (E) "-" E     #Sub
    | (E) "*" E     #Mul
    | E "/" E       #Div
    | E "^" (E)     #Pow (* raise to power *)
    | "-" E         #Neg
    (* Equality Operators *)
    | E "==" E      #EQ
    | E "<>" E      #NEQ
    (* Relational Operators *)
    | E "<" E       #LT
    | E "<=" E      #LTE
    | E ">" E       #GT
    | E ">=" E      #GTE
    (* Logical Operators *)
    | (E) "and" E   #And
    | (E) "or" E    #Or
    | (E) "xor" E   #Xor
    | "not" (E)     #Not
    (* Conditional Operator *)
    | E "?" E ":" E #Cond
    (* Variables *)
    | ID            #Var
    ;

ID = ?identifier?;

(* arithmetic precedence  *)
#Add =.= #Sub;

#Cond[inner, right]
    <. #Add
    (* division is not equal to multiplication *)
    <. {#Mul, #Div}
    (* negative exponent allowed *)
    <. #Pow[right]
    <. #Neg
    (* negative base requires parens *)
    <. #Pow[left]
    <. #Parens;

(* equality and relation precedence *)
#EQ =.= #NEQ;
#LT =.= #LTE =.= #GT =.= GTE;

#EQ (* following C convention, equality is below relation *)
    <. #LT
    (* equality and relation are below arithmetic *)
    <. {#Add, #Mul, #Div, #Neg, #Pow};

(* logical operator precedence *)
#Or <. #And;

#Cond
    <. {#Or, #Xor, #And}
    (* logical are below equality and relation *)
    <. {#EQ, #LT}
    (* both are below logical not *)
    <. #Not
    (* all lower than parentheses *)
    <. #Parens;
```

This grammar tries to follow mathematical conventions without relating operators that have no conventional relationship. Powers are right associative and higher precedence than negation. The division slash is non-associative to avoid confusion. It does follow the C convention and make equality lower precedence than relations. The test cases below demonstrate the grammar has the desired properties.

| Expression            | Parses As                                             |
| --------------------- | ----------------------------------------------------- |
| `x + y * z`           | `x + (y * z)`                                         |
| `(x + y) * z`         | `(x + y) * z`                                         |
| `x < y < z`           | Error, non-associative                                |
| `a xor x == y`        | `a xor (x == y)`                                      |
| `a xor b or c`        | Error, `xor` and `or` are not related                 |
| `x / y / z`           | Error, non-associative to avoid confusion             |
| `x / y * z`           | Error, `/` and `*` are not related to avoid confusion |
| `x ^ y ^ z`           | `x ^ (y ^ z)` (right-associative)                     |
| `-x^y`                | `-(x^y)` (as in written math)                         |
| `x^-y+z`              | `(x ^ (-y)) + z`                                      |
| `not a + x`           | Error, `not` and `+` are not related                  |
| `a and b ? x : y + z` | `(a and b) ? x : (y + z)`                             |
| `x + y ? a : b`       | Error, `+` and `?` are not related                    |
| `a ? b ? x : y : z`   | Error, conditional operator is non-associative        |
{: .table }

## Added Grammar Notation

In the grammar above `(E) =` indicates that `E` is a "parenthesized" nonterminal. Normally, the declaration of `E` would be ambiguous, but a parenthesized nonterminal defaults to disallowing alternatives containing recursive uses of the nonterminal from being immediate children of the production. Thus <span class="nowrap">`(P) = P "~" P | P "$" P | ID;`</span> is effectively transformed to <span class="nowrap">`P = P' "~" P' | P' "$" P'; P' = ID;`</span>. This has the effect of making the operators non-associative. The intuition here is that parenthesized nonterminals will have to be fully parenthesized unless additional associativity and precedence rules are declared.

Associativity is indicated by enclosing a recursive use of the nonterminal in parentheses. A recursive use enclosed in parentheses allows the same alternative to occur as a direct child of that nonterminal. Thus <span class="nowrap">`(E) = (E) "+" E`</span> is left-associative and <span class="nowrap">`(E) = E "^" (E)`</span> is right-associative. The rule <span class="nowrap">`(P) = (P) "~" (P)`</span> is ambiguous. Again, non-associative is the default for parenthesized nonterminals, i.e. <span class="nowrap">`(E) = E "<" E`</span>. Intuitively, the parentheses indicate which side expressions should be grouped on. One wrinkle this creates is that to allow nesting of parentheses in parentheses, the nonterminal must be enclosed in parentheses as <span class="nowrap">`(E) = "(" (E) ")"`</span> or else <span class="nowrap">`((x))`</span> is illegal.

 Labels are applied to each alternative by placing them after the alternative. Labels are prefixed with a pound sign. The [ANTLR](https://www.antlr.org/) parser generator uses the same notation. Labels provide a way to refer to alternatives later. They can also be used by a parser generator to name the [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) node for that alternative.

Operator precedence is declared after the production rules using the precedence relation applied to the alternative labels. Using the labels makes it easy to give binary and unary versions of the same operator different precedences. Two operators are given the same precedence using the `=.=` operator. Relative precedence is established using the `<.` operator. As described in the previous section, chains and sets can be used to simplify the declaration of precedence. A precedence declaration affects recursive uses of the nonterminal in the alternatives it relates. Alternatives with higher precedence may be direct children at any use of the nonterminal. Alternatives with equal precedence may only be direct children where the nonterminal is enclosed in parentheses.

In some instances, more complex operators have different precedence for different subexpressions. An array indexing operator <span class="nowrap">`(P) = P "[" P "]" #Index;`</span> would be such a situation. Here, the bracketed `P` could be of any precedence while the left `P` must be higher precedence. In such situations, we can refer to the precedence of the subexpressions using a bracket notation listing the indexes of nonterminals in the alternative. For example, <span class="nowrap">`#Index[1]`</span> refers to the first subexpression, <span class="nowrap">`#Index[2]`</span> refers to the second, and `#Index[1,2]` refers to both. For convenience, four shorthands are provided. The names `left` and `right` refer to the leftmost and rightmost nonterminal not bracketed by a terminal. In the example, `#Index[left]` is the same as `#Index[1]` while `#Index[right]` is an error because the rightmost `P` has the terminal `"]"` to its right. The name `outer` refers to both the `left` and `right` so `#X[outer]` would be equal to `#X[left, right]`. The name `inner` refers to every subexpression that is not an outer subexpression. Thus `#Index[inner]` would be equal to `#Index[2]`. In the example grammar above, this is used to allow a negative sign in the exponent while giving exponentiation higher precedence and to allow logical but not arithmetic expressions in the condition of a conditional expression.

## Don't Mix Associativity

To consider the issues involved in mixing operators with different associativity at the same precedence level, imagine adding the following to the above grammar.

```ebnf
(E) = (E) "⊕" E  #CAdd (* left-associative *)
    | E "⍟" (E)  #CPow (* right-associative *)
    | E "⊜" E    #CEQ  (* non-associative *)
    ;

#CAdd =.= #CPow =.= #CEQ;
```

By the rules stated before, what would the effect of this be? Let's look at each case.

| Expression  | Parses As     |
| ----------- | ------------- |
| `x ⊕ y ⍟ z` | Error         |
| `x ⍟ y ⊕ z` | Ambiguous     |
| `x ⊕ y ⊜ z` | Error         |
| `x ⊜ y ⊕ z` | `(x ⊜ y) ⊕ z` |
| `x ⍟ y ⊜ z` | `x ⍟ (y ⊜ z)` |
| `x ⊜ y ⍟ z` | Error         |
{: .table }

Given that this is almost certainly not what one wants, it is best to simply make it illegal to have operators with the same precedence but different associativity.

## Assignment Example

In C style languages the assignment operator is right-associative and evaluates to the value of the left-hand variable *after* it is assigned. Assignment has lower precedence than addition, so the expression `a+b=c+d` parses to `(a+b)=(c+d)` which is illegal. One might prefer that it parse as `a+(b=(c+d))`. Setting aside whether that is a good idea, it can be achieved with this scheme. The example expression grammar could be extended with assignment by adding the rule and precedences below. By splitting the precedence of the left and right, we can make assignment bind very tightly on the left, but very loosely on the right.

```ebnf
(E) = E "=" (E) #Assign;

{#Cond, #Add, #Mul, #Div, #Pow, #Neg,
        #Or, #Xor, #And, #Not, #EQ, #LT}
    <. #Assign[left];

#Assign[right]
    <. {#Cond, #Add, #Mul, #Div, #Pow, #Neg, #Parens,
        #Or, #Xor, #And, #Not, #EQ, #LT};
```

## What Now?

I'm not the first one to propose something like intransitive operator precedence. The [Fortress](https://en.wikipedia.org/wiki/Fortress_(programming_language)) language has an elaborate operator precedence scheme that is similar. Check out [*The Fortress Language Specification* v1.0](http://www.ccs.neu.edu/home/samth/fortress-spec.pdf), chapter 16 for more information. However, it was difficult to find much else. The precedence level approach seems to have completely dominated. Hopefully, I've convinced you of the value of intransitive operator precedence or at least given you something to think about. I'd love to see future programming languages adopt this approach. Unfortunately, algorithms for parsing such precedence schemes are lacking. If you want to implement such a scheme or are interested in learning more, check out these sources:

* [Parsing Fortress Syntax](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.925.5298&rep=rep1&type=pdf) by Sukyoung Ryu.
* [Parsing Mixfix Operators](http://www.cse.chalmers.se/~nad/publications/danielsson-norell-mixfix.pdf) by Nils Anders Danielsson and Ulf Norell which proposes a similar scheme for the Agda language.
* [SDF](http://www.meta-environment.org/doc/books/syntax/sdf/sdf.html#section.priorities) and [SDF3](https://www.metaborg.org/en/latest/source/langdev/meta/lang/sdf3/reference.html#priorities) which provide both position based priority and intransitive priority for parsing.

{% comment %}
Other references:

* https://en.wikipedia.org/wiki/Operator-precedence_grammar
* https://foonathan.net/blog/2017/07/24/operator-precedence.html
* http://lambda-the-ultimate.org/node/2943
* "Precedence Languages and Bounded Right Context Languages" (Stanford, 1971)
* "Total Precedence Relations," JACM (January 1970)
* Jim Gray and Michael Harrison: "Canonical Precedence Schemes," JACM (April, 1973)
* "The Classes of Environment T-Operator Precedence Languages," (1978).
* https://github.com/JuliaLang/julia/issues/18714
{% endcomment %}

*[EBNF]: Extended Backus–Naur form
*[AST]: Abstract Syntax Tree
*[DAG]: Directed Acyclic Graph
