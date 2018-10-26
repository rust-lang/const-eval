# Further restrictions for constants

The [const safety](const_safety.md) concerns apply to all computations happening
at compile-time, `static` and `const` alike.  However, there are some additional
considerations about `const` specifically.  These arise from the idea that
```rust
const CONST: T = EXPR;
```
is supposed to behave as-if `EXPR` was written at every use site of `CONST`.

## References

One issue is constants of reference type:
```rust
const REF: &u32 = &EXPR;
```
Instead of creating a new allocation for storing the result of `EXPR` on every
use site, we just have a single global "static" allocation and every use of
`REF` uses its address.  It's as if we had written:
```rust
const REF: &u32 = { const _VAL = EXPR; static _STATIC = EXPR; &_STATIC };
```
(`EXPR` is assigned to a `const` first to make it subject to the restrictions
discussed in this document.)

There are three reasons why this could be an issue.

### Pointer equality

We effectively "deduplicate" all the allocations that would otherwise locally be
created at each use site of `REF`.  This is observable when the program compares
these pointers for equality.  We consider this okay, i.e., programs may not rely
on such constants all getting distinct addresses.  They may not rely on them all
getting the same address either.

### Interior mutability

If the reference has type `&Cell<i32>` it is quite clear that the program can
easily observe whether two references point to the same memory even without
comparing their address: Changes through one reference will affect reads through
the other.  So, we cannot allow constant references to types that have interior
mutability (types that are not `Freeze`).

However, we can do better than that: Even if a *type* is not `Freeze`, it can
have *values* that do not exhibit any interior mutability.  For example, `&None`
at type `&Option<Cell<i32>>` would be rejected by the naive analysis above, but
is actually accepted by the compiler because we know that there is no
`UnsafeCell` here that would permit interior mutability.

### `Sync`

Finally, the same constant reference is actually shared across threads.  This is
very similar to multiple threads having a shared reference to the same `static`,
which is why `static` must be `Sync`.  So it seems like we should reject
non-`Sync` types, conforming with the desugaring described above.

However, this does not currently happen, and there are several crates across the
ecosystem that would break if we just started enforcing this now. See
[this issue](https://github.com/rust-lang/rust/issues/49206) and the
[PR attempting to fix this](https://github.com/rust-lang/rust/pull/54424/).

### `Drop`

Values of "needs drop" types
can only be used as the final initialization value of a `const` or `static` item.
They may not be used as intermediate values that would be dropped before the item
were initialized. As an example:

```rust
struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("foo dropped");
    }
}

const FOO: Foo = Foo; // Ok, drop is run at each use site in runtime code
static FOOO: Foo = Foo; // Ok, drop is never run

// Not ok, cannot run `Foo::drop` because it's not a const fn
const BAR: i32 = (Foo, 42).1;
```

This restriction might be lifted in the future after trait impls
may be declared `const` (https://github.com/rust-rfcs/const-eval/pull/8).

Note that in promoteds this restriction can never be lifted, because
otherwise we would silently stop calling the `Drop` impl at runtime and
pull it to much earlier (compile-time).

## Reading statics

Beyond values of reference type, we have to be careful that *computing* a
`const` cannot read from a static that could get mutated (because it is `static
mut`, or because it has interior mutability).  That would lead to the constant
expression (being computed at compile-time) not having the same value any more
when evaluated at run-time.

This is distinct to the concern about interior mutability above: That concern
was about first computing a `&Cell<i32>` and then using it at run-time (and
observing the fact that it has been "deduplicated"), this here is about using
such a value at compile-time even though it might be changed at run-time.
