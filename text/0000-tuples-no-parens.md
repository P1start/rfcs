- Feature Name: `unparenthesized_tuples`
- Start Date: 2016-05-01
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow tuple expressions, patterns, and types to be written without parentheses
in most unambiguous contexts.

# Motivation
[motivation]: #motivation

Tuples are used in many places throughout Rust programs. However, in many
contexts (especially multiple assignment), their syntax seems needlessly bulky.
The Python programming language solves this problem by allowing the parentheses
to be omitted in many contexts. This allows for a much lighter-looking multiple
assignment syntax (`a, b = 1, 2`) among other uses of this feature.

One common use of tuples is in indexing of multi-dimensional array types. For
example, [`nalgebra`][nalgebra]’s matrix types implement
`Index<(usize, usize)>`. However, due to Rust’s tuple syntax, such indexing must
be written as `mat[(x, y)]`, which is quite inconvenient. Allowing tuples to be
unparenthesised would make this syntax look a fair bit nicer.

[nalgebra]: http://nalgebra.org/

# Detailed design
[design]: #detailed-design

- Change the syntax of tuples in expressions, patterns, and types to allow `a,
  b, c, ...` as well as the current `(a, b, c, ...)`.

- Unparenthesised tuple expressions have a precedence lower than every operator
  but the assignment operator and closure expressions. This is primarily for
  future-compatibility with multiple assignment (i.e., `a, b = 1, 2`) if the
  language ever gets such a feature.

- In the parser, define a new type of restricted expression (similar to the
  no-struct-literal expression type defined in [RFC 92][rfc92]) to be used in
  contexts where unparenthesised tuple expressions would be ambiguous. This
  restricted expression type would not allow unparenthesised tuple expressions.
  Specifically, the following expression contexts would be changed to be
  this new type of restricted expression:

  - function call parameters, including tuple struct initialiser fields;
  - struct initialiser fields;
  - all array expression elements;
  - the contents of parenthesised expressions;
  - macro `expr` parameters;
  - either side of the assignment operator when the assignment expression is in
    an unparenthesised tuple expression–disallowing context;
  - the return value expression of a closure when the closure expression is in
    an unparenthesised tuple expression–disallowing context.

  And the following type and pattern contexts would not permit unparenthesised
  tuple types or patterns:

  - (tuple) struct definition fields;
  - enum variant definition fields;
  - function and closure definition arguments, both types and patterns;
  - type parameters;
  - (tuple) struct pattern fields;
  - enum pattern fields;
  - array pattern elements;
  - array element types;
  - parenthesised types;
  - macro `pat` and `ty` parameters.

  In these contexts, a comma signifies something other than a tuple, and so
  would be interpreted differently.

  Zero-length tuples (`()`) are a special case: these can only be written as
  `()`; i.e., their parentheses may not be omitted.

- Define compatibility with the grammar changes introduced in [RFC 92][rfc92]
  by disallowing struct literals in tuple expressions if the tuple expression
  itself is in a context where struct literals would not be allowed.

[rfc92]: https://github.com/rust-lang/rfcs/blob/master/text/0092-struct-grammar.md

## Examples

### Valid

```rust
// Tuple patterns and expressions without parentheses:
let x, y = 1, 2;

// Tuple types can also be written without parentheses:
let x, y: i32, f64 = 47, 3.14;

// Tuples of length 1 can be written without parentheses:
let x,: i32, = 1,;

// Trailing commas are allowed:

let x: i32, i32, i32, =
    1,
    2,
    3,
;

// match can use unparenthesised tuple patterns and expressions:
match x, y {
    0...10, 11...20 => ...,
    a, b => ...,
}

// For loops can use unparenthesised tuple patterns:
for a, b in vec1.iter().zip(vec2.iter()) { ... }

// A for loop using unparenthesised tuple patterns in an array literal is still
// syntactically valid:
let x: [(); 1] = [for i, j in iter { ... }];

// Comma has lower precedence than most operators:
let x: bool, i32 = 1 * 2 + 3 / 4 == 0 || 3 - 4 % 2 > 72, 45 & 2 >> 3;

// Comma has lower precedence than -, so here it is interpreted as unary:
let x: i32, i32 = 1, -1;

// Closures have lower precedence than comma:
let x = |a, b| a, b;
// But in contexts that don’t allow unparenthesised tuple expressions (in
// this case, a parenthesised expression) it is interpreted differently:
let b: i32 = 1;
let x: _, i32 = (|a, b| a, b);

// Similarly for the assignment operator:
let mut x = 1, 2;
x = 3, 4;
// But:
let mut x = 1;
let _: ((), i32) = (x = 3, 4);

// An extra set of parentheses is required in struct definitions:
struct Foo((i32, f64));
struct Bar { x: (i32, f64) }
// ...and also in struct initialisers:
let foo = Foo((1, 2));
let bar = Bar { x: (1, 2) };
// ...and also in function calls:
let foo = frobnicate((baz, qux));
// ...and also in type parameters:
let foo: Foo<(i32, f64)> = get_foo();

// Parentheses can be omitted in return expressions and return type
// declarations:
fn foo() -> i32, i32 {
    return 1, 2
}

// Parentheses can be omitted when indexing values:
let x: i32 = mat[0, 1];
```

### Invalid

```rust
// Although technically unambiguous, commas in function argument patterns are
// disallowed for clarity and consistency with closures:
fn foo(x, y: (i32, i32)) { ... }

// Although unparenthesised tuples in array types are also unambiguous, for
// consistency with array literals this is disallowed:
let x: [i32, i64; 1] = [(1, 2)];

// Struct literals are disallowed in unparenthesised tuple patterns in match,
// if, and similar constructs:
match Baz { ... }, Qux { ... } {
    baz, qux => ...
}

if Baz { ... }, Qux { ... } == get_baz(), get_qux() {
    ...
}
```

# Drawbacks
[drawbacks]: #drawbacks

Tuples written without parentheses can be visually confusing in some contexts,
due to the other meanings of `,` in the language.

# Alternatives
[alternatives]: #alternatives

This RFC as written permits tuples in almost every unambiguous context. However,
a context that is unambiguous to the parser isn’t necessarily easy for a human
reader to understand. There is a lot of room for adjustment to the rules about
where parentheses can be omitted; in particular, Python forbids unparenthesised
tuples in `if` statements, while this RFC does not.

# Unresolved questions
[unresolved]: #unresolved-questions

Are there any other ambiguities regarding unparenthesised tuples that this RFC
fails to cover?
