- Feature Name: `const_ub`
- Start Date: 2020-10-10
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Define UB during const evaluation to lead to an unspecified result for the affected CTFE query, but not otherwise infect the compilation process.

# Motivation
[motivation]: #motivation

So far, nothing is specified about what happens when `unsafe` code leads to UB during CTFE.
This is a major blocker for stabilizing `unsafe` operations in const-contexts.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are some values that Rust needs to compute at compile-time.
This includes the initial value of a `const`/`static`, and array lengths (and more general, const generics).
Computing these initial values is called compile-time function evaluation (CTFE).
CTFE in Rust is very powerful and permits running almost arbitrary Rust code.
This begs the question, what happens when there is `unsafe` code and it causes Undefined Behavior (UB)?

The answer is that in this case, the final value that is currently being executed is arbitrary.
For example, when UB arises while computing an array length, then the final array length can be any `usize`, or it can be (partially) uninitialized memory.
No guarantees are made about this final value, and it can be different depending on host and target architecture, compiler flags, and more.
However, UB will not otherwise adversely affect the currently running compiler; type-checking and lints and everything else will work correctly given whatever the result of the CTFE computation is.

Note, however, that this means compile-time UB can later cause runtime UB when the program is actually executed:
for example, if there is UB while computing the initial value of a `Vec<i32>`, the result might be a completely invalid vector that causes UB at runtime when used in the program.

Sometimes, the compiler might be able to detect such problems and show an error or warning about CTFE computation having gone wrong (for example, the compiler might detect when the array length ends up being uninitialized).
But other times, this might not be the case -- UB is not reliably detected during CTFE.
This can change from compiler version to compiler version: CTFE code that causes UB could build fine with one compiler and fail to build with another.
(This is in accordance with the general policy that unsound code is not subject to strict stability guarantees.)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When UB arises as part of CTFE, the result of this evaluation is an unspecified constant.
The compiler might be able to detect that UB occurred and raise an error or a warning, but this is not mandated, and absence of lints does not imply absence of UB.

# Drawbacks
[drawbacks]: #drawbacks

This means UB during CTFE can silently "corrupt" the build in a way that the final program has UB when being executed
(but not more so than if the CTFE code would instead have been run at runtime).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The most obvious alternative is to say that UB during CTFE will definitely be detected.
However, that is expensive and might even be impossible.
Even Miri does not currently detect all UB, and Miri is already performing many additional checks that would significantly slow down CTFE.
Furthermore, since optimizations can "hide" UB (an optimization can turn a program with UB into one without), this means we would have to run CTFE on unoptimized MIR.
And finally, implementing these checks requires a more precise understanding of UB than we currently have; basically, this would block having any potentially-UB operations at const-time on having a spec for Rust that precisely describes their UB in a checkable way.
In particular, this would mean we need to decide on an aliasing model before permitting raw pointers in CTFE.

Another extreme alternative would be to say that UB during CTFE may have arbitrary effects in the host compiler, including host-level UB.
Basically this would mean that CTFE would be allowed to "leave its sandbox".
This would allow JIT'ing CTFE and running the resulting code unchecked.
While compiling untrusted code should only be done with care (including additional sandboxing), this seems like an unnecessary extra footgun.

# Prior art
[prior-art]: #prior-art

C++ requires compilers to detect UB in `constexpr`.
However, the fragment of C++ that is available to `constexpr` excludes pointer casts (TODO: and pointer arithmetic and unions?), which makes such checks not very complicated and avoids all the poorly specified parts of UB.

If we found a way to run CTFE on unoptimized MIR, then detecting UB for programs that do not use unions, `transmute`, or raw pointers is not very hard.
CTFE already has almost all the checks required for this, except for alignment checks which are disabled during CTFE.
(Disabling them was the easiest way forward to solve some issues around packed structs in patterns, but we could use a different solution and reinstate CTFE alignment checks.
The relevant code paths still exist for Miri.)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Currently none.

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC provides an easy way forward for "unconst" operations, i.e., operations that are safe at run-time but not at compile-time.
Primary examples of such operations are anything involving the integer representation of pointers, which cannot be known at compile-time.
If this RFC were accepted, we could declare such operations UB during CTFE (and thus naturally they would only be permitted in an `unsafe` block).
This still leaves the door open for providing better guarantees in the future.
