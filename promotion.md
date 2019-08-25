# Const promotion

["(Implicit) Promotion"][promotion-rfc] is a mechanism that affects code like `&3`:
Instead of putting it on the stack, the `3` is allocated in global static memory
and a reference with lifetime `'static` is provided.  This is essentially an
automatic transformation turning `&EXPR` into
`{ const _PROMOTED = &EXPR; EXPR}`, but only if `EXPR` qualifies.

Note that promotion happens on the MIR, not on surface-level syntax.  This is
relevant when discussing e.g. handling of panics caused by overflowing
arithmetic.

On top of what applies to [consts](const.md), promoteds suffer from the additional issue that *the user did not ask for them to be evaluated at compile-time*.
Thus, if CTFE fails but the code would have worked fine at run-time, we broke the user's code for no good reason.
Even if we are sure we found an error in the user's code, we are only allowed to [emit a warning, not a hard error][warn-rfc].
That's why we have to be very conservative with what can and cannot be promoted.

[promotion-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1414-rvalue_static_promotion.md
[warn-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1229-compile-time-asserts.md

## Rules

### 1. Panics

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

### 2. Const safety

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

For `const`, based on the const safety check described [here](const_safety.md),
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

*Dynamic check.* The Miri engine already dynamically detects const safety
violations, but the main point of promoteds is ruling them out statically.

### 3. Drop

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

However, since this is in explicit const context, we could be less strict about
promotion in this situation.

Promotion is *not* involved in something like this:

```rust
#![feature(const_vec_new)]
const EMPTY_BYTES: &Vec<u8> = &Vec::new();

const NESTED: &'static Vec<u8> = {
    // This does not work when we have an inner scope:
    let x = &Vec::new(); //~ ERROR: temporary value dropped while borrowed
    x
};
```

In `EMPTY_BYTES`, the reference obtains the lifetime of the "enclosing scope",
similar to how `let x = &mut x;` creates a reference whose lifetime lasts for
the enclosing scope. This is decided during MIR building already, and does not
involve promotion.

## Open questions

* There is a fourth kind of CTFE failure -- resource exhaustion.  What do we do
  when that happens while evaluating a promoted?
