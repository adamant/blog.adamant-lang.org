---
layout: post
title: "Basic Memory Management in Adamant"
date: 2018-12-17 22:00:00 -0500
tags: ["Adamant", "Memory Management"]
author: "Jeff Walker"
---
The Adamant programming language uses compile-time memory management (CTMM). That is, it doesn't use a garbage collector, nor does it require the programmer to do manual memory management. Instead, references are checked at compile time for safety, and the compiler determines when to free memory. There has been research on ideas like this, but there wasn't a widely used, practical programming language with compile-time memory management until Rust. Its single owner model with borrowing has demonstrated the feasibility of CTMM. Adamant is improving on the ergonomics of the single owner model and hoping to provide additional options. Rust is a systems programming language where the focus is on relatively low-level details of the hardware. Adamant is a high-level language in the vein of C♯, Java, Scala, Kotlin, and Swift. Whenever possible, it abstracts away low-level details.

## Benefits

Compile-time memory management has many advantages over manual memory management. These advantages are part of what drove the adoption of the Rust programming language. Hopefully, future systems programming languages will see this as an essential feature to include. For Adamant, however, the crucial question is: what are the benefits of its approach over a garbage collected language? As with anything, there will be tradeoffs, but for many situations, the benefits outweigh the costs. Let's look at some of those benefits.

### Unified Resource Management

CTMM uses the resource acquisition is initialization (RAII) idiom to unite the management of other computer resources with memory management. Resources such as file handles, network sockets, locks and anything else with limited availability that must be released. Garbage collected languages leave the management of these to the programmer. Object finalizers can be a backstop, but without guarantees about when or if they are run, the developer must be sure to close files and release resources as soon as they are no longer used. Several languages advocate the disposable pattern for this, but it is notoriously tricky and verbose. Enough so that some provide statements specifically to aid in resource cleanup. In Adamant, the same mechanisms that ensure safe, timely release of memory also handle these other resources. No additional patterns or constructs are needed. This eliminates bugs from failure to release resources.

### Safety

The great promise of Rust is safety. In systems programming safety is a big deal. A lot of that safety in Rust comes from the single owner memory model. Garbage collectors provide memory safety, but that isn't the only kind of safety that matters. Software bugs cost the world millions of dollars every year. We need new languages that help us write code with fewer bugs.

#### Safe Concurrency

The single owner memory management model isn't just about memory safety. It also enables safe concurrency. Concurrent use of shared mutable state is the source of most concurrency issues. Mutable state that *isn't* shared is fine. Immutable data that *is* shared is fine. When concurrent processes try to read and write to shared mutable state, that is when bad things can happen. Many languages provide locking mechanisms developers can use. But locks are easy to misuse, and it is easy to create deadlock bugs accidentally. This is such a problem that languages built for concurrency, such as Erlang, often disallow shared mutable state entirely by mandating one particular concurrency strategy. Avoiding shared mutable state is also providing some of the impetus for the growing popularity of functional languages. Adamant uses CTMM and mutability permissions to enable safe shared mutable state without locks. How is that possible? Well, it controls the sharing of mutable state between threads so that no state is ever mutable by one thread at the same time other threads have access to it. So technically, it prevents the sharing. However, it is so easy to safely pass around ownership, borrow mutability and share immutable access to state that it enables fearless use of concurrency.

#### Safe Asynchronous Programming

The rise of web programming and changes in the performance characteristics of computers have led to more asynchronous programming. New async/await features pioneered by C♯, and now being added to JavaScript, Scala, and Python help support this. While asynchronous programming can be done in a single-threaded manner, the real potential lies in allowing asynchronous code to run in multiple threads. That would mean fine-grained concurrency across every asynchronous method call boundary. Managing that concurrency is a recipe for bugs. This is a problem many asynchronous C♯ programs already have even though it isn’t widely acknowledged. Safe, fearless concurrency also means fully unleashing safe asynchronous programming.

#### Safe Iteration

