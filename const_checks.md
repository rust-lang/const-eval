# Static and dynamic checks

This repository describes a set of rules that various forms of compile-time evaluated code needs to satisfy.
The compiler contains two kinds of checks that guard against violations of these rules: static checks and dynamic checks.

## Dynamic checks

Dynamic checks are conceptually very simple: when evaluating the compile-time code, we look at what it does, and if what it does violates the rules, we halt evaluation.
Thus, a dynamic check generally makes it very clear what is being protected against.

The main disadvantage of dynamic checks is that they can only run when the compile-time code is being evaluated, which is after monomorphization.
We generally try to avoid post-monomorphization errors as they inherently make for a bad user experience.
While there are technical aspects that could be improved here, the main problem is that the site where the error is reported is disconnected from the site where the root cause is.
Such problems can be observed when creating an associated constant that uses associated constants from generic parameters.
These generic parameters are unknown, so the usage of these associated constants may cause errors depending on the *value* of the generic parameter's associated constants.

[Promotion analysis](promotion.md) also makes little sense dynamically as it is about code transformation.
All we can do is check after the transformation if the generated code makes sense.

## Static checks

Static checks work by "looking" at the code.
(This is "static" as in [static analysis](https://en.wikipedia.org/wiki/Static_analysis), not to be confused with Rust's `static` keyword.)
The idea is to predict what the code will do without actually executing it.
That means we can even analyze code that we cannot run pre-monomorphization.
However, static checks are necessarily less precise than dynamic checks (this is the famous halting problem), which means they will reject code that the dynamic checks would accept.

The key property is that the static check is *sound*, which means that everything accepted by the static checks will also be accepted by the dynamic checks.
It might seem like this makes the dynamic checks unnecessary, but they are still tremendously useful.
On the one hand, the dynamic checks serve as a safety net for when we have bugs in the static checks (dynamic checks are much easier to get right, typically).
On the other hand, having the dynamic checks forces us through the helpful exercise of figuring out what the dynamic property we are enforcing actually *is*, which is crucial when we want to adjust the static check to be more precise.
The dynamic check serves as the "ground truth" that the static check approximates, and we can improve that approximation over time.
