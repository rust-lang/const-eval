# Const promotion

"Promotion" is the act of splicing a part of a MIR computation out into a
separate self-contained MIR body which is evaluated at compile-time like a
constant. This mechanism has been introduced by [RFC 1414][promotion-rfc] with
the goal of equipping some references-to-temporaries with a `'static` lifetime,
which is sometimes called "lifetime extension".

[promotion-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1414-rvalue_static_promotion.md

Promotion / lifetime extension affects code like `&3`: Instead of putting it on
the stack, the `3` is allocated in global static memory and a reference with
lifetime `'static` is provided.  This is essentially an automatic transformation
turning `&EXPR` into `{ const _PROMOTED = &EXPR; _PROMOTED }`, but only if
`EXPR` qualifies. Topmost projections are not promoted, so `&EXPR.proj1.proj2`
turns into `{ const _PROMOTED = &EXPR; &(*_PROMOTED).proj1.proj2 }`.

Note that promotion happens on the MIR, not on surface-level syntax.  This is
relevant when discussing e.g. handling of panics caused by overflowing
arithmetic.

## Promotion and fallability of const-evaluation

On top of what applies to [consts](const.md), promoteds suffer from the
additional issue that *the user did not ask for them to be evaluated at
compile-time*.  Thus, if CTFE fails but the code would have worked fine at
run-time, we broke the user's code for no good reason.  Even if we are sure we
found an error in the user's code, we are only allowed to
[emit a warning, not a hard error][warn-rfc].

For example:
```rust
fn foo() {
    if false {
        let x = &(1/0);
    }
}
```
If we performed promotion here, this would turn into
```rust
fn foo() {
    if false {
        const _PROMOTED = &(1/0);
        let x = _PROMOTED;
    }
}
```
When compiling this function, we have to evaluate all constants that occur
inside the function body, even if they might only be used in dead code. This
means we have to evaluate `_PROMOTED`, which will error -- and now what, should
we halt compilation? That would be wrong since there is no problem with this
code, the "bad" division never actually happens as it occurs in dead code.
(Note that the considerations would be the same even if `foo` were a `const
fn`.)

As a consequence, we only promote code that can never fail to evaluate (see
[RFC 3027]). This ensures that even if promotion happens inside dead code, this
will not turn a "runtime error in dead code" (which is not an error at all) into
a compile-time error. In particular, we cannot promote calls to arbitrary `const
fn`, as discussed in detail in
[rust-lang/const-eval#19](https://github.com/rust-lang/const-eval/issues/19).
Thus, only functions marked `#[rustc_promotable]` are promotable.

[RFC 3027]: https://rust-lang.github.io/rfcs/3027-infallible-promotion.html

There is one exception to this rule: the bodies of `const`/`static`
initializers. This code is never compiled, so we do not actually have to
evaluate constants that occur in dead code. If we are careful enough during
compilation, we can ensure that only constants whose value is *actually needed*
are evaluated. We thus can be more relaxed about promotion; in practice, what
this means is that we will promote calls to arbitrary `const fn`, not just those
marked `#[rustc_promotable]`.

[warn-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/1229-compile-time-asserts.md

## "enclosing scope" rule

Notice that some code involving `&` *looks* like it relies on promotion /
lifetime extension but actually does not:

```rust
const EMPTY_BYTES: &Vec<u8> = &Vec::new(); // Ok without lifetime extension
```

`Vec::new()` cannot get promoted because it needs dropping. And yet this
compiles. Why that? The reason is that the reference obtains the lifetime of
the "enclosing scope", similar to how `let x = &mut x;` creates a reference
whose lifetime lasts for the enclosing scope. This is decided during MIR
building already, and does not involve lifetime extension.

In contrast, this does not compile:

```rust
const OPT_EMPTY_BYTES: Option<&Vec<u8>> = Some(&Vec::new());
```

The "enclosing scope" rule only fires for outermost `&`, just like in `fn` bodies.

## Promotability

We have described the circumstances where promotion is desirable, but what
expressions are actually eligible for promotion? We refer to eligible
expressions as "promotable" and describe the restrictions on such expressions
below.

First of all, expressions have to be [allowed in constants](const.md). The
restrictions described there are needed because we want `const` to behave the
same as copying the `const` initializer everywhere the constant is used; we need
the same property when promoting expressions. But we need more.

Note that there is no point in doing additional dynamic checks to ensure that we
do get these restrictions right. The entire point of the promotion restrictions
is to avoid failing compilation for code that would have been fine without
promotion. The best a dynamic check could do is tell us after the fact that we
should not have promoted something, but then it is already too late -- and the
dynamic checks for that are exactly the ones we are already doing for constants
and statics.

### Panics, overflow and bounds checks

Let us look at what happens when we promote `&(0_usize - 1)` in a debug build.
This code is promoted even though we cannot promote code that could fail, and
this code will fail with an overflow error! What is happening?  We have to look
at the underlying MIR representation of this code to explain what happens:

```
_tmp1 = CheckedSub (const 0usize) (const 1usize)
assert(!_tmp1.1) -> [success: bb2; unwind: ..]

bb2:
_tmp2 = tmp1.0
_res = &_tmp2
```

Both `_tmp1` and `_tmp2` are promoted.  `_tmp1` evaluates to `(~0, true)`, so
the assertion will always fail at run-time. Computing `_tmp2` evaluates to `~0`.

In other words, the actually failing check is not promoted, only the computation
that serves as input to the check is promoted.

An earlier version of Miri used to error on arithmetic overflow even in release
mode. This breaks promotion, because now promoting code like `_tmp1` would
introduce promotes that fail to evaluate, which is not acceptable as explained
above!

Something similar but more subtle happens when promoting array accesses: the
bounds check is not promoted, but the array access is. However, before accepting
a temporary for promotion, we ensure that array accesses are definitely
in-bounds. This leads to MIR without bounds checks, but we know the array access
will always succeed.

### Const safety

We have explained how we ensure that evaluating a promoted does not panic, but
what about other kinds of failure -- what about hitting an unsupported operation
or undefined behavior? To make sure this does not happen, only const safe code
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
`#[rustc_promotable]` attribute are subject to promotion. Those functions must
be manually reviewed to never raise CTFE errors.

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

### Access to a `const` or `static`
[access-static]: #access-to-a-const-or-static

When accessing a `const` in a promotable context, its value gets computed
at compile-time anyway, so we do not have to check the initializer.  However, the
restrictions described above still apply for the *result* of the promoted
computation: in particular, it must be a valid `const` (i.e., it cannot
introduce interior mutability) and it must not require dropping.

For instance the following would be legal even though calls to `do_it` are not
eligible for implicit promotion:

```rust
const fn do_it(x: i32) -> i32 { 2*x }
const ANSWER: i32 = {
  let ret = do_it(21);
  ret
};

let x: &'static i32 = &ANSWER;
```

An access to a `static`, including just taking references to a `static`, is only
promotable within the initializer of another `static`. This is for the same
reason that `const` initializers
[cannot access statics](const.md#reading-statics).

Crucially, however, the following is *not* legal:

```rust
const X: Cell<i32> = Cell::new(5); // ok
const XREF: &Cell<i32> = &X; // not ok
fn main() {
    let x: &'static _ = &X; // not ok
}
```

Just like allowing `XREF` would be a problem because, by the inlining semantics,
every user of `XREF` should get their own `Cell`; it would also be a problem to
promote here because if that code gets executed multiple times (e.g. inside a
loop), it should get a new `Cell` each time.

### Named locals

Promotable expressions cannot refer to named locals. This is not a technical
limitation with the CTFE engine. While writing `let x = {expr}` outside of a
const context, the user likely expects that `x` will live on the stack and be
initialized at run-time.  Although this is not (to my knowledge) guaranteed by
the language, we do not wish to violate the user's expectations here.

Note that constant-folding still applies: the optimizer may compute `x` at
compile-time and even inline it everywhere if it can show that this does not
observably alter program behavior.  Promotion is very different from
constant-folding as promotion can introduce observable differences in behavior
(if const-evaluation fails) and as it is *guaranteed* to happen in some cases
(and thus exploited by the borrow checker).  This is reflected in the fact that
promotion affects lifetimes, but constant folding does not.

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

## Open questions

* There is a fourth kind of CTFE failure -- resource exhaustion.  What do we do
  when that happens while evaluating a promoted?