While less of an issue, there are other safety issues addressed by CTMM. As one example, consider the problem of iterator invalidation. While iterating over a collection, it is unsafe to modify that collection by adding or removing items from it. Both C♯ and Java check for this condition at runtime and throw an exception. Adamant detects this at compile time and reports an error. This means less time spent debugging and fewer runtime checks and failure modes.

### Simplicity

Simplicity has long been a virtue among computer programmers. They've enshrined it in principles like "Keep it simple, stupid" (KISS), “You aren’t gonna need it” (YAGNI), and “Don’t repeat yourself" (DRY). Many wise programmers try to minimize the code running in their app. Doing so leads to improved maintainability, performance, predictability and fewer bugs. Yet every application relying on a garbage collector has a large amount of code running in the background to manage memory. It isn't uncommon that there are more lines of code in the garbage collector than in the program itself. Sure that code has been carefully written, tuned and debugged by teams of developers. Still, complaints of slow and unpredictable performance and memory usage are too common. However well written the garbage collector is, the only bug-free code is the code that doesn't exist. CTMM eliminates the garbage collector, thereby removing one more complex piece from the runtime system. It replaces it with a straightforward strategy for safe memory management.

### Optimizable

Many developers don't realize the performance impact of allocating memory. A heap allocation often has to acquire locks on shared heap management data structures. Then it must find an appropriate free space in which to allocate the memory. Compacting garbage collectors can improve this some. However, garbage collected languages encourage a programming style where most objects are allocated on the heap one by one, thereby maximizing the use of heap allocation. Sophisticated compilers perform escape analysis to determine which allocations never leave the scope of a function so they can be allocated on the stack. By contrast, the single owner memory model contains within it the information about which objects escape. Indeed, it may provide a fuller picture because the compiler's escape analysis may not span enough functions to make a correct determination. More importantly, CTMM encourages a programming style where fewer objects escape the function in which they are allocated. This means more allocations can be optimized onto the stack for maximum performance.

### Transparency

Closely related to the runtime simplicity of CTMM is its transparency. That is, it is easy for developers to see what memory management operations will be performed and where. Developers can easily predict the behavior of their code and modify that behavior when necessary.

### Enjoyable

