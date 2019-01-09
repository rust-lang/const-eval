- Feature Name: const_generic_const_fn_bounds
- Start Date: 2018-10-05
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow `impl const Trait` for trait impls where all method impls are checked as const fn.

Make it legal to declare trait bounds on generic parameters of const functions and allow
the body of the const fn to call methods on the generic parameters that have a `const` modifier
on their bound.

# Motivation
[motivation]: #motivation

Currently one can declare const fns with generic parameters, but one cannot add trait bounds to these
generic parameters. Thus one is not able to call methods on the generic parameters (or on objects of the
generic parameter type), because they are fully unconstrained.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

You can call methods of generic parameters of a const function, because they are implicitly assumed to be
`const fn`. For example, the `Add` trait declaration has an additional `const` before the trait name, so
you can use it as a trait bound on your generic parameters:

```rust
const fn triple_add<T: const Add>(a: T, b: T, c: T) -> T {
    a + b + c
}
```

The obligation is passed to the caller of your `triple_add` function to supply a type whose `Add` impl is fully
`const`. Since `Add` only has `add` as a method, in this case one only needs to ensure that the `add` method is
`const`. Instead of adding a `const` modifier to all methods of a trait impl, the modifier is added to the entire
`impl` block:

```rust
struct MyInt(i8);
impl const Add for MyInt {
    fn add(self, other: Self) -> Self {
        MyInt(self.0 + other.0)
    }
}
```

The const requirement is required on all bounds of the impl and its methods,
so in the following `H` is required to have a const impl of `Hasher`, so that
methods on `state` are callable.

```rust
impl const Hash for MyInt {
    const fn hash<H>(
        &self,
        state: &mut H,
    )
        where H: const Hasher
    {
        state.write(&[self.0 as u8]);
    }
}
```

While these `const` keywords could be inferred (after all, they are required), requiring them is
forward compatible to schemes in the future that allow more fine grained control.

## Drop

A notable use case of `impl const` is defining `Drop` impls. If you write

```rust
struct SomeDropType<'a>(&'a Cell<u32>);
impl const Drop for SomeDropType {
    fn drop(&mut self) {
        self.0.set(self.0.get() - 1);
    }
}
```

Then you are allowed to actually let a value of `SomeDropType` get dropped within a constant
evaluation. This means `(SomeDropType(&Cell::new(42)), 42).1` is now allowed, because we can prove
that everything from the creation of the value to the destruction is const evaluable.

## Runtime uses don't have `const` restrictions?

`impl const` blocks additionally generate impls that are not const if any generic
parameters are not const.

E.g.

```rust
impl<T: const Add> const Add for Foo<T> {
    fn add(self, other: Self) -> Self {
        Foo(self.0 + other.0)
    }
}
```

allows calling `Foo(String::from("foo")) + Foo(String::from("bar"))` even though that is (at the time
of writing this RFC) most definitely not const, because `String` only has an `impl Add for String`
and not an `impl const Add for String`.

This goes in hand with the current scheme for const functions, which may also be called
at runtime with runtime arguments, but are checked for soundness as if they were called in
a const context. E.g. the following function may be called as
`add(String::from("foo"), String::from("bar"))` at runtime.

```rust
const fn add<T: const Add>(a: T, b: T) -> T {
    a + b
}
```

This feature could have been added in the future in a backwards compatible manner, but without it
the use of `const` impls is very restricted for the generic types of the standard library due to
backwards compatibility.
Changing an impl to only allow generic types which have a `const` impl for their bounds would break
situations like the one described above.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of this RFC is (in contrast to some of its alternatives) mostly
changes around the syntax of the language (allowing `const` modifiers in a few places)
and ensuring that lowering to HIR and MIR keeps track of that.
The miri engine already fully supports calling methods on generic
bounds, there's just no way of declaring them. Checking methods for constness is already implemented
for inherent methods. The implementation will have to extend those checks to also run on methods
of `impl const` items.

## Implementation instructions

1. Add an `is_const` field to the AST's `TraitRef`
2. Adjust the Parser to support `const` modifiers before trait bounds
3. Add an `is_const` field to the HIR's `TraitRef`
4. Adjust lowering to pass through the `is_const` field from AST to HIR
5. Add a a check to `librustc_typeck/check/wfcheck.rs` ensuring that all generic bounds
    in an `impl const` block have the `in_const` flag set and all methods' `constness` field is
    `Const`.
6. Feature gate instead of ban `Predicate::Trait` other than `Sized` in
    `librustc_mir/transform/qualify_min_const_fn.rs`
