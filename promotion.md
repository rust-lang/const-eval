# Const promotion

Note that promotion happens on the MIR, not on surface-level syntax.  This is
relevant when discussing e.g. handling of panics caused by overflowing
arithmetic.

## Rules

### 1. No side effects

Promotion is not allowed to throw away side effects.  This includes
panicking. let us look at what happens when we promote `&(0_usize - 1)`:
In the MIR, this looks roughly like
```
_tmp1 = CheckedSub (const 0usize) (const 1usize)
assert(!_tmp1.1) -> [success: bb2; unwind: ..]

bb2:
_tmp2 = tmp1.0
_res = &_tmp2
```
Both `_tmp1` and `_tmp2` are promoted to statics.  `_tmp1` evaluates to `(~0,
true)`, so the assertion will always fail at run-time.  Computing `_tmp2` fails
with a panic, which is thrown away -- so we have no result.  In principle, we
could generate any code for this because we know the code is unreachable (the
assertion is going to fail).  Just to be safe, we generate a call to
`llvm.trap`.

### 2. Const safety

Only const safe code gets promoted. The exact details for `const safety` are
discussed in [here](const_safety.md).

An example of this would be `&(&1 as *const i32 as usize % 16 == 0)`. The actual
location is not known at compile-time, so we cannot promote this.  Generally, we
can guarantee const-safety by not promoting when an unsafe or unconst operation
is performed.  However, things get more tricky when `const` and `const fn` are
involved.

For `const`, based on the const safety check described [here](const_safety.md),
we can rely on there not being const-unsafe values in the `const`, so we should
be able to promote freely.

For `const fn`, there is no way to check anything in advance.  We can either
just not promote, or we can move responsibility to the `const fn` and promote
*if* all function arguments pass the const safety check.  So, `foo(42usize)`
would get promoted, but `foo(&1 as *const i32 as usize)` would not.  When this
call panics, compilation proceeds and we just hard-code a panic to happen as
well at run-time.  However, when const evaluation fails with another error, we
have no choice but to abort compilation of a program that would have compiled
fine if we would not have decided to promote.  It is the responsibility of `foo`
to not fail this way when working with const-safe arguments.
