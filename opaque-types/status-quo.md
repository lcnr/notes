# status quo

## mir borrowck

### `replace_regions_in_mir`

This replaces all regions in the MIR and its promoteds.

This first creates the `UniversalRegions` for the given body:
- `'static` at index `0`
- the early-bound regions of the item, created in `fn defining_ty`
    - actual early bound regions for items + regions in the closure signature, inline const return type
- the late-bound regions of each parent *inside-out*
    - for each parent, we iterate over `tcx.late_bound_vars` and add a new universal region for each one
    - if the parents are 3 nested closure `a b c`, we first add the late-bound regions for the parent `b` and then for `a`
- the whole body `fr_fn_body`

It adds a map from `'static` and the actual early-bound regions to their index.

We now replace all regions in the body with fresh vars. This does not rely on the `UniversalRegions` but
we have to first create these so that they have the lowest indices.

### `register_predefined_opaques_for_next_solver`

This currently happens here

### compute non-region metadata

Precompute a bunch of necessary information later used by region checking.

#### `LocationTable`

A simple utility struct for mapping temporal locations to their index.

#### `MoveData`

precomputes information about moves and initializations, both the locations where they happen and the places they affect. Computed via a simple walk over the body.

#### `MaybeInitializedPlaces`

For each location in the cfg, compute whether a local *may* be initialized at a given point. Computed via fixpoint.

#### `BorrowSet`

Precomputed information about borrows in the cfg. This is a simple walk over the cfg. The only place introducing borrows are assignments. Two-phase borrows get actived when the assigned location is used.

We have to ignore some borrows. We ignore borrows of places which have whose projections contain a dereference of either a raw pointer or an immutable reference, e.g. `&*x` or `&(*x).field`. These borrows can never overlap with anything. The outlives constraints introduced by such borrows are handled separately.

For each borrow we store its [`BorrowData`](https://github.com/rust-lang/rust/blob/37e74596c0b59e81b9ac58657f92297ef4ccb7ef/compiler/rustc_borrowck/src/borrow_set.rs#L74-L88). Most notably, this contains the region for which the borrow is live, as well as the kind of borrow.

### `nll::compute_regions`

This starts by running the MIR type-checker. This first computes the the assumptions which are known to hold inside of the function.
- `UniversalRegionRelations`
    - it wraps the already frozen `UniversalRegions` computed early on
    - the assumed outlives relations of these free regions
- `RegionBoundPairs`: the assumed `TypeOutlives` from implied bounds.
- `known_type_outlives_obligations`: assumed `TypeOutlives` from where-clauses

We also compute the `normalized_inputs_and_output`. Using this, we can now start type checking in earnest. 

We check user type-annotations. This is responsible for checking that the yet unconstrained nll vars are be equal to the universal regions provided by the user.

We walk the body twice. Once with the `TypeVerifier`, and once in `TypeChecker::typeck_mir`. While these two uses are distinct, we should merge them to a single pass.

#### `liveness::generate`

After collecting all constraints by walking the body, we compute liveness information.

As an optimization, we only compute liveness for locals where it may be relevant. We don't need to compute liveness information for locals referencing at least one region local to the body, i.e. which does not already outlive a free region. This is purely an optimization. This depends on the current state of the region constraint graph.

We then call `trace::trace`. For each local it computes all locations at which it needs to be live. This is done by walking the CFG backwards until any writes of the local as whether a local is live before being overwritten does not matter. We do so both for reads of the local, in which case it has to be *USE-LIVE*, and drops of the local, in which case it only has to be *DROP-LIVE*.


? it mutates on `typeck`

## region assumptions

Region constraints are stored in the `UniversalRegionRelations` in borrowck and the `FreeRegionMap` during lexical region resolution.

Type outlives are stored both in `region_bound_pairs` and in `known_type_outlives_obligations`.

The `region_bound_pairs` only contain already destructured type outlives only added in `UniversalRegionRelationsBuilder::add_outlives_bound` and `OutlivesEnvironment::with_bounds`. It is only ever used for type outlives constraints from implied bounds.

The `known_type_outlives_obligations` only contain the explicit outlives constraints from the `param_env`. We must not desugar outlives bounds from the environment as otherwise `where &'a &'b: 'c` would imply `'b: 'a` without this getting checked by the user.