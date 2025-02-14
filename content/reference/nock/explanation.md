+++
title = "Explanation"
weight = 2
+++

To make Nock make sense, let's work through the spec line by line.
First the data model:

## Nouns

```
An atom is any natural number.
A noun is an atom or a cell.
A cell is any ordered pair of nouns.
```

Nouns are the dumbest data model ever.  Nouns make JSON look like
XML and XML look like ASN.1.  It may also remind you of Lisp's
S-expressions - you can think of nouns as "S-expressions without
the S."

To be exact, a noun **is** an S-expression, except that classic
S-expressions have multiple atom types ("S" is for "symbol").
Since Nock is designed to be used with a higher-level type system
(such as Hoon's), it does not need low-level types.  An atom is
just an unsigned integer of any size.

For instance, it's common to represent strings (or even whole
text files) as atoms, arranging them LSB first - so "foo" becomes
`0x6f6f66`.  How do we know to print this as "foo", not `0x6f6f66`?
We need external information - such as a Hoon type.  Similarly,
other common atomic types - signed integers, floating point, etc
- are all straightforward to map into atoms.

It's also important to note that, unlike Lisp, Nock cannot create
cyclical data structures.  It is normal and common for nouns in a
Nock runtime system to have acyclic structure - shared subtrees.
But there is no Nock computation that can make a child point to
its parent.  One consequence: Nock has no garbage collector.
(Nor can dag structure be detected, as with Lisp `eq`.)

There is also no single syntax for nouns.  If you have nouns you
have Nock; if you have Nock you have Hoon; if you have Hoon, you
can write whatever parser you like.

## Reductions

It's important to recognize that the pseudocode of the Nock spec
is just that: pseudocode.  It looks a little like Hoon.  It isn't
Hoon - it's just pseudocode.  Or in other words, just English.
At the bottom of every formal system is a system of axioms, which
can only be written in English.  (Why pseudocode, not Hoon?  Since
Hoon is defined in Nock, this would only give a false impression
of nonexistent precision.)

The logic of this pseudocode is a pattern-matching reduction,
matching from the top down.  To compute Nock, repeatedly reduce
with the first line that matches.   Let's jump right in!

### Nock principles

```
nock(a)  *a
```

Nock is a pure (stateless) function from noun to noun.  In our
pseudocode (and only in our pseudocode) we express this with the
prefix operator `*`.

This function is defined for every noun, but on many nouns it
does nothing useful.  For instance, if `a` is an atom, `*a`
reduces to... `*a`.  In theory, this means that Nock spins
forever in an infinite loop.  In other words, Nock produces no
result - and in practice, your interpreter will stop.

(Another way to see this is that Nock has "crash-only" semantics.
There is no exception mechanism.  The only way to catch Nock
errors is to simulate Nock in a higher-level virtual Nock -
which, in fact, we do all the time.  A simulator (or a practical
low-level interpreter) can report, out of band, that Nock would
not terminate.  It cannot recognize all infinite loops, of
course, but it can catch the obvious ones - like `*42`.)

Normally `a` in `nock(a)` is a cell `[s f]`, or as we say

```
[subject formula]
```

Intuitively, the formula is your function and the subject is
its argument.  We call them something different because Hoon,
or any other high-level language built on Nock, will build its
own function calling convention which **does not** map directly
to `*[subject formula]`.

### Noun syntax

```
[a b c]  [a [b c]]
```

Ie, brackets (in our pseudocode, as in Hoon) associate to the
right.  For those with Lisp experience, it's important to note
that Nock and Hoon use tuples or "improper lists" much more
heavily than Lisp.  The list terminator, normally 0, is never
automatic.  So the Lisp list

```
(a b c)
```

becomes the Nock noun

```
[a b c 0]
```

which is equivalent to

```
[a [b [c 0]]]
```

Note that we can and do use unnecessary brackets anyway, for
emphasis.  Let's move on to the axiomatic functions:

### Trivial operators

```
?[a b]  0
?a      1
+[a b]  +[a b]
+a      1 + a
=[a a]  0
=[a b]  1
```

Here we define more pseudocode operators, which we'll use in
reductions further down.  So far we have four built-in functions:
`*` meaning Nock itself, `?` testing whether a noun is a cell or
an atom, `+` incrementing an atom, and `=` testing for equality.
Again, no rocket science here.

