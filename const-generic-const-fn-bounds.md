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

You can call call methods of generic parameters of a const function, because they are implicitly assumed to be
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

The const requirement is propagated to all bounds of the impl or its methods,
so in the following `H` is required to have a const impl of `Hasher`, so that
methods on `state` are callable.

```rust
impl const Hash for MyInt {
    fn hash<H>(
        &self,
        state: &mut H,
    )
        where H: Hasher
    {
        state.write(&[self.0 as u8]);
    }
}
```

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

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of this RFC is (in contrast to some of its alternatives) mostly
changes around the syntax of the language (adding `const` modifiers in a few places)
and ensuring that lowering to HIR and MIR keeps track of that.
The miri engine already fully supports calling methods on generic
bounds, there's just no way of declaring them. Checking methods for constness is already implemented
for inherent methods. The implementation will have to extend those checks to also run on methods
of `impl const` items.

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

Should `impl const` blocks additionally generate impls that are not const if any generic
parameters are not const?

E.g.

```rust
impl<T: Add> const Add for Foo<T> {
    fn add(self, other: Self) -> Self {
        Foo(self.0 + other.0)
    }
}
```

would allow calling `Foo(String::new()) + Foo(String::new())` even though that is (at the time
of writing this RFC) most definitely not const, because `String` only has an `impl Add for String`
and not an `impl const Add for String`.

This would go in hand with the current scheme for const functions, which may also be called
at runtime with runtime arguments, but are checked for soundness as if they were called in
a const context.

## Require `const` bounds on everything inside an `impl const` block?

Instead of inferring `const`ness on all bounds and functions inside a `impl const` block,
we force the user to supply these bounds. This is more consistent with not inferring `const`
on `const` function argument types and generic bounds. The `Hash` example from above would
then look like

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
