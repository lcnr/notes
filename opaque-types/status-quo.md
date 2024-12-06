# status quo

## mir borrowck

Start with `replace_regions_in_mir` which replaces all regions in the MIR and its promoteds.

This first creates the `UniversalRegions` for the given body:
- `'static` at index `0`
- the early-bound regions of the item, created in `fn defining_ty`
    - actual early bound regions for items + regions in the closure signature, inline const return type
- the late-bound regions of each parent *inside-out*
    - for each parent, we iterate over `tcx.late_bound_vars` and add a new universal region for each one
    - if the parents are 3 nested closure `a b c`, we first add the late-bound regions for the parent `b` and then for `a`

It adds a map from `'static` and the actual early-bound regions to their index.