All of these benefits aren't worth much if it is too much work and pain to write code in a language with CTMM. For that, let's look at Rust, the one language with CTMM that has seen widespread use. Every year [Stack Overflow](https://stackoverflow.com/) surveys their users on a variety of programming related topics. In the [2018 Developer Survey](https://insights.stackoverflow.com/survey/2018/#most-loved-dreaded-and-wanted), Rust was the most loved programming language. This is the third year in a row it has held that spot. This is determined by the percentage of programmers working in a language who "expressed interest in continuing to develop with it." There are many reasons why developers so appreciate Rust, but its memory management is a part of it. As the Rust book writes, "ownership is Rust’s most unique feature." Whatever challenges the single owner model brings to programming in Rust, these developers still enjoy working in it, and Adamant improves on the ergonomics of CTMM in Rust by abstracting away more of the low-level details.

## An Introduction

With those benefits, CTMM is worth considering. Let's take a look at how it works in Adamant. This is an introduction to memory management in Adamant intended for programmers familiar with strongly typed object-oriented languages like those listed before. Knowledge of memory management in Rust might be helpful, but shouldn't be necessary. This introduction only covers the parts of Adamant that have a parallel to Rust. They are an adaptation of the ideas in Rust to a high-level object-oriented language with reference types. The most radical departure is in how lifetime relationships are handled (though the [Dyon scripting language](https://github.com/PistonDevelopers/dyon) independently invented something similar). Despite that, readers familiar with Rust might be best served by taking Adamant on its own terms at first and only later comparing it to Rust. Thinking about Adamant code in terms of the equivalent Rust code can be confusing and lead to incorrect intuitions.

Please keep in mind that:

* Adamant is still under development so the syntax, especially around lifetimes, may change in the future.
* This is an introduction; it does not cover all the features of CTMM in Adamant.
* CTMM can be confusing to those unfamiliar with it but is relatively simple once understood. This post is by necessity brief and may not explain it in a way accessible to all readers. If something doesn't make sense, please check back when there is more documentation available.
* The features presented here are only a starting point for what might be possible.

## Value Types and Reference Types

Before we can talk about compile-time memory management, we need to understand the distinction between *value types* and *reference types*. A value type is allocated directly in its variable. That is, it is allocated on the stack or in the declaring object. For value types, assignment typically means the value is copied.[^move] A reference type is allocated separately from the variable. The variable is just a reference to the object that is allocated on the heap. Reference types allow for multiple variables to reference the same object. That way, an update to the object through one reference is visible from all other references to the object. For reference types, assignment copies the reference, not the object. After an assignment, there is one more variable that references the same object.

As with many languages, Adamant's primitive number types are value types. Thus simple math operations work the same as they do in other languages. There is nothing new to learn and nothing extra to deal with.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">square</span>(<span class="hljs-params">x: <span class="hljs-keyword">int</span></span>) -> <span class="hljs-keyword">int</span></span>
{
    <span class="hljs-keyword">return</span> x * x;
}
</code></pre>

In Adamant developers can declare new value types using *structs*. This is similar to C♯ structs, and Java is adding support for user-defined value types soon. Structs are useful for specific things, but most of the time, classes are used instead of structs.

<pre><code class="hljs nohighlight"><span class="hljs-class"><span class="hljs-keyword">public</span> <span class="hljs-keyword">copy</span> <span class="hljs-keyword">struct</span> <span class="hljs-title">complex_number</span></span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> real: <span class="hljs-keyword">float</span>;
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> imaginary: <span class="hljs-keyword">float</span>;

    <span class="hljs-comment">// Constructor shorthand that initializes the fields</span>
    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">new</span>(<span class="hljs-params">.real, .imaginary</span>) </span>{ }
}
</code></pre>

Although the primitive types are value types, most types in Adamant are reference types. Just like in other object-oriented languages all classes and traits are reference types (traits are akin to interfaces or protocols). Reference types are where CTMM differs from garbage collected memory management. They are the focus from here on.

## Mutability

One more thing before we deal with memory management directly, in Adamant, both objects and variables are immutable by default. For an object to be mutable, its type must be declared with the <code><span class="hljs-keyword">mut</span></code> keyword. To make a variable binding mutable declare it with <code><span class="hljs-keyword">var</span></code> instead of <code><span class="hljs-keyword">let</span></code>. Note that the mutability of objects and variables bindings are independent.

<pre><code class="hljs nohighlight"><span class="hljs-keyword">let</span> numbers1: List[<span class="hljs-keyword">int</span>] = <span class="hljs-keyword">new</span> List(); <span class="hljs-comment">// a list of ints</span>
<span class="error">numbers1.add</span>(<span class="hljs-number">236</span>); <span class="hljs-comment">// ERROR: Can't mutate non-mutable list</span>
<span class="error">numbers1 =</span> <span class="hljs-keyword">new</span> List(); <span class="hljs-comment">// ERROR: Can't modify let bindings</span>

<span class="hljs-keyword">let</span> numbers2: <span class="hljs-keyword highlight">mut</span> List[<span class="hljs-keyword">int</span>] = <span class="hljs-keyword">new</span> List();
numbers2.add(<span class="hljs-number">89</span>); <span class="hljs-comment">// adds 89 to the list</span>
<span class="error">numbers2 =</span> <span class="hljs-keyword">new</span> List(); <span class="hljs-comment">// ERROR: Can't modify let bindings</span>

<span class="hljs-keyword highlight">var</span> numbers3: List[<span class="hljs-keyword">int</span>] = <span class="hljs-keyword">new</span> List();
<span class="error">numbers3.add</span>(<span class="hljs-number">89</span>); <span class="hljs-comment">// ERROR: Can't mutate non-mutable list</span>
numbers3 = #[<span class="hljs-number">1</span>, <span class="hljs-number">2</span>, <span class="hljs-number">3</span>]; <span class="hljs-comment">// numbers refers to a list of 1, 2, 3</span>
</code></pre>

## Ownership

With that background, we can look at memory management. In Adamant, every object has a single reference that *owns* it. The owning reference determines when it will be deleted. As soon as that reference goes out of scope, the object will be deleted. Ownership is part of the type of a variable and is declared with the special lifetime <code><span class="hljs-keyword">owned</span></code>. We'll see what lifetimes are as we go along. For now, think of it as an extra annotation on the type of the variable. Lifetimes are distinguished by the lifetime operator `$`.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>(<span class="hljs-params">console: <span class="hljs-keyword">mut</span> Console</span>)</span>
{
    <span class="hljs-keyword">let</span> greeting: <span class="hljs-keyword">mut</span> <span class="before-highlight">String_Builder</span><span class="highlight">$<span class="hljs-keyword">owned</span></span> = <span class="hljs-keyword">new</span> String_Builder();
    greeting.append(<span class="hljs-string">"Hello"</span>);
    greeting.append(<span class="hljs-string">", World!"</span>);
    console.write_line(greeting.to_string());
}
</code></pre>

In the example above, a new `String_Builder` is made and assigned to the variable `greeting`. Since `greeting` is the only variable referencing the string builder, it has to be the owner. The variable's type reflects this. Declaring ownership all the time would be very tedious, but in Adamant, the type of local variables can be omitted, and the compiler will figure out the correct type. Even in cases where the type must be specified because the compiler can't figure it out, the lifetime can often still be omitted. For example, an equivalent declaration is <code><span class="nowrap"><span class="hljs-keyword">let</span> greeting =</span> <span class="nowrap"><span class="hljs-keyword">mut</span> <span class="hljs-keyword">new</span> String_Builder();</span></code>.

## Transferring Ownership

Ownership of an object can be passed to and returned from functions and passed between variables. Passing ownership is done using the <code><span class="hljs-keyword">move</span></code> keyword. Using a variable after a reference has been moved out of it is an error.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">append_testing</span>(<span class="hljs-params">builder: <span class="hljs-keyword">mut</span> String_Builder$<span class="hljs-keyword">owned</span></span>)
    -> String_Builder$<span class="hljs-keyword">owned</span></span>
{
    builder.append(<span class="hljs-string">"testing..."</span>);
    <span class="hljs-keyword">return</span> builder;
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>(<span class="hljs-params">console: <span class="hljs-keyword">mut</span> Console</span>)</span>
{
    <span class="hljs-keyword">let</span> x = <span class="hljs-keyword">mut</span> <span class="hljs-keyword">new</span> String_Builder(<span class="hljs-params"></span>);
    <span class="hljs-keyword">let</span> y = <span class="hljs-keyword">mut</span> <span class="before-highlight">append_testing(</span><span class="hljs-params"><span class="hljs-keyword highlight">move</span> x</span>);
    <span class="error">x</span>.append(<span class="hljs-string">"not allowed"</span>) <span class="hljs-comment">// ERROR: reference moved out of x</span>
    <span class="hljs-keyword">let</span> z = <span class="hljs-keyword highlight">move</span> y; <span class="hljs-comment">// Move ownership from y to z</span>
    console.write_line(z.to_string())
}
</code></pre>

In this example, the `append_testing` function receives ownership of the string builder, appends a string to it and then returns ownership of that string builder.

## Borrowing

If passing an object to a function always required moving it and thereby losing ownership, that would be very annoying. Notice the `append_testing` function had to return the string builder so it could be used afterward.  If it hadn’t, nothing further could have been done with the object. The solution to this is *borrowing*. Borrowing lets an object be used temporarily without changing which reference owns it. Borrowing is the default when passing and assigning references. With borrowing, the `append_testing` function is easier to write and use.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">append_testing</span>(<span class="hljs-params">builder: <span class="hljs-keyword">mut</span> String_Builder</span>)</span>
{
    builder.append(<span class="hljs-string">"testing..."</span>);
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>(<span class="hljs-params">console: <span class="hljs-keyword">mut</span> Console</span>)</span>
{
    <span class="hljs-keyword">let</span> x = <span class="hljs-keyword">mut</span> <span class="hljs-keyword">new</span> String_Builder();
    append_testing(x);
    console.write_line(x.to_string())
}
</code></pre>

Borrowing an object beyond when it is deleted isn't allowed.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">borrow_too_long</span>(<span class="hljs-params"></span>)</span>
{
    <span class="hljs-keyword">let</span> x: Book;
    {
        <span class="hljs-comment">// y owns the book</span>
        <span class="hljs-keyword">let</span> y = <span class="hljs-keyword">new</span> Book(<span class="hljs-string">"The Fable of the Dragon-Tyrant"</span>);
        x = y; <span class="hljs-comment">// x borrows the book</span>
        <span class="hljs-comment">// the book is deleted when y goes out of scope at `}`</span>
    <span class="error">}</span> <span class="hljs-comment">// ERROR: object referenced by x borrowed too long</span>
    x.read();
}
</code></pre>

In the function above, the reference `y`owns the `Book` object. At the end of the block `y` is declared in, it goes out of scope and automatically deletes the `Book` object. However, `x` has borrowed that object and still has a reference to it that is used later in `x.read()`. That call could crash when it tried to use the deleted `Book` object. Instead, the compiler tells us it is invalid. Ownership and borrowing thereby provide *memory safety*. In low-level languages, there are lots of ways to cause errors by misusing memory. In memory safe languages like C♯, Java, Rust and Adamant, that isn't possible.

## Mutability when Borrowing

The borrowing rules provide not only memory safety, but also *concurrency safety*. They prevent *data races*. A data race occurs when multiple threads try to read and write the same place in memory without correct locking. Data races can result in nondeterministic behavior and reading invalid data. To prevent that, only one mutable borrow of an object can be active at a time. Alternatively, multiple immutable borrows are allowed if there is no mutable reference to the object. When an object is borrowed, the owning reference is temporarily restricted as well.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">add_pairs</span>(<span class="hljs-params">x: List[<span class="hljs-keyword">int</span>], y: List[<span class="hljs-keyword">int</span>]</span>)
    -> List[<span class="hljs-keyword">int</span>]$<span class="hljs-keyword">owned</span></span>
{
    <span class="hljs-keyword">return</span> x.zip(y, <span class="hljs-function"><span class="hljs-keyword">fn</span>(<span class="hljs-params">a, b</span>)</span> <span class="hljs-keyword">=></span> a + b).to_list();
}

<span class="hljs-comment">// Takes an element from x and adds it to y</span>
<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">move_element</span>(<span class="hljs-params">x: <span class="hljs-keyword">mut</span> List[<span class="hljs-keyword">int</span>], y: <span class="hljs-keyword">mut</span> List[<span class="hljs-keyword">int</span>]</span>)</span>
{
    <span class="hljs-keyword">let</span> n = x.remove_at(<span class="hljs-number">0</span>);
    y.add(n);
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>(<span class="hljs-params"></span>)</span>
{
    <span class="hljs-keyword">let</span> numbers: <span class="hljs-keyword">mut</span> List[<span class="hljs-keyword">int</span>] = #[<span class="hljs-number">1</span>, <span class="hljs-number">2</span>, <span class="hljs-number">3</span>]; <span class="hljs-comment">// initialize list</span>

    <span class="hljs-comment">// immutably borrow numbers twice. sum == #[2, 4, 6]</span>
    <span class="hljs-keyword">let</span> sum = <span class="hljs-keyword">mut</span> add_pairs(numbers, numbers);

    <span class="hljs-comment">// mutably borrow numbers and sum once each</span>
    move_element(<span class="hljs-keyword">mut</span> numbers, <span class="hljs-keyword">mut</span> sum);

    <span class="hljs-comment">// can't mutably borrow numbers twice</span>
    move_element(<span class="error"><span class="hljs-keyword">mut</span> numbers</span>, <span class="error"><span class="hljs-keyword">mut</span> numbers</span>); <span class="hljs-comment">// ERROR</span>
}
</code></pre>

## The Forever Lifetime

Ownership and borrowing are sufficient for most functions. However, some values exist for the lifetime of the program and will never be deleted. String literals are the most common example. For values like that, there is the special lifetime <code><span class="hljs-keyword">forever</span></code>. Borrowing from an object with the lifetime <code><span class="hljs-keyword">forever</span></code> is the same as borrowing from an owned object.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>(<span class="hljs-params">console: <span class="hljs-keyword">mut</span> Console</span>)</span>
{
    <span class="hljs-keyword">let</span> greeting: <span class="before-highlight hljs-built_in">String</span><span class="highlight">$<span class="hljs-keyword">forever</span></span> = <span class="hljs-string">"Hello, World!"</span>;
    console.write_line(greeting); <span class="hljs-comment">// borrows the string</span>
}
</code></pre>

## Fields

So far, we’ve seen how Adamant manages memory in functions. Now let's look at how to manage memory in classes. In object hierarchies, the most common relationship between objects is composition (as opposed to aggregation). In composition, the child object can’t exist independent of the parent object, for example, a house (parent) is composed of its rooms (children). For this reason, the default lifetime for class fields is <code><span class="hljs-keyword">owned</span></code>.

<pre><code class="hljs nohighlight"><span class="hljs-class"><span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">House</span></span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> driveway: Driveway; <span class="hljs-comment">// implicit $owned</span>
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> rooms: List[Room]; <span class="hljs-comment">// implicit List[Room]$owned</span>
}
</code></pre>

## Lifetime Constraints

A lifetime is an abstract idea of how long an object or reference will exist when the program is running. The borrowing rules require that the lifetime of all borrowed references is less than that of the object being borrowed. Sometimes objects are borrowed in more sophisticated ways, and it is necessary to spell out the relationship between the lifetimes of a function's parameters and return value. We do this with lifetime relationship operators.

<pre><code class="hljs nohighlight"><span class="hljs-class"><span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">Car</span></span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> model_year: Year;
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> tires: List[Tire]; <span class="hljs-comment">// implicit List[Tire]$owned</span>

    <span class="hljs-comment">// the special parameter "self" is like "this"</span>
    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">oldest_tire</span>(<span class="hljs-params"><span class="hljs-keyword">self</span></span>) -> <span class="before-highlight">Tire</span><span class="highlight">$&lt; <span class="hljs-keyword">self</span></span>
    </span>{
        <span class="hljs-keyword">return</span> tires.order_by(<span class="hljs-function"><span class="hljs-keyword">fn</span>(<span class="hljs-params">t</span>)</span> <span class="hljs-keyword">=></span> t.replaced_on).first();
    }
}
</code></pre>

The `oldest_tire` function returns the tire that was replaced longest ago. The car object still owns the tire; the `oldest_tire` function is returning a borrowed reference to that tire. The type <code>Tire$&lt; <span class="hljs-keyword">self</span></code> is read "Tire with a lifetime less than self" and indicates the relationship between the lifetimes of the tire object and the car object. The compiler uses that to ensure memory and concurrency safety by preventing modification or deletion of the car object as long as the tire is borrowed. In this case, it isn’t necessary to spell out the lifetime relationships as we did.  In Adamant, there are a few simple rules for lifetime elision that allow the omission of lifetimes in common cases. One of those cases is methods, for which the assumption is that the return value is borrowed from inside the object. So in this case, the function could be declared <code><span class="hljs-function"><span class="nowrap"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">oldest_tire</span></span><span class="nowrap">(<span class="hljs-keyword">self</span>)</span> <span class="nowrap">-> Tire</span></span></code>. There are cases where that isn't possible though.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">second_car</span>(<span class="hljs-params">first: Car, second: Car</span>) -> Car$&lt; second</span>
{
    first.drive();
    <span class="hljs-keyword">return</span> second;
}
</code></pre>

In the `second_car` function, it isn't possible to leave out the lifetime relationship. Based on the parameters, it's possible the returned car is related to either the first or second car. Rather than guessing which, the compiler requires that it be specified.

## Lifetime Parameters

Occasionally, lifetime relationships exist that can't be specified using just simple relationships. When that happens, we use lifetime parameters. A lifetime parameter provides an abstract lifetime that may not be the lifetime of any actual object but explains the relationship between the lifetimes of the various objects and references. Lifetime parameters are generic parameters prefixed with the lifetime operator. Like other generic parameters, their value can often be inferred when calling a function.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title before-highlight">newer_car</span><span class="highlight">[$<span class="hljs-symbol">a</span>]</span>(c1: Car$> a, c2: Car$> a) -> Car$&lt; a</span>
{
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">if</span> c1.model_year >= c2.model_year <span class="hljs-keyword">=></span> c1
           <span class="hljs-keyword">else</span> <span class="hljs-keyword">=></span> c2;
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>(<span class="hljs-params"></span>) -> <span class="hljs-keyword">int</span></span>
{
    <span class="hljs-keyword">let</span> c17 = <span class="hljs-keyword">new</span> Car(<span class="hljs-number">2017</span>);
    <span class="hljs-keyword">let</span> c18 = <span class="hljs-keyword">new</span> Car(<span class="hljs-number">2018</span>);
    <span class="hljs-keyword">let</span> newer = newer_car(c17, c18); <span class="hljs-comment">// lifetime $a is inferred</span>
    <span class="hljs-keyword">return</span> newer.model_year;
}
</code></pre>

As you might guess from its name, the `newer_car` function returns the car with the newer model year. The borrowed reference must not outlive either car, since either could be returned. To express this, we introduce a lifetime parameter <code>$<span class="hljs-symbol">a</span></code> that represents a lifetime in between the lifetimes of the objects passed in and the borrowed reference returned. Note that this lifetime may, or may not, actually be equal to the lifetime of one of the cars or the returned reference.

Lifetime parameters can also be useful for putting borrowed references into the fields of objects.

<pre><code class="hljs nohighlight"><span class="hljs-class"><span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">Employee</span>[$<span class="hljs-symbol">manager</span>]</span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> name: String$<span class="hljs-keyword">forever</span>;
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> boss: Employee?$<span class="hljs-symbol">manager</span>;

    <span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">new</span>(<span class="hljs-params">.name, .boss = <span class="hljs-literal">none</span></span>) </span>{ }
}

<span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">main</span>(<span class="hljs-params">console: <span class="hljs-keyword">mut</span> Console</span>)</span>
{
    <span class="hljs-keyword">let</span> boss = <span class="hljs-keyword">new</span> Employee(<span class="hljs-string">"Alice"</span>);
    <span class="hljs-keyword">let</span> employee = <span class="hljs-keyword">new</span> Employee(<span class="hljs-string">"Bob"</span>, alice);
    console.write_line(
        <span class="hljs-string">"<span class="hljs-subst">\(boss.name)</span> is <span class="hljs-subst">\(employee.name)</span>'s boss."</span>);
    <span class="hljs-comment">// prints "Alice is Bob's boss."</span>
}
</code></pre>

Here we use a lifetime parameter to borrow a reference to an employee's boss. Of course, some employees don't have a boss (they are top dog) so the reference to the boss is optional as indicated by the `?` after the type. When creating employee objects, their lifetimes will have to be less than the lifetime of the boss object (unless we were to assign them a new boss).

## Remaining Cases

Experience with Rust shows that a significant amount of code can be written easily with nothing more than single ownership and borrowing. Explicit lifetime parameters and constraints are needed only occasionally. Adamant enables this approach in a way that is simpler and more familiar to many programmers. In Adamant, the majority of code requires minimal annotations added to what the same program would be in a garbage collected language. For example, in MVC architecture web applications, most memory allocations are already scoped to the lifetime of the web request. Developers should quickly become proficient enough in Adamant that compile-time memory management is an insubstantial burden most of the time.

However, single ownership with borrowing does not support all use cases. It doesn't support object graphs that can’t be represented as a tree or that include references to parent nodes. We need other approaches for these. Assuming something like 80% of cases are handled, we need a solution for the remaining 20%. It's important to remember that a perfect compile-time solution isn’t required. If 90% to 95% of cases are handled at compile time, we can deal with the remaining manually or with reference counting. The Swift language has demonstrated the viability of an automatic reference counting approach. In future posts, I’ll discuss the problematic cases in more detail and look at some ideas for how to address them.

[^move]: Adamant also supports value types with move semantics. This is the default for structs in Rust. However, in Adamant, value types with move semantics are rare.
