---
layout: post
title: "Reachability Annotations"
date: 2020-03-07 09:15:00 -0800
tags: ["Adamant", "Language Design", "Memory Management"]
author: "Jeff Walker"
---
Rust's lifetime annotations are one of the more confusing features of the language. It's planned that Adamant will use a memory management strategy similar to Rust's. However, Adamant needs to be easier to use. As such, I'm working hard to come up with a better alternative than lifetime annotations for Adamant. That could be just an easier or clearer syntax for the same thing or a radical rethinking of how reference lifetimes are handled at function calls. The latest incarnation of those ideas is what I'm calling *reachability annotations*.

The original Adamant language specification contained something very similar to Rust's lifetime annotations except with different operators and the ability to inline the constraints next to the variable types. A refinement of that was described in the first blog post about memory management in Adamant as [lifetime constraints]({% post_url 2018-12-17-basic-memory-management-in-adamant %}). From there, the design evolved into "[A New Approach to Lifetimes in Adamant]({% post_url 2018-12-26-new-approach-to-lifetimes %})". It was based around which parameters' lifetimes went into creating the return value. But, sometimes, that isn't sufficient. Lifetime constraints can still be needed. Reachability annotations are the next step in the design evolution.

Rather than describing lifetimes and their relationships, reachability annotations indicate which objects might be directly or indirectly referenced by another. For the moment, the syntax for this is the reachability operator `~>`. Given two variables, `x` and `y`, the expression `x ~> y` would indicate that `y` is potentially reachable from `x`. There are two ways that could be the case. Either the variable `x` could directly reference the object referenced by `y`, or `x` could reference an object which referenced the object referenced by `y`. Of course, there can be more levels of indirection. Any object reachable by following a series of references from `x` could be the one referencing `y`. Reachability annotations aren't just for variables. They mostly appear in types. Given some type `T`, the type `T ~> y` is the type of things with type `T` that might reference directly or indirectly the value of the variable `y`.

Stating reachability annotations can be awkward. I haven't found an easy way to read `x ~> y` without reversing it to "`y` is reachable from `x`". Thinking about the graph formed by objects and their references to each other can be helpful. Then each object defines a subgraph composed of all the objects reachable from it. The reachability operator indicates that the subgraph of the first object may include the subgraph of the second object. To get a better understanding of this, we'll work through some examples.

## Annotations on Return Types

The primary place reachability annotations are needed is on return types. Within the body of a function, the compiler can infer reachability. When calling another function, the compiler couldn’t infer reachability unless it were to analyze the program as a whole. That can be slow. It can also lead to unexpected behavior as the implementation of one function can affect the inferred reachability in another. Without fixed reachability on function return types, a change to the implementation of a function could break backward compatibility in unobvious ways.

For ease of comparison, I'll continue to use the same example I used in my previous posts. A function that takes two strings and returns the longer of the two. This is straight forward to think about with reachability annotations. The returned reference could reference either string passed to the function. To annotate that something could reference multiple things, we list them separated by commas.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">longest</span>(<span class="hljs-params">x: String, y: String</span>) -> String <span class="highlight">~> x, y</span></span>
{
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">if</span> x.grapheme_count() > y.grapheme_count()
                => x
           <span class="hljs-keyword">else</span>
                => y;
}
</code></pre>

The highlighted code expresses that the returned `String` reference could reference the value of either `x` or `y`. Of course, it can't reference both. The reachability operator is not a promise that something will be referenced. Instead, it indicates something *may* be referenced.

