+++
title = "Basic Types"
weight = 50
template = "doc.html"
aliases = ["docs/reference/hoon-expressions/basic/"]
+++
A type is usually understood to be a set of values.  Hoon values are all nouns,
so a Hoon type is a set of nouns.

Hoon's type system conducts various type-checks at compile time in order to
ensure type safety.  For example, one's program might have an integer squaring
function, such that given some atom `n` the return value is `n^2`.  The output
value should be an unsigned integer (i.e., an atom).  Hoon uses type inference
on the expression that defines the squaring function in to determine what
possible values it could produce.  If the inferred type 'nests' under the
desired type then the program compiles; otherwise the compile fails with a
`nest-fail` crash.

(For an introduction on how to use Hoon's type system, see Chapter 2 of the Hoon
tutorial.)

In this document we discuss the recursive data structure Hoon uses for type
inference.  Because the Hoon compiler is written in Hoon this structure is
likewise defined in Hoon.

## `$type`

Below is a simplified version the `+$  type` arm, which defines the data
structure Hoon uses to keep track of types.  But this data structure is used for
more than simply type inference.  It also handles the resolution of names,
including both faces and arm names.  (See Chapter 1 of the Hoon tutorial for an
introduction to name resolution.)

As noted, this is a simplified version of `type`.  We undo and
explain the simplifications in the [advanced types](/docs/hoon/reference/advanced)
section.

```hoon
+$  term  @tas
+$  type  $~  %noun
          $@  $?  %noun
                  %void
              ==
          $%  [%atom p=term q=(unit @)]
              [%cell p=type q=type]
              [%core p=type q=(map term hoon)]
              [%face p=term q=type]
              [%fork p=(set type)]
              [%hold p=type q=hoon]
          ==
```

Effectively this structure defines a set of nouns that Hoon uses to keep track
track of (1) type information, and (2) name resolution information.  This set of
nouns is recursively defined.

### `?(%noun %void)`

`%noun` is the label for a noun about which nothing else is known.  Every piece
of Hoon data is a noun, so everything nests under `%noun`.

`%void` is the label for the empty set.

### `[%cell p=type q=type]`

`[%cell p=type q=type]` is for cells.  The `type` of the head is labelled `p`
and the tail `type` is `q`.

### `[%fork p=(set type)]`

`[%fork p=(set type)]` is for the union of all types in the set `p`.  That is,
when the type of an expression is known to be a `%fork`, the expression must
evaluate as one of the types in `p`.

Hoon uses branching (usually with conditional runes of the `?` family) for type
inference in order to rule out which of the types in `p` can apply.

### `[%hold p=type q=hoon]`

A `%hold` type, with type `p` and hoon `q`, is a lazy reference to the type of
`(mint p q)`.  In English, it means: "the type of the product when we compile
Hoon expression `q` against a subject with type `p`."

### `[%face p=term q=type]`

A `[%face p=term q=type]` wraps the label `p` around the type `q`.  `p` is a
`term` or `@tas`, an atomic ASCII string which obeys symbol rules: lowercase and
digit only, infix hyphen, first character must be a lowercase letter.

### `[%atom p=term q=(unit atom))]`

`%atom` is for an atom, with two twists.  `q` is a `unit`, Hoon's
equivalent of a nullable pointer or a Haskell `Maybe`.  If `q`
is `~`, null, the type is **warm**; any atom is in the type.
If `q` is `[~ x]`, where `x` is any atom, the type is **cold**;
its only legal value is the constant `x`.

`p` in the atom is a terminal used as an **aura**, or soft atom
type.  Auras are a lightweight, advisory representation of the
units, semantics, and/or syntax of an atom.  An aura is an atomic
string; two auras are compatible if one is a prefix of the other.

For instance, `@t` means UTF-8 text (LSB low), `@ta` means ASCII
text, and `@tas` means an ASCII symbol.  `@u` means an unsigned
integer, `@ud` an unsigned decimal, `@ux` an unsigned
hexadecimal.  You can use a `@ud` atom as a `@u` or vice versa,
but not as a `@tas`.

Auras can also end with an optional, capitalized suffix, which
defines the atom's bitwidth as a log starting from `A`.  For
example, `@udD` is an unsigned decimal byte; `@uxG` is an
unsigned 64-bit hexadecimal.

You can make up your own auras and are encouraged to do so, but
here are some conventions bound to constant syntax:

```
@c              UTF-32 codepoint
@d              date
  @da           absolute date
  @dr           relative date (ie, timespan)
@n              nil
@p              phonemic base (plot)
@r              IEEE floating-point
  @rd           double precision  (64 bits)
  @rh           half precision (16 bits)
  @rq           quad precision (128 bits)
  @rs           single precision (32 bits)
@s              signed integer, sign bit low
  @sb           signed binary
  @sd           signed decimal
  @sv           signed base32
  @sw           signed base64
  @sx           signed hexadecimal
@t              UTF-8 text (cord)
  @ta           ASCII text (knot)
    @tas        ASCII text symbol (term)
@u              unsigned integer
  @ub           unsigned binary
  @ud           unsigned decimal
  @uv           unsigned base32
  @uw           unsigned base64
  @ux           unsigned hexadecimal
```

Auras are truly soft; you can turn any aura into any other,
statically, by casting through the empty aura `@`.  Hoon is not
dependently typed and can't statically enforce data constraints
(for example, it can't enforce that a `@tas` is really a symbol).

### `[%core p=type q=(map term hoon)]`

`%core` is for a code-data cell.  The data (or **payload**) is the
tail; the code (or **battery**) is the head.  `p`, a type, is the
type of the payload.  `q` is a mapping of arm names and Hoon expressions.  It is the source code for the core battery.

(For an introduction to cores and arms, see Chapter 1 of the Hoon tutorial.)

Each expression in the battery source is compiled to a formula, with
the core itself as the subject.  The battery is a tree of these
formulas, or **arms**.  An arm is a computed attribute against its
core.

All code-data structures in normal languages (functions, objects,
modules, etc) become cores in Hoon.  A Hoon battery looks a bit
like a method table, but not every arm is a "method" in the OO
sense.  An arm is a computed attribute.  A method is an arm whose
product is a Hoon function (or **gate**).

A gate (function, lambda, etc) is a core with one arm, whose name
is the empty symbol `$`, and a payload whose shape is `[sample
context]`.  The **context** is the subject in which the gate was
defined; the **sample** is the argument.

To call this function on an argument `x`, replace the sample (at
tree address `6` in the core) with `x`, then compute the arm.
(Of course, we don't mutate the noun, we make a mutant copy.)
