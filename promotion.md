# Const promotion

"Promotion" is the act of guaranteeing that code not written in a const context
(e.g. initalizer of a `const` or `static`, or an array length expression) will
be run at compile-time.

## Promotion contexts

There are a few different contexts where promotion is beneficial.

### Lifetime extension

"Lifetime extension" is a mechanism that affects code like `&3`:
Instead of putting it on the stack, the `3` is allocated in global static memory
and a reference with lifetime `'static` is provided.  This is essentially an
automatic transformation turning `&EXPR` into
`{ const _PROMOTED = &EXPR; EXPR}`, but only if `EXPR` qualifies.

Note that promotion happens on the MIR, not on surface-level syntax.  This is
relevant when discussing e.g. handling of panics caused by overflowing
arithmetic.

Lifetime extension is described in [RFC 1414][promotion-rfc]. The RFC uses the
word "promotion" to refer exclusively to lifetime extension, since this was the
first context where promotion was done.

[promotion-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1414-rvalue_static_promotion.md

### Non-`Copy` array initialization

Another promotion context, the initializer of an array expression, was
introduced in [RFC 2203][]. Here, promotion allows arrays of
non-`Copy` types to be initialized idiomatically, for example
`[Option::<Box<i32>>::None; 32]`.

[RFC 2203]: https://github.com/rust-lang/rfcs/blob/master/text/2203-const-repeat-expr.md

### `#[rustc_args_required_const(...)]`

Additionally, some platform intrinsics require certain parameters to be
immediates (known at compile-time). We use the `#[rustc_args_required_const]`
attribute, introduced in
[rust-lang/rust#48018](https://github.com/rust-lang/rust/pull/48018), to
specify these parameters and (aggressively, see below) try to promote the
corresponding arguments.

### Implicit and explicit contexts

On top of what applies to [consts](const.md), promoteds suffer from the additional issue that *the user did not ask for them to be evaluated at compile-time*.
Thus, if CTFE fails but the code would have worked fine at run-time, we broke the user's code for no good reason.
Even if we are sure we found an error in the user's code, we are only allowed to [emit a warning, not a hard error][warn-rfc].
That's why we have to be very conservative with what can and cannot be promoted.

For example, users might be surprised to learn that whenever they take a
reference to a temporary, that temporary may be promoted away and never
actually put on the stack. In this way, lifetime extension is an "implicit
promotion context": the user did not ask for the value to be promoted.

On the other hand, when a user passes an expression to a function with
`#[rustc_args_required_const]`, they are explicitly asking for that expression
to be evaluated at compile-time even though they have not written it in a
`const` declaration. We call this an "explicit promotion context".

Currently, non-`Copy` array initialization is treated as an implicit context.

The distinction between these two determines whether calls to arbitrary `const
fn`s (those without `#[rustc_promotable]`) are promotable (see below). See
[rust-rfcs/const-eval#19](https://github.com/rust-rfcs/const-eval/issues/19)
for a thorough discussion of this. At present, this is the only difference
between implicit and explicit contexts. The requirements for promotion in an
implicit context are a superset of the ones in an explicit context.

[warn-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1229-compile-time-asserts.md

### Promotion contexts in `const` and `static`

We defined above that promotion guarantees that code in a non-const context
will be executed at compile-time. However, lifetime extension and non-`Copy`
array initialization are useful features *inside* `const`s and `static`s as
well. Strictly speaking, the transformation used to enable these features
inside a const-context is not promotion; no `promoted`s are created in the MIR.
However the same rules for promotability are used with one modification:
Because the user has already requested that this code run at compile time, all
contexts are treated as explicit.

## Promotability

We have described the circumstances where promotion is desirable, but what
expressions are actually eligible for promotion? We refer to eligible
expressions as "promotable" and describe the restrictions on such expressions
below.

### Named locals

Promotable expressions cannot refer to named locals. This is not a technical
limitation with the CTFE engine. While writing `let x = {expr}` outside of a
const context, the user likely expects that `x` will live on the stack and be
initialized at run-time.  Although this is not (to my knowledge) guaranteed by
the language, we do not wish to violate the user's expectations here.

### Single assignment

We only promote temporaries that are assigned to exactly once. For example, the
lifetime of the temporary whose reference is assigned to `x` below will not be
extended.

```rust
let x: &'static i32 = &if cfg!(windows) { 0 } else { 1 };
```

Once again, this is not a fundamental limitation in the CTFE engine; we are
perfectly capable of evaluating such expressions at compile time. However,
determining the promotability of complex expressions would require more
resources for little benefit.

### Access to a `const` or `static`

Accesses to `const`s are always promotable, regardless of the body of the
`const`. For instance, while the previous example was not legal, the
following would be:

```rust
const NOT_WINDOWS: i32 = if cfg!(windows) { 0 } else { 1 };
let x: &'static i32 = &NOT_WINDOWS;
```

However, an access to a `static` is only promotable within the initializer of
another `static`.

### Panics

Promotion is not allowed to throw away side effects.  This includes panicking.
Let us look at what happens when we promote `&(0_usize - 1)` in a debug build:
We have to avoid erroring at compile-time, because that would be promotion
breaking compilation, but we must be sure to error correctly at run-time.  In
the MIR, this looks roughly like

```
_tmp1 = CheckedSub (const 0usize) (const 1usize)
assert(!_tmp1.1) -> [success: bb2; unwind: ..]

bb2:
_tmp2 = tmp1.0
_res = &_tmp2
```

Both `_tmp1` and `_tmp2` are promoted.  `_tmp1` evaluates to `(~0, true)`, so
the assertion will always fail at run-time.  Computing `_tmp2` fails with a
panic, which is thrown away -- so we have no result.  In principle, we could
generate any code for this because we know the code is unreachable (the
assertion is going to fail).  Just to be safe, we generate a call to
`llvm.trap`.

As long as CTFE only panics when run-time code would also have panicked, this
works out correctly: The MIR already contains provisions for what to do on
panics (unwind edges etc.), so when CTFE panics we can generate code that
hard-codes a panic to happen at run-time.  In other words, *promotion relies on
CTFE correctly implementing both normal program behavior and panics*.  An
earlier version of miri used to panic on arithmetic overflow even in release
mode.  This breaks promotion, because now promoting code that would work (and
could not panic!) at run-time leads to a compile-time CTFE error.

*Dynamic check.* The Miri engine already dynamically detects panics, but the
main point of promoteds is ruling them out statically.

### Const safety

We have explained what happens when evaluating a promoted panics, but what about
other kinds of failure -- what about hitting an unsupported operation or
undefined behavior?  To make sure this does not happen, only const safe code
gets promoted. The exact details for `const safety` are discussed in
[here](const_safety.md).

An example of this would be `&(&1 as *const i32 as usize % 16 == 0)`. The actual
location is not known at compile-time, so we cannot promote this.  Generally, we
can guarantee const-safety by not promoting when an unsafe or unconst operation
is performed -- if our const safety checker is correct, that has to cover
everything, so the only possible remaining failure are panics.

However, things get more tricky when `const` and `const fn` are involved.

For `const`, based on the const safety check described [here](const_safety.md#const-safety-check-on-values),
we can rely on there not being const-unsafe values in the `const`, so we should
be able to promote freely.  For example:

```rust
union Foo { x: &'static i32, y: usize }
const A: usize = unsafe { Foo { x: &1 }.y };
const B: usize = unsafe { Foo { x: &2 }.y };
let x: &bool = &(A < B);
```

Promoting `x` would lead to a compile failure because we cannot compare pointer
addresses.  However, we do not even get there -- computing `A` or `B` fails with
a const safety check error because these are values of type `usize` that contain
a `Scalar::Ptr`.

For `const fn`, however, there is no way to check anything in advance.  We can
either just not promote, or we can move responsibility to the `const fn` and
promote *if* all function arguments pass the const safety check.  So,
`foo(42usize)` would get promoted, but `foo(&1 as *const i32 as usize)` would
not.  When this call panics, compilation proceeds and we just hard-code a panic
to happen as well at run-time.  However, when const evaluation fails with
another error (unsupported operation or undefined behavior), we have no choice
but to abort compilation of a program that would have compiled fine if we would
not have decided to promote.  It is the responsibility of `foo` to not fail this
way when working with const-safe arguments.

For this reason, only `const fn` that were explicitly marked with the
`#[rustc_promotable]` attribute are subject to promotion.

*Dynamic check.* The Miri engine already dynamically detects const safety
violations, but the main point of promoteds is ruling them out statically.

### Drop

Expressions returning "needs drop" types can never be promoted. If such an
expression were promoted, the `Drop` impl would never get called on the value,
even though the user did not explicitly request such behavior by using an
explicit `const` or `static` item.

As expression promotion is essentially the silent insertion of a `static` item, and
`static` items never have their `Drop` impl called, the `Drop` impl of the promoted
value would never get called.

While it is sound to `std::mem::forget` any value and thus not call its `Drop` impl,
it is unlikely to be the desired behavior in most cases and very likey to be confusing
to the user. If such behavior is desired, the user can still use an explicit `static`
or `const` item and refer to that.

*Dynamic check.* The Miri engine could dynamically check this by ensuring that
the result of computing a promoted is a value that does not need dropping.

## `&` in `const` and `static`

Promotion is also responsible for making code like this work:

```rust
const FOO: &'static i32 = {
    let x = &13;
    x
};
```

However, since this is in explicit const context, we are less strict about
promotion in this situation: all function calls are promoted, not just
`#[rustc_promotable]` functions:

```rust
const fn bar() -> i32 { 42 }

const FOO: &'static i32 = {
    let x = &bar(); // this gets promoted
    x
};
```

However, we still do not promote *everything*; e.g., drop-checking still applies:

```rust
const DROP: &'static Vec<u8> = {
    let x = &Vec::new(); //~ ERROR: temporary value dropped while borrowed
    x
};
```

Notice that some code involving `&` *looks* like it relies on promotion but
actually does not:

```rust
const EMPTY_BYTES: &Vec<u8> = &Vec::new(); // Ok without promotion
```

As we have seen above, `Vec::new()` does not get promoted. And yet this
compiles. Why that? The reason is that the reference obtains the lifetime of
the "enclosing scope", similar to how `let x = &mut x;` creates a reference
whose lifetime lasts for the enclosing scope. This is decided during MIR
building already, and does not involve promotion.

## Open questions

* There is a fourth kind of CTFE failure -- resource exhaustion.  What do we do
  when that happens while evaluating a promoted?
