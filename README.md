# Constant Evaluation

Constant evaluation is the process of running Rust code at compile-time
in order to use the result e.g. to set the discriminant of enum variants
or as an array length.

Examples:

```rust
enum Foo {
    A = 5,
    B = 7 - 3,
}
type NineStrings = [&str, 3 * 3];
```

The Rust compiler runs the [MIR](https://rustc-dev-guide.rust-lang.org/mir/index.html)
in the [`MIR` interpreter (miri)](https://rustc-dev-guide.rust-lang.org/const-eval),
which sort of is a virtual machine using `MIR` as "bytecode".

## Table of Contents

* [Unstable Feature Skill Tree](skill-tree.md)
* [Const Safety](const_safety.md)
* The three "kinds" of compile-time evaluated data:
  * [Statics](static.md) (`static`, `static mut`)
  * [Constants](const.md) (`const`, array sizes, non-`Copy` array initializers)
  * [Promoteds](promotion.md) (rvalue promotion)

## Related RFCs

### Const Promotion

[RFC 1414](https://github.com/rust-lang/rfcs/pull/1414) injects a hidden static for any
`&foo` expression as long as `foo` follows a certain set of rules.
These rules are discussed [here](promotion.md)

### Drop types in constants/statics

[RFC 1440](https://github.com/rust-lang/rfcs/pull/1440) allows using types that implement
`Drop` (or have fields that do so) inside constants and statics. It is guaranteed that the
`Drop::drop` method will never be called on the static/const object.

### Constants in Repeat expressions

[RFC 2203](https://github.com/rust-lang/rfcs/pull/2203) allows the use `!Copy` types for the
initializer of repeat expressions, as long as that value is constant.

This permits e.g. `[Vec::new(); 42]`.

### Statically known bugs and panics in runtime code are warnings

[RFC 1229](https://github.com/rust-lang/rfcs/pull/1229) formalized the concept that

```rust
let x: usize = 1 - 2;
```

is allowed to produce a `lint` but not an `error`. This allows us to make these analyses
more powerful without suddenly breaking compilation. The corresponding lint is the `const_err`
lint, which is `deny` by default and will thus break compilation of the crate currently being built,
even if it does not break the compilation of the current crate's dependencies.

### Various new const-eval features

* [`loop`](https://github.com/rust-lang/rfcs/pull/2344)
* [`if` and `match`](https://github.com/rust-lang/rfcs/pull/2342)
* [`panic!`](https://github.com/rust-lang/rfcs/pull/2345)
* [`locals and destructuring`](https://github.com/rust-lang/rfcs/pull/2341)

Some of these features interact. E.g.

* `match` + `loop` yields `while`
* `panic!` + `if` + `locals` yields `assert!`