We should note that in Nock and Hoon, `0` (pronounced "yes") is
true, and `1` ("no") is false.  Why?  As in Unix, using zero for
success generalizes smoothly to multiple error codes.  And it's
strange for success not to equal truth.   Or at least, this is
our official excuse.

### Tree addressing

```
/[1 a]            a
/[2 a b]          a
/[3 a b]          b
/[(a + a) b]      /[2 /[a b]]
/[(a + a + 1) b]  /[3 /[a b]]
```

Slightly more interesting is our tree numbering.  Every noun is
of course a tree.  The `/` operator - pronounced "slot" - imposes
an address space on that tree, mapping every nonzero atom to a
tree position.

1 is the root.  The head of every node `n` is `2n`; the tail is
`2n+1`.  Thus a simple tree:

```
     1
  2      3
4   5  6   7
         14 15
```

If the value of every leaf is its tree address, this tree is

```
[[4 5] [6 14 15]]
```

and, for some examples of addressing:

```
/[1 [[4 5] [6 14 15]]]
```

is `[[4 5] [6 14 15]]`

```
/[2 [[4 5] [6 14 15]]]
```

is `[4 5]`

```
/[3 [[4 5] [6 14 15]]]
```

is `[6 14 15]`, and

```
/[7 [[4 5] [6 14 15]]]
```

is `[14 15]`

Hopefully this isn't terribly hard to follow.

### Editing a Noun, `#`

```
#[1 a b]            a
#[(a + a) b c]      #[a [b /[(a + a + 1) c]] c]
#[(a + a + 1) b c]  #[a [/[(a + a) c] b] c]
```

The `#` operator replaces part of a noun with another noun.  `#[x y z]` replaces address `x` of `z` with the noun `y`.

Take some noun, say, `[22 33 44 55]`.  Let's replace part of it with another noun, `[123 456]`.  We can replace the entire original noun as follows:

```
#[1 [123 456] [22 33 44 55]]
```

is

```
[123 456]
```

If we only want to replace the value at address `2`:

```
#[2 [123 456] [22 33 44 55]]
```

is

```
[[123 456] 33 44 55]
```

And we can replace address `3`:

```
#[3 [123 456] [22 33 44 55]]
```

is

```
[22 123 456]
```

## Instructions

These rules define Nock itself - ie, the `*` operator.

### `0`, slot

```
*[a 0 b]  /[b a]
```

`0` is Nock's tree-addressing instruction.  Let's try it out
from the Arvo command line.

Note that we're using Hoon syntax here.  Since we do not use Nock
from Hoon all that often (it's sort of like embedding assembly in
C), we've left it a little cumbersome.  In Hoon, instead of
writing `*[a 0 b]`, we write

```
.*(a [0 b])
```

So, to reuse our slot example, let's try the interpreter:

```
~zod:dojo> .*([[4 5] [6 14 15]] [0 7])
```

produces `[14 15]`.

### `1`, constant

```
*[a 1 b]  b
```

`1` is the constant instruction.  It produces its argument without
reference to the subject.  So

```
~zod:dojo> .*(42 [1 153 218])
```

yields `[153 218]`.

### `2`, evaluate

`2` is the Nock instruction:

```
*[a 2 b c]  *[*[a b] *[a c]]
```

If you can compute a subject and a
formula, you can evaluate them in the interpreter.  In most
fundamental languages, like Lisp, `eval` is a curiosity.  But
Nock has no `apply` - so all our work gets done with `2`.

Let's convert the previous example into a stupid use of `2`,
with constant subject and formula:

```
~zod:dojo> .*(77 [2 [1 42] [1 1 153 218]])
```

This gives the same `[153 218]`

### `3`, `4`, and `5`, axiomatic functions

```
*[a 3 b]  ?*[a b]
*[a 4 b]  +*[a b]
*[a 5 b c]  =[*[a b] *[a c]]
```

These are axiomatic functions:

For instance, if `x` is a formula that calculates some product,
`[4 x]` calculates that product plus one.  Hence:

```
~zod:dojo> .*(57 [0 1])
57
~zod:dojo> .*(57 [4 0 1])
58
```
Similarly,
```
~zod:dojo> .*([132 19] [0 3])
19
~zod:dojo> .*([132 19] [4 0 3])
20
```

