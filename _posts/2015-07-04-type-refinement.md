---
layout: post
title: "Functional Programming in SpiderMonkey Redux"
authors: "Spenser Bauman"
date: 2015-07-04
---

In his blog post [Two Reasons Functional Programming Is Slow in
SpiderMonkey](http://rfrn.org/~shu/2013/03/20/two-reasons-functional-style-is-slow-in-spidermonkey.html),
Shu-yu Guo breaks down some of the issues SpiderMonkey has when optimizing
functional style programs.
Since I am working on this very issue as a Mozilla intern, I am going to try to
give an updated account of what SpiderMonkey does wrong and some of the initial
fixes I am working on.

As a running example, I'll make use of Shu's `Array#forEach` benchmark

{% highlight javascript %}
function benchmark(n, iters, f) {
    var outer = [];
    for (var i = 0; i < n; i++) {
        var inner = [];
        for (var j = 0; j < n; j++)
            inner.push(Math.random() * 100);
        outer.push(inner);
    }
    var start = Date.now();
    for (var i = 0; i < iters; i++)
        f(outer);
    return Date.now() - start;
}

Array.prototype.myForEach = function (f) {
    for (var i = 0; i < this.length; i++) {
        f(this[i]);
    }
}

function doForEach(outer) {
    var max = -Infinity;
    outer.myForEach(function (inner) {
        inner.myForEach(function (v) {
            if (v > max)
                max = v;
        });
    });
}

console.log(benchmark(50, 50000, doForEach));
{% endhighlight %}

and explain why it is nearly 3x slower than the more imperative version.

{% highlight javascript %}
function doForEach(outer) {
    var max = -Infinity;
    for (var i = 0; i < outer.length; i++) {
        var inner = outer[i];
        for (var j = 0; j < inner.length; j++) {
            var v = inner[j];
            if (v > max)
                max = v;
        }
    }
}
{% endhighlight %}

Now, SpiderMonkey does quite a good job optimizing the second version, producing
code where the inner loop operates over contiguous arrays of `Double`s without
intervening type tests.
The code generated for the first version is markedly worse due to the
interactions of two shortcomings of SpiderMonkey:

1. Poor inlining heuristics for recursive functions
2. Poor type specialization of certain operations

Problem 1 is a result of the conservative nature of SpiderMonkey, which must
balance code quality with compile time.
SpiderMonkey's current heuristic for preventing unrolling of recursive functions
will only inline a function once, even if the function is not recursive.
This means that the outer call to `myForEach` (and its kernel) is inlined into
`doForEach`, and a function call is emitted for the inner call to `myForEach`.

Unfortunately, SpiderMonkey cannot generate optimized code for `myForEach`
without inlining it into the caller, which bring us to problem 2.
SpiderMonkey tracks the types of function arguments and the result types of
JavaScript operations and uses that information to generate code specialized to
those types.
This benchmark invokes `myForEach` on arrays of double and arrays of arrays of
doubles, requiring SpiderMonkey to generate code general enough to handle both
possibilities.

Somewhat annoyingly, a simple change to the code fixes both of these issues:
simply duplicate `myForEach` and use a different version at each call site.
Now, SpiderMonkey inlines both calls and has type information specific to the
call site.

{% highlight javascript %}
Array.prototype.myForEach = function (f) {
    for (var i = 0; i < this.length; i++) {
        f(this[i]);
    }
}

Array.prototype.myForEach2 = function (f) {
    for (var i = 0; i < this.length; i++) {
        f(this[i]);
    }
}

function doForEach(outer) {
    var max = -Infinity;
    outer.myForEach(function (inner) {
        inner.myForEach2(function (v) {
            if (v > max)
                max = v;
        });
    });
}
{% endhighlight %}

This version is only 20% slower than using manual `for`-loops.
While this works, it is unsatisfying and does not generalize well - we expect to
use `myForEach` at many locations in practice, requiring many duplicates.

What if we fix the inliner?
---

As a simple test, turn off the inlining restriction on recursive functions.
Somewhat surprisingly, this does not have the desired effect; though both calls
to `myForEach` are inlined, the benchmark actually slows down.
Upon inspection, the generated code for each use of `myForEach` is not
specialized to the type information available from `doForEach`.
In particular, the results of array accesses are not specialized, requiring
unbox and type check operations on all array accesses.

Problem 2 rears its ugly head again.
Walking through the SpiderMonkey code which generates array access instructions,
we see that array access generate instructions based only on the *observed
results* of element accesses - the type(s) of the array is never consulted when
deciding the result type.
By inlining both calls to `myForEach` into `doForEach` sufficient type
information should be available to specialize the expected result type of
`this[i]` in `myForEach`.

How can this refinement be accomplished in SpiderMonkey?
SpiderMonkey tracks the set of types which may result from each operation.
These sets are generated by observing the results during execution and may be
invalidated later.
Fortunately, SpiderMonkey tracks, as part of an object's type, the set of types
produced by indexing said object by an integer.
For an object type \\( o \\), this set is denoted as \\( \\mathrm{index}(o) \\).
Then, given the type set for the input to an indexing operation, we expect the
result type set to be

\\[ \\mathrm{types} \\left( a[i] \\right) = \\bigcup_{o ~ \\in ~ \\mathrm{types}(a)} \\mathrm{index}(o) \\]

When this reasoning is incorporated into SpiderMonkey, the `forEach` example
performs as well as the version with duplicated code.
This version is still ~20% slower than manual use of `for`-loops, but that is
still a sight better than 300% slower.