7. Remove the call in https://github.com/rust-lang/rust/blob/f8caa321c7c7214a6c5415e4b3694e65b4ff73a7/src/librustc_passes/ast_validation.rs#L306
8. Adjust the reference and the book to reflect these changes.

## Const type theory

This RFC was written after weighing practical issues against each other and finding the sweet spot
that supports most use cases, is sound and fairly intuitive to use. A different approach from a
type theoretical perspective started out with a much purer scheme, but, when exposed to the
constraints required, evolved to essentially the same scheme as this RFC. We thus feel confident
that this RFC is the minimal viable scheme for having bounds on generic parameters of const
functions. The discussion and evolution of the type theoretical scheme can be found
[here](https://github.com/rust-rfcs/const-eval/pull/8#issuecomment-452396020) and is only 12 posts
and a linked three page document long. It is left as an exercise to the reader to read the
discussion themselves.

# Drawbacks
[drawbacks]: #drawbacks

It is not a fully general design that supports every possible use case,
but it covers the most common cases. See also the alternatives.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Effect system

A fully powered effect system can allow us to do fine grained constness propagation
(or no propagation where undesirable). This is way out of scope in the near future
and this RFC is forward compatible to have its background impl be an effect system.

## Fine grained `const` annotations

One could annotate methods instead of impls, allowing just marking some method impls
as const fn. This would require some sort of "const bounds" in generic functions that
can be applied to specific methods. E.g. `where <T as Add>::add: const` or something of
the sort. This design is more complex than the current one and we'd probably want the
current one as sugar anyway

## No explicit `const` bounds

One could require no `const` on the bounds (e.g. `T: Trait`) and assume constness for all
bounds. An opt-out via `T: ?const Trait` would then allow declaring bounds that cannot be
used for calling methods. This design causes discrepancies with `const fn` pointers as
arguments (where the constness would be needed, as normal function pointers already exist
as the type of constants). Also it is not forward compatible to allowing `const` trait bounds
on non-const functions

## Infer all the things

We can just throw all this complexity out the door and allow calling any method on generic
parameters without an extra annotation `iff` that method satisfies `const fn`. So we'd still
annotate methods in trait impls, but we would not block calling a function on whether the
generic parameters fulfill some sort of constness rules. Instead we'd catch this during
const evaluation.

This is strictly the most powerful and generic variant, but is an enormous backwards compatibility
hazard as changing a const fn's body to suddenly call a method that it did not before can break
users of the function.

# Future work

This design is explicitly forward compatible to all future extensions the author could think
about. Notable mentions (see also the alternatives section):

* an effect system with a "notconst" effect
* const trait bounds on non-const functions allowing the use of the generic parameter in
  constant expressions in the body of the function or maybe even for array lenghts in the
  signature of the function
* fine grained bounds for single methods and their bounds

It might also be desirable to make the automatic `Fn*` impls on function types and pointers `const`.
This change should probably go in hand with allowing `const fn` pointers on const functions
that support being called (in contrast to regular function pointers).

## Deriving `impl const`

```rust
#[derive(Clone)]
pub struct Foo(Bar);

struct Bar;

impl const Clone for Bar {
    fn clone(&self) -> Self { Bar }
}
```

could theoretically have a scheme inferring `Foo`'s `Clone` impl to be `const`. If some time
later the `impl const Clone for Bar` (a private type) is changed to just `impl`, `Foo`'s `Clone`
impl would suddenly stop being `const`, without any visible change to the API. This should not
be allowed for the same reason as why we're not inferring `const` on functions: changes to private
things should not affect the constness of public things, because that is not compatible with semver.

One possible solution is to require an explicit `const` in the derive:

```rust
#[derive(const Clone)]
pub struct Foo(Bar);

struct Bar;

impl const Clone for Bar {
    fn clone(&self) -> Self { Bar }
}
```

which would generate a `impl const Clone for Foo` block which would fail to compile if any of `Foo`'s
fields (so just `Bar` in this example) are not implementing `Clone` via `impl const`. The obligation is
now on the crate author to keep the public API semver compatible, but they can't accidentally fail to
uphold that obligation by changing private things.

## RPIT (Return position impl trait)

```rust
const fn foo() -> impl Bar { /* code here */ }
```

does not allow us to call any methods on the result of a call to `foo`, if we are in a
const context. It seems like a natural extension to this RFC to allow

```rust
const fn foo() -> impl const Bar { /* code here */ }
```

which requires that the function only returns types with `impl const Bar` blocks.

## Specialization

Impl specialization is still unstable. There should be a separate RFC for declaring how
const impl blocks and specialization interact. For now one may not have both `default`
and `const` modifiers on `impl` blocks.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Everything has been addressed in the reviews