### Distribution

```
*[a [b c] d]  [*[a b c] *[a d]]
```

Since Nock of an atom just crashes, the practical domain of the
Nock function is always a cell.  Conventionally, the head of this
cell is the "subject," the tail is the "formula," and the result
of Nocking it is the "product."  Basically, the subject is your
data and the formula is your code.

We could write this line less formally:

```
*[subject [formula-x formula-y]]
=>  [*[subject formula-x] *[subject formula-y]]
```

In other words, if you have two Nock formulas `x` and `y`, a
formula that computes the pair of them is just `[x y]`.  We can
recognize this because no atom is a valid formula, and
every formula that **does not** use distribution has an atomic head.

If you know Lisp, you can think of this feature as a sort of
"implicit cons."  Where in Lisp you would write `(cons x y)`,
in Nock you write `[x y]`.

For example,

```
~zod:dojo> .*(42 [4 0 1])
```

where `42` is the subject (data) and `[4 0 1]` is the formula
(code), happens to evaluate to `43`.  Whereas

```
~zod:dojo> .*(42 [3 0 1])
```

is `1`.  So if we evaluate

```
~zod:dojo> .*(42 [[4 0 1] [3 0 1]])
```

we get

```
[43 1]
```

Except for the crash defaults, we've actually
completed all the _essential_ aspects of Nock.  The instructions up
through 5 provide all necessary computational functionality.
Nock, though very simple, is actually much more complex than it
formally needs to be.

Operators 6 through 11 are sugar.  They exist because Nock is
not a toy, but a practical interpreter.  Let's see them all
together:

## Sugar

```
*[a 6 b c d]        *[a *[[c d] 0 *[[2 3] 0 *[a 4 4 b]]]]
*[a 7 b c]          *[*[a b] c]
*[a 8 b c]          *[[*[a b] a] c]
*[a 9 b c]          *[*[a c] 2 [0 1] 0 b]
*[a 10 [b c] d]     #[b *[a c] *[a d]]

*[a 11 [b c] d]     *[[*[a c] *[a d]] 0 3]
*[a 11 b c]         *[a c]
```

Let's try to figure out what these do, simplest first:

### `11`, hint

```
*[a 11 b c]  *[a c]
```

If `x` is an atom and `y` is a formula, the formula `[11 x y]`
appears to be equivalent to... `y`.  For instance:

```
.*([132 19] [11 37 [4 0 3]])
```

Why would we want to do this?  `11` is actually a hint instruction.
The `37` in this example is discarded information - it is not
used, formally, in the computation.  It may help the interpreter
compute the expression more efficiently, however.

Every Nock computes the same result - but not all at the same
speed.  What hints are supported?  What do they do?  Hints are a
higher-level convention which do not, and should not, appear in
the Nock spec.  Some are defined in Hoon.  Indeed, a naive Nock
interpreter not optimized for Hoon will run Hoon quite poorly.
When it gets the product, however, the product will be right.

There is another instruction `11` rule:

```
*[a 11 [b c] d]     *[[*[a c] *[a d]] 0 3]
```

This complex hint
throws away an arbitrary `b`, but computes the formula `c`
against the subject and... throws away the product.  This formula
is simply equivalent to `d`.  Of course, in practice the product
of `c` will be put to some sordid and useful use.  It could even
wind up as a side effect.

(Why do we even care that `c` is computed?  Because `c` could
crash.  A correct Nock cannot simply ignore it, and treat both
variants of `11` as equivalent.)

### `7`, compose

```
*[a 7 b c]          *[*[a b] c]
```

Suppose we have two formulas, `b` and `c`.  What is the formula
`[7 b c]`?  This example will show you:

```
~zod:dojo> .*(42 [7 [4 0 1] [4 0 1]])
44
```

Good old composition, on Nock formulas.  It's easy to see how
this is a variant of Nock `2`.  The data to evaluate is simply `b`,
and the formula is `c` quoted.

### `8`, extend

```
*[a 8 b c]          *[[*[a b] a] c]
```

`8` is `7`, except that the subject for `c` is not simply the
product of `b`, but the ordered pair of the product of `b` and
the original subject.  Hence:

```
.*(42 [8 [4 0 1] [0 1]])
[43 42]
```

```
~zod:dojo> .*(42 [8 [4 0 1] [4 0 3]])
43
```

