# Const promotion

## Rules

### 1. No side effects

Promotion is not allowed to throw away side effects.
This includes panicking. So in order to promote `&(0_usize - 1)`,
the subtraction is thrown away and only the panic is kept.

### 2. Const safety

Only const safe code gets promoted. This means that promotion doesn't happen
if the code does some action which, when run at compile time, either errors or
produces a value that differs from runtime.

An example of this would be `&1 as *const i32 as usize == 42`. While it is highly
unlikely that the address of temporary value is `42`, at runtime this could be true.

The exact details for `const safety` are discussed in [here](const_safety.md).