---
layout: post
title: "Inlining Recursive Functions in SpiderMonkey"
authors: "Spenser Bauman"
date: 2015-07-24
---

Mission Statement
---

SpiderMonkey must walk a fine line between code quality and compilation time.
A proper inlining heuristic is a major component in striking this balance;
too much inlining results in long compile times.
Recursive functions are an obvious case where we need to be careful.
Unrestrained inlining can result in an exponential growth in code size for
functions like naive `fibonacci`.
For this reason, SpiderMonkey will inline a recursive function at most one
level.
The inliner maintains a stack of functions inlined to get to the current
callsite under consideration.
If the candidate function is already in this stack, then SpiderMonkey vetoes
inlining.

The heuristic described above successfully prevents explosions in code size due
to recursion, however, it is easily fooled into vetoing inlining of
non-recursive functions.
A simple example is nested uses of `forEach`:

{% highlight javascript %}
function benchmark(outer) {
  outer.forEach(function (inner) {
    inner.forEach(function (v)
      max = Math.max(v, max);
    });
  });
}
{% endhighlight %}

Lets say the JIT decides to compile the `benchmark` function.
The outer call to `forEach` is inlined into `benchmark` as is the call to the
its kernel function.
The inliner then vetoes the inner call to `forEach`, since it already inlined
`forEach` once.
Though `forEach` is not written recursively, the inliner now believes it is
recursive.
This shortcoming might not be so tragic except for the fact that a deep inlining of
`benchmark` is required for good performance.
For reference, this version of `benchmark` is currently about 4x slower in
SpiderMonkey than one written using explicit for-loops.

Failure to inline both calls to `forEach` degrades performance in two ways

1. The benchmark must now perform a function call on each iteration of the outer
   loop. Due to the amount of work performed in the inner loop, this is a minor
   problem for this benchmark.
2. The compiler cannot specialize the body of `forEach` based on information
   (particularly type information) at the inner callsite.

Point two causes  the inner loop to operate over JS values with type checks and
casts rather than operating over native doubles while keeping the accumulator
`max` in a register.
In short, the propagation of type information enabled by inlining is a major
boon to many programs, and so we want inlining to work for these
pseudo-recursive functions as well.

Solution
---

From the criteria outlined above, we want to extend the inliner to better handle
these pseudo-recursive functions without using overly complicated or expensive
heuristics.
Thus the question becomes "what distinguishes recursive from pseudo-recursive
functions?"
One might expect an examination of the callgraph to be sufficient, yet the
callgraph only tells us where a function is used, not how.
In fact, SpiderMonkey's current algorithm essentially watches for cycles in the
callgraph and vetoes inlining when a cycle is found.
Fortunately, we can observe how functions are used while inlining by observing
the path traced through the callgraph (which SpiderMonkey already does).

When a function is used in a recursive manner, the program eventually reaches
a steady state in the path traced through the callgraph.
An initial call is needed to get into a recursive loop, but from then on the
program cycles through some number of functions and eventually ends up back at
the recursively defined function creating the loop.
So, rather than asking "did the inliner get into a loop," we should be asking
"did the inliner get into a loop *in the same way* as it did before?"

You might ask, what does "in the same way" mean?
The simplest possible definition simply factors in the caller of each function
when deciding whether or not to inline.
That is to say, rather than checking if the function under consideration has
already been inlined, we check whether it has been inlined into the current
caller.
This heuristic is (at least empirically) sufficient to prevent unrestrained
inlining of recursive and mutually recursive functions, while fully unfolding
the body of `benchmark` to contain no function calls.
More complicated variants can be formulated based on paths of varying lengths
leading up to the current inlining decision, but I have yet to find an example
which warrants the additional complexity.
This simplicity also eases the burden of dealing with corner cases introducing
performance regressions.

