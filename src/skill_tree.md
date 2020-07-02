# Skill tree for const eval features

<style>
      /* the lines within the edges */
      .edge:active path,
      .edge:hover path {
        stroke: fuchsia;
        stroke-width: 3;
        stroke-opacity: 1;
      }
      /* arrows are typically drawn with a polygon */
      .edge:active polygon,
      .edge:hover polygon {
        stroke: fuchsia;
        stroke-width: 3;
        fill: fuchsia;
        stroke-opacity: 1;
        fill-opacity: 1;
      }
      /* If you happen to have text and want to color that as well... */
      .edge:active text,
      .edge:hover text {
        fill: fuchsia;
      }
</style>

```skill-tree
[[group]]
name = "mut_ref"
label = "mutable references"
href = "https://github.com/rust-lang/rust/issues/57349"
items = []

[[group]]
name = "const_mut_ref"
label = "constants with mutable\nreferences in their final value"
href = "https://github.com/rust-lang/rust/issues/71212"
requires = ["mut_ref"]
items = []

[[group]]
name = "file_output"
label = "write to files\nfrom constants"
href = "https://github.com/rust-lang/const-eval/issues/25"
items = []

[[group]]
name = "final_heap"
label = "heap allocations\nin the final value of constants"
requires = ["heap"]
href = "https://github.com/rust-lang/const-eval/issues/20"
items = []

[[group]]
name = "heap"
label = "heap allocations"
requires = []
items = []
href = "https://github.com/rust-lang/const-eval/issues/20"

[[group]]
name = "iterators"
label = "iterators"
requires = ["mut_ref", "trait_impl"]
items = []

[[group]]
name = "iterator combinators"
label = "iterator_combinators"
requires = ["iterators", "closures"]
items = []

[[group]]
name = "for"
label = "for loops"
requires = ["trait_impl", "mut_ref"]
items = []

[[group]]
name = "slice_eq"
label = "[T]::eq"
requires = ["for", "fuzzy-ptr-comparisons", "trait_impl"]
items = []

[[group]]
name = "transmute"
label = "transmute"
requires = ["unconst_rules"]
items = []

[[group]]
name = "box_new"
label = "Box::new"
requires = ["heap", "trait_bound_opt_out", "drop"]
items = []

[[group]]
name = "ptr-is-null"
label = "<*T>::is_null"
requires = ["fuzzy-ptr-comparisons"]
items = []

[[group]]
name = "fuzzy-ptr-comparisons"
label = "guaranteed_eq and\nguaranteed_ne"
requires = []
items = []

[[group]]
name = "unconst_rules"
label = "Need to come up\nwith a scheme\nfor doing unsafe\nin const fn"
items = []
href = "https://github.com/rust-lang/const-eval/issues/14"

[[group]]
name = "question_mark"
label = "using ? in const"
requires = ["trait_impl"]
items = []

[[group]]
name = "mutex_new"
label = "Mutex::new"
href = "https://github.com/rust-lang/rust/issues/66806"
items = []
requires = ["final_heap", "parking_lot", "unconst_rules"]

[[group]]
name = "const_fn_in_patterns"
label = "const fn callable in patterns"
href = "https://github.com/rust-lang/rust/issues/57240"
requires = ["rfc:2920"]
items = []

[[group]]
name = "from_str"
label = "FromStr"
href = "https://github.com/rust-lang/rust/issues/59133"
requires = ["trait_impl"]
items = []

[[group]]
name = "int_parse"
label = "<int>::from_str"
requires = ["from_str", "iterators"]
items = []

[[group]]
name = "const-float"
label = "floats in const fn"
href = "https://github.com/rust-lang/rust/issues/57241"
items = [
    { label = "from_bits" },
    { label = "to_bits" },
    { label = "general usage of float math,\narguments and return types" },
]

[[group]]
name = "const-assert"
label = "assert!"
requires = ["panic"]
items = []

[[group]]
name = "const-assert-eq"
label = "assert_eq!"
requires = ["const-assert", "trait_impl", "panic_fmt"]
items = []

[[group]]
name = "rfc"
items = [
    { label = "const blocks", port = "2920", href = "https://github.com/rust-lang/rfcs/pull/2920" },
]

[[group]]
label = "panic! with formatting"
name = "panic_fmt"
requires = ["format_args", "panic"]
items = []

[[group]]
label = "feature gate\nconst_panic"
name = "panic"
href = "https://github.com/rust-lang/rust/issues/51999"
items = []

[[group]]
label = "feature gate\nconst_discriminant"
name = "discriminant"
href = "https://github.com/rust-lang/rust/pull/69825"
items = []

[[group]]
label = "feature gate\nconst_trait_bound_opt_out"
name = "trait_bound_opt_out"
href = "https://github.com/rust-lang/rust/issues/67794"
items = []

[[group]]
label = "feature gate\nconst_trait_impl"
name = "trait_impl"
href="https://github.com/rust-lang/rust/issues/67792"
items = []

[[group]]
label = "feature gate\nconst_raw_ptr_deref"
name = "raw_ptr_deref"
href="https://github.com/rust-lang/rust/issues/51911"
items = []
requires = ["unconst_rules"]

[[group]]
label = "feature gate\nconst_raw_ptr_to_usize_cast"
name = "raw_ptr_to_usize_cast"
href="https://github.com/rust-lang/rust/issues/51910"
items = []
requires = ["unconst_rules"]

[[group]]
label = "feature gate\nconst_fn_union"
name = "union"
href = "https://github.com/rust-lang/rust/issues/51909"
items = []
requires = ["unconst_rules"]

[[group]]
label = "feature gate\nconst_extern_fn"
name = "extern_const_fn"
href = "https://github.com/rust-lang/rust/issues/64926"
items = []

[[group]]
label = "const function pointers"
name = "const_fn_ptr"
href = "https://github.com/rust-lang/rust/issues/63997"
items = []
requires = ["trait_impl", "trait_bound_opt_out"]

[[group]]
label = "format!"
name = "format"
items = []
requires = ["string", "format_args"]

[[group]]
label = "format_args!"
name = "format_args"
items = []
requires = ["trait_impl", "fuzzy-ptr-comparisons"]

[[group]]
label = "String operations"
name = "string"
items = []
requires = ["vec"]

[[group]]
label = "Vec operations"
name = "vec"
items = []
requires = ["mut_ref", "heap", "trait_impl", "drop", "raw_ptr_deref"]

[[group]]
label = "Drop"
name = "drop"
items = []
requires = ["mut_ref", "trait_impl"]

[[group]]
label = "Defining and invoking\nClosures"
name = "closure"
items = []
requires = []

[[group]]
label = "Unsized locals"
name = "unsized_locals"
items = []
requires = []
```
