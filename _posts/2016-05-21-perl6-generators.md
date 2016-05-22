---
title:  "Generator Elucidator"
date:   2016-05-21 08:56:00
description: Perl 6 alternatives to Python and JavaScript generators.
---

Generators are an interesting trope of programming languages that have been around for a long time but have just recently become a popular tool for the modern programmer. Python has had generators for some time but JavaScript just recently gained them in ES2015.  What exactly is a generator in these languages and does Perl 6 support something like it?  It turns out Perl 6 has a number of features that—though not a one-to-one comparison—provide the same or more expressiveness.

## Python & JS Generators

Python [generators](https://www.python.org/dev/peps/pep-0255/) were added back in 2001.  To define a generator you declare just as you would a regular function (`def foo():`) but instead of using `return` you use the `yield` keyword.  This allows you to pass a value to the caller of your function and then pause the execution of your generator function until it is called again.  At which point, it will proceed from just following the previous `yield` statement until it reaches another `yield` or the end of the function.

Generators like this (and JavaScript's) are known as [semi-coroutines](https://en.wikipedia.org/wiki/Coroutine#Comparison_with_generators) and this is because unlike full coroutines, they don't pass to some other coroutine, they merely return intermediary values to the caller's scope.  While seemingly limited, this pattern is quite useful for creating iterators in a highly-customizable way. For the terminology nut: a generator *function* builds a generator *iterator* which can then be called and used just as other iterables.  In both Python and JavaScript this is done by calling the `next` function on the iterator (in JavaScript's case it's actually a method) or via the `for` loop.

> **Note**: Generators are also used as a means of representing asynchronous control flow but I will leave that to you to find out why Perl 6 already has [numerous](http://doc.perl6.org/language/concurrency) builtin features for accomplishing that. In this article, I will focus on representing generator iterators and the ability to pause execution.

## Face Value

While the generator pattern can be useful in the right context, its direct equivalent is not present in Perl 6. However, there are a number of other tools at your disposal that cover all of the places you would want to use a generator.  First, let's take a look at an example generator function.

```python
def countfrom(n):
    while True:
        yield n
        n += 1

counter = countfrom(10)
next(counter) # 10
next(counter) # 11
```

There we have a simple Python generator that when called with `next`, will execute the function until the `yield` producing the value `n` that was provided to the function.  Then it stops.  The incrementing of `n` _does not_ occur until a successive call to `next`.  At which point it would increment and then follow the `while` back to the start of the loop and yield the incremented value.

## Stateful Solution

I think the simplest—and perhaps closest—representation of this would be with the [`state`] declarator:

```perl6
sub countfrom($n) {
    # state variables maintain their value each call to the sub
    state $counted = $n;
    return $counted++;
}

# xx means "times" i.e., it creates a list of RHS elements
# each containing the LHS.
say countfrom(10) xx 10; #={(10 11 12 13 14 15 16 17 18 19)}
```

This sub accurately reflects the simple generator above.  The [`state`] variable is maintained between calls and thus each successive call to `countfrom` is returning an updated value.  The argument to the routine isn't even used in subsequent calls.  To better illustrate this you could return an anonymous routine that would allow calling without providing a value:

```perl6
sub countfrom($n) {
    sub {
      state $counted = $n;
      return $counted++;
    }
}

my &counter = countfrom(10);
say counter() xx 10; #={(10 11 12 13 14 15 16 17 18 19)}

my &indices = countfrom(0);
say indices() xx 10; #={(0 1 2 3 4 5 6 7 8 9)}
```

At usage-level this is very close to the Python code however unlike in Python this function could not be dropped into a `for` loop.  But there are other structures that could utilize this and I'll cover one after I show a slightly more complicated generator function.

---

Let's assume we had a generator that was to calculate some value and return it, then calculate some more and return the new value, then execute some more and return a final value.  Here is the basic Python version:

```perl6
def simple_generator_function():
    # calculating...
    yield 1
    # calculating...
    yield 10
    # calculating...
    yield 100
```

One possible solution here would be to separate the calculations and "yields" into separate code blocks inside a state array and just return a shifted and called block:

```perl6
sub simple {
    state @yields = [
        { #`{ Calculating... } 1 }
        { #`{ Calculating... } 2 }
        { #`{ Calculating... } 3 }
    ];
    try @yields.shift.()
}

say simple(); #-> 1
say simple(); #-> 2
say simple(); #-> 3
say simple(); #-> Nil
```

That isn't so bad. The unfortunate part is that you have to introduce separate lexical scopes for each part of the process so you'd have to carefully pull out any variables that are needed across each block and declare them in the outermost routine scope. Now let me show you how you can use this in a looping construct:

```perl6
my $i = simple();
repeat {
    say $i * 100
} while $i = simple()
```

Once the value reaches `Nil` the block no longer executes.

---

Lets nitpick a couple things about the [`state`] approach.  For one, you are now introducing state into a routine.  State already has a bad enough reputation, now we're gonna put some inside a routine?! That's nuts!  I thought part of the point of generators was to avoid extra state. (Brief counterpoint: though we definitely should do our best to limit scope, careful (re: minimal or isolated) usage of these `state` variables can be a boon to your workflow and your codes readability.)

Another nit is that this solution can quickly become complex.  Each unpaused part of the routine needs to be a separate closure and pretty soon you might have all sorts of `my` or [`state`] declarations at the beginning of the outermost routine body to account for separated scopes. (Brief counterpoint: more explicit scoping is not always bad; complex Python generators can quickly become a mess and with this pattern you would be required to rein it in.)

## Gather, Take, Repeat

Perhaps the closest match for generators is really the list-building [`gather`/`take`] duo.  The `gather` keyword—in some ways similar to the `function*` keyword in JavaScript—informs the compiler that "within these walls, `take` returns a value to be pushed onto an array".  A bit like laying the guts of the generator out wherein the block is the building grounds for the array's elements.

```perl6
my @values = gather {
    # calculating...
    take 1;
    # calculating...
    take 10;
    # calculating...
    take 100
}
say @values #={[1 10 100]}
```
In fact, "within these walls" is a misnomer because the gather works *dynamically* which allows you to place calls to `take` in routines called within the `gather`'s scope.

```perl6
sub take1   { take 1 }
sub take10  { take 10 }
sub take100 { take 100 }
my @values = gather {
  take1()
  take10()
  take100()
}
say @values #={[1 10 100]}
```

---

Perl 6 has three classes for array-like structures: [Seq] is anything that can lazily produce a *sequence* of values (while not actually keeping them after production). [List] is a sequence of values that sticks around. [Array] is a sequence of mutable containers that stick around. `gather` returns a [Seq] but you can convert between the three, being careful not to try to eagerly collect an infinite list.

The resultant structure ([Seq]) is inherently lazy. As long as you remain in the [Seq] domain, your generator-like will lazily execute, but if you try to turn it into an [Array], Perl 6 will eagerly collect all values.  In the example above, `@values` was eagerly collected as the assignment to the container turns it into an [Array]. Here's an example of a gather block that stays as a [Seq] and thus is lazily executed:

```perl6
(gather {
    for 'a', 'b' ... * {
        'before'.say;
        take $_;
        'after'.say;
    }
})[0..1].say
#={before}
#={after}
#={before}
#={(a b)}
```

If you want to use `shift` as you would `next` in Python, you'll need to use that `lazy` keyword to make sure that the generated [Array] does not eagerly collect all values:

```perl6
(lazy gather {
    for <a b c d e f g> {
        'before'.say;
        take ($_.ord.Int + 1).chr
        'after'.say;
    }
}).Array.shift.say;
#={before}
#={b}
```

---

The [`gather`/`take`] construct is perhaps the closest to Python/JavaScript generators that Perl 6 has but there are a few other use-cases for generators that Perl 6 has some great tools for.

## Laudable Lazy "Lists"

In Perl 6 you can easily write lazy lists (really they are lazy [Seq] values).  First of all, the basic [Range] is lazy by default (but also will be eagerly collected if you convert it to an [Array]):

```perl6
say (1..1_000_000);             #={1..1000000}
say (1..1_000_000)[5..10];      #={(6 7 8 9 10 11)}
say (100..Inf)[0..5];           #={(100 101 102 103 104 105)}
say (100..Inf).Array.[0..5]     #={hangs indefinitely}
say (lazy 100..Inf).Array.shift #={100}
```

Another way to generate lazy [Seq] values is with the sequence operator ([`...`]). It creates a [Seq] just like the range operator but it also allows you to specify a pattern:

```perl6
(1, 3... *)[^10].say #={(1 3 5 7 9 11 13 15 17 19)}
(10, 20 ... *)[5].say #={50}
(100, 200 ... 1000)[*-2..*-1].say #={(900 1000)}
(1, 2 ... *).Array.[5].say #={hangs indefinitely}
(lazy 1, 2 ... *).Array.[5].say #={6}
```
So you see, in some instances that you might use a generator in another language, Perl 6 would allow you to quite simply create expressions via nifty operators like [`..`] or [`...`] and there are many more!

---

There's one final set of tools that (though also available to Python and JavaScript programmers via `__next__` in Python and `Symbol.iterator` in JavaScript) is useful when you would consider using the generator pattern: the [Iterator] and [Iterable] roles.

## Iterator: Rise of the Classes

Like Python and Java, Perl 6 has the distinction between an [Iterator] and an [Iterable]. An [Iterator] is the actual sequence of values that is to be iterated over.  An Iterable is anything that can produce an [Iterator].  The reason for this distinction is that the [Iterator] contains the state that is shifted as it works through the available values and the [Iterable] is the originating collection that has no need to contain the state corresponding to the process of iteration.  It's a distinction of "roles".  The data in an Iterable-roled class is meant for collection while the data in an Iterator-roled class is meant for iteration. In fact, this corroborates that same distinction between a generator *function* (here an [Iterable]) and a generator *iterator* (here an [Iterator])

The [Iterable] role requires that a class provides an `iterator` method which returns some value implementing the [Iterator] role (equivalent to Python's `__iter__`). The [Iterator] role requires that a class provides a `pull-one` method (equivalent to Python's `__next__`).

```perl6
Seq.new(class :: does Iterator {
    my @values = [1, 10, 100];
    method pull-one { @values.shift }
}.new)[0..2].say; #={[1 10 100]}

for class :: does Iterable {
    method iterator {
        (gather {
            take 1;
            take 10;
            take 100;
        }).iterator
    }
}.new {
    "$_ ".print
} #={1 10 100 }
```

To nitpick on this pattern, properly utilizing this requires being fully invested in an object-oriented model for your program. In other cases, using [`gather`/`take`] or the stateful approach will prove simpler and equally expressive. It also helps that [Seq] gives you the `iterator` method which makes using them in `for` loops easy.

---

While generators or semi-coroutines are not directly available in the same form as Python or JavaScript, there are still many available paths you can take in Perl 6 to accomplish the same results.  If you find another interesting generator pattern, please provide the details in a tweet or on IRC!

[Iterable]: http://doc.perl6.org/type/Iterable
[Iterator]: http://doc.perl6.org/type/Iterator
[`state`]: http://doc.perl6.org/syntax/state
[List]: http://doc.perl6.org/type/List
[Seq]: http://doc.perl6.org/type/Seq
[Array]: http://doc.perl6.org/type/Array
[Range]: http://doc.perl6.org/type/Range
[`..`]: http://doc.perl6.org/language/operators#infix_..
[`...`]: http://doc.perl6.org/language/operators#infix_...
[`gather`/`take`]: http://doc.perl6.org/syntax/gather%20take
