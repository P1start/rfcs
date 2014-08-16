- Start Date: 2014-08-16
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)

Summary
=======

Extend the iterator protocol and `for` loops to allow for passing values *to*
iterators.

Motivation
==========

Right now, iterators are very simple—all they can do is advance (`.next`), and
stop if there’s nothing more to iterate over. However, iteration often isn’t so
simple—parsers and tokenisers, for instance, often skip over multiple
characters at once. The main cause of this problem is the lack of an ability to
*input* into an iterator.

This is a very real problem—quite often, one wishes to choose the direction of
iteration *while* iterating, or choose how many values to iterate over at once.
The general solution to doing such things is to stop using iterators altogether
and instead fall back on old-fashioned loops and unwieldy methods like
`StrSlice::char_range_at_reverse`. This is certainly no good at all, as
iterators were designed entirely for the purpose of facilitating iterating
through a set of values, which is not necessarily one-dimensional and
one-directional.

Coroutines are another problem. Coroutines, or generators, are a
commonly-requested feature for Rust. However, coroutines are often given the
option of receiving values *while* iterating, and then processing those values
for the next iteration. Sound familiar? This problem would be solved by allowing
inputs to iterators.

One could easily construct a list of some of the uses of iterator inputs:

* Coroutines. See above.

* Bidirectional iteration. This is certainly a useful feature, and, as stated
  above, is quite difficult and confusing to use today.

* Skipping over or gathering multiple values at once. This could be quite
  useful for parsers and suchlike.

* Traversing a multi-dimensional surface in a non-linear fashion. This would
  possibly be useful for anything that moves across a plane, such as a
  maze-solving algorithm or even the main loop of a tile-based game.

* Anything that can be iterated over but requires one value per iteration. This
  could be something as obscure as a game loop, sending a vector of drawables
  to the iterator.

However, as with any change, the only true way to discover all the uses of
something is to see what people do with it.

Detailed design
===============

Extend the `Iterator` trait to take *two* type parameters, and extend the
`next` method to take an input argument:

```rust
pub trait Iterator<A, I> {
    fn next(&mut self, I) -> Option<A>;

    // ...provided methods...
}
```

Change all existing iterators to implement `Iterator<A, ()>`.

Extend the syntax for `continue` to permit an expression following it (after
the loop label, if any). The expression in the `continue` statement would be
passed to the iterator’s `next` method.

Extend `for` loops to permit a final expression without a semicolon. This is
equivalent to adding a final `continue` statement with the expression as its
argument. (That is, `for ... { stmts; expr }` would be equivalent to `for ... {
stmts; continue expr; }`.)

Extend the `for` loop syntax to allow for a default value clause. This is an
optional expression that is placed after a new keyword, ‘with’, after the
`in` part of the iterator. This is the value that will be passed to the
iterator’s `next` method if:

1. it is the first iteration of the loop;
2. a `continue` statement without an expression was reached; or
3. the  end of the `for` loop’s body is reached and it has no final expression
   without a semicolon.

Example
-------

The following `for` loop:

```rust
for pattern in expression with default {
    code;
    if blah { continue; }
    if cond { continue expr; }
    res
}
```

would be roughly equivalent to something like this:

```rust
match &mut expression {
    i => {
        let mut _continue = default;
        loop {
            match i.next(_continue) {
                None => break,
                Some(mut _value) => {
                    let pattern = _value;
                    {
                        code;
                        if blah { continue; }
                        if cond { _continue = expr; continue; }
                        _continue = res;
                    }
                }
            }
        }
    }
}
```

Additions to the standard library
---------------------------------

*Note: the following section is not properly part of the RFC. It is merely a
suggestion for how the standard library could be extended with the addition of
iterator inputs.*

A number of new structs (and corresponding methods on `Iterator`) and traits
could be added to the standard library. This could include:

* A new `BidirectionalIterator` trait. Vectors, slices, strings etc. would all
  implement this trait. This trait would have a method which could return a new
  struct which would implement `Iterator<A, Direction>`. `Direction` would be a
  new enum, `enum Direction { Advance, Reverse }`. This would mean that
  something like the following code could work:

  ```rust
  for i in [1i, 2, 3].iter().bidir() with Advance {
      println!("{}", i);
      if i == 2 {
          continue Reverse;
      }
  }
  ```

* A new `SkipN` struct. This would implement `Iterator<A, int>`. This could
  make this code valid:

  ```rust
  for c in "hello! World".chars().skipn() with 1 {
      if c == '!' {
          continue 2;
      }
      println!("{}", c);
  }
  ```

Drawbacks
=========

* This is not a backwards-compatible change, and would require quite a few
  changes to all of the `Iterator` implementations in the standard library.
  However, this is a change that would presumably not be particularly difficult
  to implement in the compiler itself, due to how `for` loops are just sugar
  for other statements.

* This change could be considered unnecessary. We don’t have coroutines at the
  moment, and possibly never will, making part of the motivation for this
  change completely redundant.

Alternatives
============

* Do nothing. We keep our current iterator protocol, and have no easy way to
  input anything into an iterator.

* Give the `Iterator` trait a default type parameter, `()` and also separate
  the `next` method into two methods: `next`, which behaves exactly as it does
  right now, and another method which takes an input parameter and behaves as
  the `next` method does in this RFC. This would perhaps make the change fully
  backwards-compatible.

Unresolved questions
====================

* The precise syntax of the ‘default’ clause in `for` loops is rather
  questionable—what should it be?

* Should the `Iterator` trait’s input parameter have a default value, `()`?
  This would make the transition to the new `Iterator` trait slightly easier.