The longest function is a simple example, but more complex reachability type annotations are possible. In these examples, note that `Tuple` is a value type while `List[T]` is a reference type. Consequently, the list reference must be annotated with the `owned` reference capability. This indicates that the caller of the `make_list` function has ownership of this list, and it will be deleted when they are done with it. If you're familiar with previous versions of Adamant, you'll notice that ownership used to be a lifetime, but is now a reference capability. With the switch to reachability annotations, the concept of a lifetime doesn't make as much sense anymore.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">make_tuple</span>(<span class="hljs-params">x: String, y: String</span>)
    -> Tuple[String <span class="highlight">~> x</span>, String <span class="highlight">~> y</span>]</span>
{
    <span class="hljs-keyword">return</span> #(x, y);
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">make_list</span>(<span class="hljs-params">x: String, y: String</span>)
    -> <span class="hljs-keyword">owned</span> List[String <span class="highlight">~> x, y</span>]</span>
{
    <span class="hljs-keyword">return</span> #[x, y];
}
</code></pre>

## Parameter Reachability Annotations

Some functions mutate their parameters. When that happens, it is necessary to annotate the function with the possible change in reachability. Consider the `assign_into` function, which takes a string and a reference to a variable of type `String` and assigns the string into the variable using the dereference operator `^`. For this to be allowed, we must declare that the string may be reachable from the referenced variable after the function returns. This is a side effect of the function and is annotated as an effect using the `may` keyword.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">assign_into</span>(<span class="hljs-params">value: String, variable: <span class="hljs-keyword">ref</span> <span class="hljs-keyword">var</span> String</span>)
    <span class="hljs-keyword">may</span> <span class="highlight">variable ~> value</span></span>
{
    ^variable = value;
}
</code></pre>

## "Reachable From" Annotations

Up to this point, we've looked at annotations indicating which objects may be reachable from a reference. However, sometimes, the important information is what objects may reference the object in question. Methods often return objects that are still referenced by the object the method was called on. This needs to be annotated so the compiler can correctly manage the memory of such objects. This is done using the reverse reachability operator `<~`. An expression `x <~ y` can be read, "`x` may be reachable from `y`". In the example below, this is used to indicate that the `Tire` object returned from the `oldest_tire` method could still  be referenced by the `Car` object.

<pre><code class="hljs nohighlight"><span class="hljs-class"><span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">Car</span></span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> model_year: Year;
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> tires: <span class="hljs-keyword">owned</span> List[Tire];

    <span class="hljs-comment">// the special parameter "self" is like "this"</span>
    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">oldest_tire</span>(<span class="hljs-params"><span class="hljs-keyword">self</span></span>) -> Tire <span class="highlight">&lt;~ <span class="hljs-keyword">self</span></span>
    </span>{
        <span class="hljs-keyword">return</span> .tires.order_by(<span class="hljs-function"><span class="hljs-keyword">fn</span>(<span class="hljs-params">t</span>)</span> <span class="hljs-keyword">=></span> t.replaced_on).first();
    }
}
</code></pre>

One could imagine writing the return type of the `oldest_tire` method as `self ~> Tire`. However, that would make types difficult to read because one wouldn't know if the type name came first until finding the reachability operator. Using the reverse reachability operator ensures the type comes first. For readability, the Adamant language requires that in a reachability expression between a type and a variable, the type must appear on the left-hand side.

## Reachability in Classes

One area where reachability annotations need further development is when dealing with complicated relationships between the fields of classes. When a lifetime annotation would be required on a struct in Rust, how is that handled with reachability annotations? In the previous version of Adamant, this was handled by introducing named lifetimes as part of the class, similar to associated types. Something similar may be required with reachability annotations. Alternatively, if those correspond to specific fields, then it may be sufficient to indicate that those fields have separately tracked subgraphs. The example below shows one possible syntax for that.

<pre><code class="hljs nohighlight"><span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">Context</span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> text: <span class="highlight">~></span> String;

    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">new</span>(<span class="hljs-params">.text</span>)</span> {}
}

<span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">Parser</span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> context: <span class="highlight">~></span> Context;

    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">new</span>(<span class="hljs-params">.context</span>)</span> { }

    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">parse</span>(<span class="hljs-params"><span class="hljs-keyword">self</span></span>)
        -> Result[<span class="hljs-keyword">never</span>, String <span class="highlight">~> context.text</span>]</span>
    {
        <span class="hljs-keyword">return</span> Error(<span class="hljs-keyword">self</span>.context.text.slice(<span class="hljs-number">1</span>..);
    }
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-title">parse_context</span>(<span class="hljs-params">context: Context</span>)
    -> Result[<span class="hljs-keyword">never</span>, String <span class="highlight">~> context.text</span>]</span>
{
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">new</span> Parser(context).parse();
}
</code></pre>

Here the reachability operator is used as a unary prefix operator to indicate that the field name can be used to refer to the subgraph reachable from that object. Then the member access operator is used within reachability expressions to refer to these subgraphs. Thus the `parse_context` function can express that the `context` will not be reachable from the return value, but the context's `text` may be.

## Next Steps

Reachability annotations need further work. Reachability in classes isn't well developed and may need a better syntax. Reachability annotations may be confusing when used with mutable variables. It also isn't clear how memory management compiler errors can be clearly expressed when using reachability annotations. In Rust, such errors refer to the lifetime of various references. With reachability annotations, the concept of a lifetime doesn't exist in the syntax of the language. There is only reachability. Can error messages be clearly stated in terms of reachability?

Despite the work still needed, reachability annotations seem like a good step forward toward creating a more developer-friendly version of compile-time memory management. They are mostly isomorphic to the previous approach taken by Adamant. Thus I'm confident they can be made to work. Yet, I think they are much easier for the developer to think about and reason about. Even if Adamant isn't able to develop this idea into a production language, hopefully reachability annotations can be an inspiration for other languages.
