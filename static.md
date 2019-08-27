# Statics

Statics (`static`, `static mut`) are the simplest kind of compile-time evaluated data:
* The user explicitly requested them to be evaluated at compile-time,
  so evaluation errors from computing the initial value of a static are no concern
  (in other words, [const safety](const_safety.md) is mostly not an issue).
* They observably get evaluated *once*, with the result being put at some address known at run-time,
  so there are no fundamental restrictions on what statics can do.
* The compiler checks that statics are `Sync`, justifying sharing their address across threads.
* [Constants](const.md) and [promoteds](promotion.md) are not allowed to read from statics,
  so their final value does not have have to be [const-valid](const_safety.md#const-safety-check-on-values) in any meaningful way.
  As of 2019-08, we do check them for validity anyway, to be conservative; and indeed constants could be allowed to read from frozen statics.

## `Drop`

The compiler rejects intermediate values (created and discarded during the computation of a static initializer) that implement `Drop`.
The reason for this is simply that the `Drop` implementation might be non-`const fn`.
This restriction can be lifted once `const impl Drop for Type` (or something similar) is supported.

```rust
struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("foo dropped");
    }
}

static FOOO: Foo = Foo; // Ok, drop is never run

// Not ok, cannot run `Foo::drop` because it's not a const fn
static BAR: i32 = (Foo, 42).1;
```

*Dynamic check.* The Miri engine dynamically checks that this is done correctly
by not permitting calls of non-`const` functions.
