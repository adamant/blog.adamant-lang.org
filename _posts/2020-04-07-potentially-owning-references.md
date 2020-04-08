---
layout: post
title: "Potentially Owning References"
date: 2020-04-07 21:30:00 -0700
tags: ["Adamant", "Memory Management", "Language Design"]
author: "Jeff Walker"
---
The Rust language has shown that there are interesting points in the space of possible memory management strategies that haven't been explored. Most memory management approaches can be categorized as manual, garbage collected, reference counted, or region based. But other options and combinations are possible. The Lobster language is exploring a unique combination of reference counting and compile-time analysis. The "As Static As Possible" approach described by Proust is also interesting.[^ASAP] His paper also does a good job of describing the other memory management approaches and placing them in one possible space of categories. I've been exploring the space of compile-time memory management (CTMM) approaches for use in the Adamant language. It will have CTMM similar to Rust, but is meant to be easier to use. One of the most interesting and potentially useful ideas I've found is that of combining compile-time memory management with a bit flag for ownership. Something I'm calling potentially owning references.

## Potentially Owning References

Each language with CTMM has its own variation on the idea. In general though, they have owning references and borrowed references. An owning reference is one that owns the object it references and is responsible for deleting it. When an owning reference is passed to or returned from a function, there are no other references to the object. The owing reference is a unique reference. Owning references can be safely deleted or put into longer-lived objects. There is no risk that another reference will become invalid or delete the object. Within a function, additional references can be created to the same object as an owning reference. These borrowed references are temporary aliases of the owning reference. The compiler then ensures that all borrowed references are gone before the owning reference can be deleted or given away. For many functions and fields it is clear that each reference should either be owning or borrowed.. But sometimes, the caller would like to determine which way a function is used. Potentially owning references enable this by creating another reference type that acts as a hybrid of an owning and borrowed reference.

A potentially owning reference carries with it a one-bit flag which indicates whether that reference has ownership of the object. If the reference has ownership, then it will need to be handled like an owning reference and deleted. If the reference is borrowed, how long it exists may need to be limited to keep memory safe. Consequently, potentially owning references have the limitations of both owning and borrowed references enforced on them by the compiler. Any borrowed references created from it must be gone before it can be deleted or given away. Of course, when it would be deleted, the language inserts a check that deletes it only if the ownership flag is set. Since the reference could actually be borrowed, it must be treated as potentially being bounded by the owning reference it was borrowed from. A potentially owning reference stored into an object will require the lifetime of that object to be limited by the lifetime of the reference. Of course, the caller of a function doing that may know that the reference is owning and therefore, no limit is placed on the object.

### Examples

Consider the implementation of an immutable string type. It will hold an immutable array of Unicode characters (codepoints). Many string objects will own the array they hold. However, to avoid constantly making copies of the string data, we'd like to be able to share the character array between strings. For example, when we take a substring of a string, it would be ideal if we didn't have to copy the characters but could borrow a reference to the character array and use an offset and length to track which slice of that array was the current string. In that case, the array reference should be borrowed. We could implement this with two different subclasses of string. One subclass for regular strings that own their character array and another subclass for strings that borrow their character array. That would be a lot of code duplication and complexity. Instead, we can use a potentially owning reference. If that was done, the string class data could be declared like this.

<pre><code class="hljs nohighlight"><span class="hljs-class"><span class="hljs-keyword">public</span> <span class="hljs-keyword">class</span> <span class="hljs-title">String</span></span>
{
    <span class="hljs-keyword">public</span> <span class="hljs-keyword">let</span> length: <span class="hljs-built_in">size</span>;
    <span class="hljs-keyword">let</span> start: <span class="hljs-built_in">size</span>;
    <span class="hljs-keyword">let</span> data: <span class="hljs-keyword">uni</span> Array[<span class="hljs-built_in">codepoint</span>]; <span class="hljs-comment">// potentially owning reference</span>
}</code></pre>

This example uses the keyword `uni`, an abbreviation for "unique", to mark the character array as a potentially owning reference. This may not be the keyword used in the final version of the Adamant language. It was chosen because even though such a reference could actually be borrowed, the developer may generally treat it as if it were an owning reference. However, it isn't. So to avoid confusion, another word is needed. The word "unique" conveys that the reference can be treated as if there were no other references to the same object without directly implying ownership. It should also be noted that, in Adamant, arrays directly support slicing, and strings are implemented more efficiently using structs. This example is written to explain potentially owning references clearly.

