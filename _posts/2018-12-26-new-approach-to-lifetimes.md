---
layout: post
title: "A New Approach to Lifetimes in Adamant"
date: 2018-12-26 11:45:00 -0500
tags: ["Adamant", "Memory Management"]
author: "Jeff Walker"
---
Adamant's compile-time memory management allows the compiler to enforce memory safety. However, that requires the declaration of the sometimes complex relationships between references and values. Initially, this was done in a way similar to Rust's lifetime parameters and annotations. However, lifetimes are one of the most confusing parts of Rust. In an attempt to make lifetimes in Adamant clearer, a syntax based around expressing the relationships between lifetimes was developed. That syntax was described in the "[Basic Memory Management in Adamant]({% post_url 2018-12-17-basic-memory-management-in-adamant %}){:class="internal"}" post. However, Jonathan Goodwin pointed out that the return types in the examples weren't consistent with how lifetimes were defined and used throughout. We talked through what lifetime parameters mean in Rust and came to understand them better. By the end of that, I came up with a new approach to lifetimes in Adamant. Let's look at that approach by working through examples drawn from the [Rust Book](https://doc.rust-lang.org/1.31.1/book/).[^rust-book]

## Lifetimes for Return Values

Inside function bodies, the compiler can automatically deduce and check the relationships between references and values. In function signatures, one must declare the relationships. For simple relationships between parameters and return values, lifetime elision rules provide defaults, so the programmer doesn't need to do this. When the defaults aren't enough, the connections must be explicitly declared. An example of such a function is one that takes two strings and returns the longest of the two.[^longest] In Rust, this requires a lifetime parameter and lifetime annotations on all the references involved. However, that leads to confusion and questions about what the lifetime parameter means and how it relates to the references.

```rust
// A correctly annotated longest function in Rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

The constraints on the references and values in the `longest` function are quite complicated. However, they are the result of avoiding dangling references and of how the parameters are used to create the return value. So instead of directly trying to express those constraints, we could state how the parameters are related to the return value. In all but the most complicated situations requiring lifetime parameters, only which parameters are related to the return value is stated. The `longest` function requires annotations because both of its parameters are related to the return value. Why is that the case? In this function, either parameter could be returned. Either parameter could go into creating the return value. That is what we need to express.

In the mathematical sense, functions take their parameters and transform them into a return value. Sometimes though, a parameter may not be used, may be used only in side-effecting code or may only indirectly be used to create the return value. For example, a parameter might be used only as part of a condition, after which it is no longer needed. Thus, we *can't* just assume every parameter constrains the lifetime of the return value. We need to declare not only the parameters but which lifetimes go into producing the return value. That is precisely what the new approach does.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">longest</span>(<span class="hljs-params">x: String, y: String</span>) <span class="highlight">$x &amp; $y</span> -> String</span>
{
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">if</span> x.grapheme_count() > y.grapheme_count()
                => x
           <span class="hljs-keyword">else</span>
                => y;
}
</code></pre>

The highlighted code expresses the relationship of the parameters' lifetimes to the return value. When applied to a variable the `$` should be read as "the lifetime of". This refers to the lifetime of the referent (i.e., the object), *not* of the references themselves. The `&` is the operator for constructing intersection types and is read "and". That is `T1 & T2` is the type of all values that implement both `T1` and `T2`. Thus the code means "the lifetime of x and the lifetime of y". The placement of these to the left of the arrow is meant to indicate that just like the parameters to the function, these two lifetimes go into creating the return value. Indeed they are like parameters in that the lifetime of `x` and `y` will be different at each call site. This syntax captures the idea that either parameter could go into the return value. It does not make a direct statement about the various constraints placed on the lifetimes of the references involved. Consequently, it is much easier to reason about and validate as correct.

This syntax requires the repetition of the parameter names for the lifetime declaration. Some may find this too verbose. So, further shorthand syntax could be added. For example, a dollar sign after the parameter name might indicate that both the parameter's value and its lifetime are used. The example above would then be declared <code><span class="hljs-function"><span class="nowrap"><span class="hljs-keyword">fn</span> <span class="hljs-title">longest</span></span><span class="nowrap">(<span class="hljs-params">x$: String, y$: String</span>)</span> <span class="nowrap">-> String</span></span></code>. To keep early versions of the language simple, no shorthand is currently supported.

Of course, more complex relationships between the parameter lifetimes and return value are possible. For those situations, the arrow can be used to indicate which lifetimes go into which part of the return type. For the first example, note that `Tuple` is a value type and so does not require a lifetime declaration.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">make_tuple</span>(<span class="hljs-params">x: String, y: String</span>)
    -> Tuple<span class="before-highlight">[</span><span class="highlight">$x -></span> String, <span class="highlight">$y -></span> String]</span>
{
    <span class="hljs-keyword">return</span> #(x, y);
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">make_list</span>(<span class="hljs-params">x: String, y: String</span>)
    -> List<span class="before-highlight">[</span><span class="highlight">$x & $y</span> -> String]$<span class="hljs-keyword">owned</span></span>
{
    <span class="hljs-keyword">return</span> #[x, y];
}
</code></pre>

It may be possible to simplify the declaration of `make_list` to <code><span class="hljs-function"><span class="nowrap"><span class="hljs-keyword">fn</span> <span class="hljs-title">make_list</span></span><span class="nowrap">(<span class="hljs-params">x:&nbsp;String, y: String</span>)</span> <span class="nowrap">$x & $y</span> <span class="nowrap">-> List[String]$<span class="hljs-keyword">owned</span></span></span></code> since there is no other place the lifetimes could go into the return value except for the strings in the list.

## Parameter Lifetime Constraints

Sometimes, the issue is not how the lifetimes of the parameters relate to the return values, but rather how they relate to each other. In those situations, we need to express constraints between lifetimes. Rust uses lifetime subtyping for this. In Rust, `'a: 'b` can be read "the lifetime a outlives the lifetime b". This relationship can be confusing and hard to remember. To see why the subtype relationship implies `a` outlives `b`, consider that a subtype must be substitutable for its supertype. The new approach allows the expression of the same relationships without introducing lifetime parameters.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">assign_into</span>(<span class="hljs-params">value: String, variable: <span class="hljs-keyword">ref</span> <span class="hljs-keyword">var</span> String</span>)
    <span class="hljs-keyword">where</span> <span class="highlight">$value > $*variable</span></span>
{
    *variable = value;
}
</code></pre>

The `assign_into` function takes a string and a reference to a variable of type `String` and assigns the string into the variable using the dereference operator `*`. For this to be safe, we must declare that the lifetime of the value we are assigning is greater than the lifetime of the space we are assigning it into. This is done using a generics constraints clause introduce by `where`. The expression directly states the relationship between the variable we are assigning into and the lifetime of the value. Notice that rather than using a subtype relationship, we can use comparison operators on lifetimes. It may also be possible to allow the use of the arrow in where clauses in which case the constraint would become `$value -> *variable`.

## Lifetimes of Borrowed References in Classes

Of course, functions aren't the only place where the relationship between lifetimes must be declared. Rust uses lifetime parameters and annotations in struct declarations too. An example from the Rust Book that illustrates this is based on a parser returning an error.[^parser] Below is the correct Rust code. The Rust 2018 edition simplifies this slightly. Additional inference rules were added, and it now infers the relationship between the lifetimes in the `Parser` struct. It can now be declared `struct Parser<'c, 's>`.

```rust
struct Context<'s>(&'s str);