Why would we want to do this?  Imagine a higher-level language
in which the programmer declares a variable.  This language is
likely to generate an `8`, because the variable is computed
against the present subject, and used in a calculation which
depends both on the original subject and the new variable.

### `9`, invoke

`9` is a calling convention:

```
*[a 9 b c]          *[*[a c] 2 [0 1] 0 b]
```

With `c`, we produce a noun which contains both code and data - a
**core**.  We use this core as the subject, and evaluate the
formula within it at slot `b`.

### `10`, replace at address

`10` is how Nock updates a tree by altering a particular slot.

```
*[a 10 [b c] d]      #[b *[a c] *[a d]]
```

This operation relies on the `#` hax operator, which has the form

```
#[mem-slot new-val target-tree]
```

This replaces the memory slot `mem-slot` in `target-tree` with `new-val`.

So all together, `10` means that we calculate `*[a c]` and `*[a d]`, then replace memory slot `b` in the latter with the result of the former.

### `6`, if-then-else

This looks much scarier than it is:

```
*[a 6 b c d]        *[a *[[c d] 0 *[[2 3] 0 *[a 4 4 b]]]]
```

If `b` evaluates to `0`, we produce `c`; if `b` evaluates to `1`,
we produce `d`; otherwise, we crash.

For instance:

```
~zod:dojo> .*(42 [6 [1 0] [4 0 1] [1 233]])
43
~zod:dojo> .*(42 [6 [1 1] [4 0 1] [1 233]])
233
```

In real life, of course, the Nock implementor knows that `6` is
"if" and implements it as such.  There is no practical sense in
reducing through this macro, or any of the others.  We could have
defined "if" as a built-in function, like increment - except that
we can write "if" as a macro.  If a funky macro.

We can actually simplify the semantics of `6`, at the expense of
breaking the system a little, by creating a macro that works as
"if" only if `b` is a proper boolean and produces `0` or `1`.
Perhaps we have a higher-level type system which checks this.

This simpler "if" would be:

```
*[a 6 b c d]    *[a [2 [0 1] [2 [1 c d] [[1 0] [4 4 b]]]]]
```

Or without so many unnecessary brackets:

```
*[a 6 b c d]    *[a 2 [0 1] 2 [1 c d] [1 0] [4 4 b]]
```

How does this work?  We've replaced `[6 b c d]` with the formula
`[2 [0 1] [2 [1 c d] [[1 0] [4 4 b]]]]`.  We see two uses of `2`,
our evaluation instruction - an outer and an inner.

Call the inner one `i`.  So we have `[2 [0 1] i]`.  Which means
that, to calculate our product, we use `[0 1]` - that is, the
original subject - as the subject; and the product of `i` as
the formula.

Okay, cool.  So `i` is `[2 [1 c d] [[1 0] [4 4 b]]]`.  We compute
Nock with subject `[1 c d]`, formula `[[1 0] [4 4 b]]`.

Obviously, `[1 c d]` produces just `[c d]` - that is, the ordered
pair of the "then" and "else" formulas.  `[[1 0] [4 4 b]]` is a
line 19 cell - its head is `[1 0]`, producing just `0`, its tail
`[4 4 b]`, producing... what?  Well, if `[4 b]` is `b` plus `1`,
`[4 4 b]` is `b` plus `2`.

We're assuming that `b` produces either `0` or `1`.  So `[4 4 b]`
yields either `2` or `3`.  `[[1 0] [4 4 b]]` is either `[0 2]` or
`[0 3]`.  Applied to the subject `[c d]`, this gives us either
`c` or `d` - the product of our inner evaluation `i`.  This is
applied to the original subject, and the result is "if."

But we need the full power of the funk, because if `b` produces,
say, `7`, all kinds of weirdness will result.  We'd really like
`6` to just crash if the test product is not a boolean.  How can
we accomplish this?  This is an excellent way to prove to
yourself that you understand Nock: figure out what the real `6`
does.  Or you could just agree that `6` is "if," and move on.

(It's worth noting that in practical, compiler-generated Nock, we
never do anything as funky as these `6` macro internals.  There's
no reason we couldn't build formulas at runtime, but we have no
reason to and we don't - except when actually metaprogramming.
As in most languages, normally code is code and data is data.)
