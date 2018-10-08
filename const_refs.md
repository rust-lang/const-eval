# References in constants

Constants of reference type are not an entirely straight-forward topic, for
reasonings unrelated to [const safety](const_safety.md).  The issue is that
every use of a constant like
```rust
const REF: &u32 = &42;
```
is supposed to behave as if the value of the constant was copy-pasted into every
place where it is used.  However, the *real* behavior is that a single global
"static" allocation is created containing the `42`, and every use of `REF` gets
evaluated to the address of that static.  There are three reasons why this could
be an issue.

## Pointer equality

We effectively "deduplicate" all the `42` that would otherwise locally be
created at each use site of `REF`.  This is observable when the programs
compares these pointers for equality.  We consider this okay, i.e., programs may
not rely on such constants all getting distinct addresses.  They may not rely on
them all getting the same address either.

## Interior mutability

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

## `Sync`

Finally, the same constant reference is actually shared across threads.  This is
very similar to multiple threads having a shared reference to the same `static`,
which is why `static` must be `Sync`.  So it seems like we should reject
non-`Sync` types.

However, this does not currently happen, and there are several crates across the
ecosystem that would break if we just started enforcing this now. See
[this issue](https://github.com/rust-lang/rust/issues/49206) and the
[PR attempting to fix this](https://github.com/rust-lang/rust/pull/54424/).