struct Parser<'c, 's: 'c> {
    context: &'c Context<'s>,
}

impl<'c, 's> Parser<'c, 's> {
    fn parse(&self) -> Result<(), &'s str> {
        Err(&self.context.0[1..])
    }
}

fn parse_context(context: Context) -> Result<(), &str> {
    Parser { context: &context }.parse()
}
```

Here, multiple lifetime parameters are necessary. If `Parser` were declared with only a single lifetime parameter then the `parse_context` function would not compile. It takes ownership of the context, so the reference to the context has a much shorter lifetime than the string it contains. With a single lifetime parameter, these two lifetimes get collapsed into one, and the compiler is no longer able to tell that the string returned in the error result lives long enough to be safely returned from the `parse_context` function.

In Adamant, error handling like this would probably be done using exceptions. The lifetime issues would be similar, but to keep the examples parallel, I'll assume a result type is used. Unfortunately, it isn't possible to use the same trick we did with functions to entirely remove the lifetime parameters. In a class, multiple methods may need to use the same lifetime and expressing the relationships between all the lifetimes in the various methods would quickly get out of hand. However, there is a way to simplify them and make them more intuitive. The critical insight is that lifetimes can be treated more like [associated types](https://doc.rust-lang.org/1.31.1/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types) rather than generic parameters.

<pre><code class="hljs nohighlight"><span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">Context</span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">lifetimes</span> $text;

    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> text: String$text;

    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">new</span>(<span class="hljs-params">.text</span>)</span> {}
}

<span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">Parser</span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">lifetimes</span> $context: Context;

    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> context: Context$context;

    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">new</span>(<span class="hljs-params">.context</span>)</span> { }

    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">parse</span>(<span class="hljs-params"><span class="hljs-keyword">self</span></span>)
        -> Result[<span class="hljs-keyword">never</span>, $context.text -> String]</span>
    {
        <span class="hljs-keyword">return</span> .Error(<span class="hljs-keyword">self</span>.context.text.slice(<span class="hljs-number">1</span>..);
    }
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-title">parse_context</span>(<span class="hljs-params">context: Context</span>)
    -> Result[<span class="hljs-keyword">never</span>, $context.text -> String]</span>
{
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">new</span> Parser(context).parse();
}
</code></pre>

The `lifetimes` keyword introduces a comma-separated list of named lifetimes associated with a class. In this example, the lifetime names match the names of the properties. This is possible because lifetimes have a distinct namespace. A class may have private fields with associated lifetimes where the field cannot be accessed, but the lifetime can. Indeed associated lifetimes must be declared public. Rarely, a class may even have lifetimes that don't correspond to any fields in the class. This might be done for classes that use unsafe pointers or for base classes with abstract methods using the lifetimes.

The associated lifetimes are essentially lifetime properties of the object, accessible from variables of that type using the lifetime operator. These properties chain so that a variable `p: Parser` would have a lifetime property of `$p.context.text` available. Thus associated lifetimes are themselves "typed". This is what the declaration `$context: Context` in the parser class is indicating. This typing and nesting replaces the need to declare multiple lifetimes on the parser class as is done in the Rust code. Nested lifetimes are automatically treated as independent. As you can see, these named lifetimes are bound to fields of the class and are available for use in methods and functions.

The typing of lifetimes may have an additional benefit. It may allow the lifetime elision rules to handle more sophisticated cases. For example, a `make_tuple` function like the example above except with two different parameter types might require no annotations because the types associated with the lifetimes of the parameters can only match up to the types in the return type in a single way, i.e. <code><span class="hljs-function"><span class="nowrap"><span class="hljs-keyword">fn</span> <span class="hljs-title">make_tuple</span></span><span class="nowrap">(<span class="hljs-params">x: String, y: List[<span class="hljs-keyword">int</span>]</span>)</span> <span class="nowrap">-> Tuple[String, List[<span class="hljs-keyword">int</span>]]</span></span></code>.

I think the use of named lifetimes in classes is far less confusing than the use of lifetime parameters in functions. In a function, a single lifetime parameter is tied to multiple references. However, that lifetime often doesn't correspond to the lifetime of any of the objects or references involved. In a class, each named lifetime will typically be used with a single field. It can thus be thought of as more directly corresponding with the actual lifetime of the value in that field.

The syntax used above is deliberately verbose. Users often prefer unfamiliar features to have clearer, more explicit syntax. This syntax makes it completely clear that the lifetimes are separate from the fields, but used by them and that they are public. However, it does repeat the field name and type. A shorthand syntax allowing the lifetime to be declared as part of the field could make this much less burdensome. However, it isn't clear how to convey that these lifetimes match their field names and are public even when the field isn't. Something like <code><span class="nowrap"><span class="hljs-keyword">private</span> <span class="hljs-keyword">let</span> parser:</span> <span class="nowrap">Parser$<span class="hljs-keyword">borrowed</span>;</span></code> is less than ideal. The explicit syntax is being used for now until a better shorthand syntax can be found.

## What Now?

There will probably be issues with this approach. Corner cases will have to be addressed. However, this seems like a much better starting point from which to evolve Adamant’s lifetime handling. I feel this could represent a real step forward in the usability of lifetimes in a programming language.

[^rust-book]: [*The Rust Programming Language*](https://doc.rust-lang.org/1.31.1/book/) as distributed with Rust v1.31.1
[^longest]: [*The Rust Programming Language*](https://doc.rust-lang.org/1.31.1/book/) Chap. 10 Section 3 "[Validating References with Lifetimes](https://doc.rust-lang.org/1.31.1/book/ch10-03-lifetime-syntax.html#generic-lifetimes-in-functions)"
[^parser]: [*The Rust Programming Language*](https://doc.rust-lang.org/1.31.1/book/) Chap. 19 Section 2 "[Advanced Lifetimes](https://doc.rust-lang.org/1.31.1/book/ch19-02-advanced-lifetimes.html)"