With the string fields declared this way, we can create new strings that own their data. For example, the `append` method returns a new string that owns a new character array. The method return type `uni String` indicates that it returns a potentially owned string object. It is not about the type of the character array inside that object.

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">append</span>(<span class="hljs-params"><span class="hljs-keyword">self</span>, other: String</span>) -> <span class="hljs-keyword">uni</span> String</span>
{
    <span class="hljs-keyword">let</span> newLength = .length + other.length;
    <span class="hljs-keyword">let</span> newData = <span class="hljs-keyword">mut</span> <span class="hljs-keyword">new</span> Array[<span class="hljs-built_in">codepoint</span>]();
    .data.copy_to(newData, .start, <span class="hljs-number">0</span>, .length);
    other.data.copy_to(newData, other.start, .length, other.length);
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">new</span> String(<span class="hljs-keyword">move</span> newData, <span class="hljs-number">0</span>, newLength);
}</code></pre>

The `String` class also allows for borrowed string data. For example, the `substring` method returns a new string that borrows the character array of the source string. The method return type `uni String <~ self` indicates it returns a potentially owned string object whose data may be reachable from the object `self`. The returned string needs to be deleted before the self string to ensure memory safety. (See the previous post for a full explanation of [reachability annotations]({% post_url 2020-03-07-reachability-annotations %}){:class="internal"}.)

<pre><code class="hljs nohighlight"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">fn</span> <span class="hljs-title">substring</span>(<span class="hljs-params"><span class="hljs-keyword">self</span>, start: <span class="hljs-built_in">size</span>, length: <span class="hljs-built_in">size</span></span>)
    -> <span class="hljs-keyword">uni</span> String <~ <span class="hljs-keyword">self</span>
    <span class="hljs-keyword">requires</span> start + length <= .length</span>
{
    <span class="hljs-keyword">return</span> <span class="hljs-keyword">new</span> String(.data, .start + start, length);
}</code></pre>

Potentially owning references combined with immutable references allow for even more flexibility. Consider the declaration below of a string constant. The character data for that string should be stored once in memory. A new array shouldn't be allocated each time this code is run. It is safe to assign constant data to immutable potentially owning references. Since they are immutable, it is impossible for someone to mutate the constant data through them. Since they can borrow data, it won't attempt to delete the constant data.

<pre><code class="hljs nohighlight"><span class="hljs-keyword">let</span> example: <span class="hljs-keyword">const</span> String = <span class="hljs-string">"This is my string."</span>;
</code></pre>

## Compile-Time vs. Runtime

I said Adamant used CTMM, but potentially owning references rely on a flag that must be passed with the reference and checked before deleting the object they reference. Does that mean they shouldn't count as CTMM? To me, the purpose of CTMM is to provide safety and efficiency in memory management. Today most popular languages rely on the purely runtime memory management of garbage collection. Yet, [garbage collection is a hack]({% post_url 2018-11-28-garbage-collection-is-a-hack %}){:class="internal"}. At the moment it is a very necessary hack that makes programming much more enjoyable and productive. I don't think programming languages a thousand years from now will use garbage collection. Language designers will have figured out memory management strategies that use compile-time rules possibly combined with some runtime tracking in a way that elegantly manages memory efficiently with little added burden to the programmer. That is the direction new CTMM languages should be headed in. It is fine if some runtime operations are needed, as long as those are efficient and elegant. Indeed, Rust, the poster child for CTMM languages, sometimes needs drop flags to handle function bodies with code paths that conditionally drop references.

## Implementation

Potentially owning references can be implemented fairly simply and efficiently on modern hardware. The ownership flag needed is not a property of the object itself. Different potentially owning references could refer to the same object, and only one of them will have ownership of that object. A flag must be stored with each reference. How can that be done efficiently? It is important to remember that CPUs are now much faster than memory access, so a few extra instructions needed to access memory won't have much impact. Potentially owning references can be implemented using tagged pointers. Assuming all object pointers are aligned to two-byte or larger memory boundaries, the lowest bit of a pointer will always be zero. The lowest bit can then be used to store the ownership flag. This allows the flag to be passed to functions and stored in objects as part of the pointer without any additional memory or operations. When dereferencing such a pointer, the bottom bit can be masked off before memory access. That is a single extra non-branching instruction before accessing memory. When objects should be deleted, the delete is conditioned on the bottom bit of the pointer.

## A Starting Point

Designing a new CTMM strategy is challenging. Taking existing designs and altering them slightly tends to produce invalid, unsafe approaches. Finding a new one requires simultaneously changing multiple things to produce a new consistent design. Potentially owning references enable a new degree of flexibility in CTMM by allowing a function or object to either borrow or take ownership of a reference. While they are only a single piece of the puzzle, potentially owning references could be the foundation of another important approach to memory management in programming.

[^ASAP]: Proust, Raphaël L. “[ASAP: As Static As Possible Memory Management.](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-908.pdf)” July 2017.

*[CTMM]: Compile-Time Memory Management
