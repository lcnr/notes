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

It starts by creating the `DenseLocationMap`, mapping from `Locations` - `block` and `statement` index - to a dense set of `PointIndex`.

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

---

We then call `trace::trace`. For each local it computes all locations at which it needs to be live. This is done by walking the CFG backwards until any writes of the local as whether a local is live before being overwritten does not matter. We do so both for reads of the local, in which case it has to be *USE-LIVE*, and drops of the local, in which case it only has to be *DROP-LIVE*.

It first builds a `LocalUseMap` by walking the body and collecting the list of definitions, uses, and drops of each relevant local.

`LivenessResults::compute_for_all_locals` then computes liveness for each individual relevant local as described above.

---

We then mark all regions as live in the location where they occur in the body. This is necessary for regions in the body which aren't used by any live locals.

Finally we register member constraints for all opaque types in the storage at this point. More on these later. We then convert all regions mentioned by the opaque to nll vars.

#### `RegionInferenceContext::new`

We've now collected all constraints from this body and do the actual region solving. `RegionInferenceContext::new` starts by adding `'scc: 'static` constraints for all components which are required to outlive a placeholder.

We then build the constraint graph and sccs for our region constraints.

We finally compute the actual value for each scc, i.e. all locations, universal regions, and placeholders contained in the scc.

We then map the nll vars of the member constraints to their sccs.

Finally, we add the requirements of all free and placeholder regions. Free regions `X` have to be live at all locations in the body as well as the pseudo-location `end(X)`. We still rely on `liveness_constraints` for diagnostics, so we also consider sccs containing free regions to always be live.

#### `RegionInferenceContext::solve`

We now take these initial liveness constraints for each sccs and progate them in `RegionInferenceContext::propagate_constraints`. As this is a non-cyclic graph, this is a simple walk. We start with the largest scc which outlives every other scc: given `A: B` we first check `B`, then `A`.

In `RegionInferenceContext::compute_value_for_scc` we first propagate the constraints from all sccs outlived by the current one.

We then apply all member constraints `'member_region in choice_regions` where `'member_region` is part of this scc via `RegionInferenceContext::apply_member_constraint`. This does not check the member constraint itself, but in case there is a unique choice we mutate the constraint graph by equating `'member_region` with that lifetime. More on that separately.

For such region, we then add a the constraint `'member_region: 'c`. If this modified the graph, we push it to the `member_constraints_applied` list. This is only used to build a constraint path for diagnostics.

---

Now that we've built the region constraint graph, we continue with `RegionInferenceContext::check_type_tests`. These `type_tests` are the parts of `TypeOutlives` requirements we were unable to destructure into region constraints. They often require disjunction. Because of this, we prove `type_tests` using the new frozen region constraint graph.

They are proven using `RegionInferenceContext::eval_verify_bound`. If that fails inside of a closure, we try call `RegionInferenceContext::try_promote_type_test`, see [the dev-guide][closure-constraints]. If that also fails, we error.

---

`RegionInferenceContext::check_universal_regions` makes sure that region inference didn't add any additional outlives bounds between free regions. This also checks that placeholders were not unified with any other regions. For each universal region this checks all other universals it is required to outlive. If it is not implied by the `universal_region_relations`, we again try to promote this constraint to the parent as explained in [the dev-guide][closure-constraints].

[closure-constraints]: https://rustc-dev-guide.rust-lang.org/borrow_check/region_inference/closure_constraints.html

---

`RegionInferenceContext::check_member_constraints` checks that all the member constraints hold. This is simpler than `RegionInferenceContext::apply_member_constraint` as it now simply asserts that the member region is equal to one of the choice regions.

With this we've now finished computing regions.

#### `RegionInferenceContext::infer_opaque_types`

This happens once we've computed the region graph after type checking. For every opaque type in the storage, we first compute the `arg_regions` of the opaque. These are all regions contained in its (captured) arguments and `'static`.

It then maps all regions in the hidden type to these `arg_regions`, replacing them with `'erased` if they aren't present. This is correct as we've already checked the member constraints.

We then call `fn infer_opaque_definition_from_instantiation`, mapping the generic parameters used in the defining scope to the parameters of the opaque. 

## region assumptions

Region constraints are stored in the `UniversalRegionRelations` in borrowck and the `FreeRegionMap` during lexical region resolution.

Type outlives are stored both in `region_bound_pairs` and in `known_type_outlives_obligations`.

The `region_bound_pairs` only contain already destructured type outlives only added in `UniversalRegionRelationsBuilder::add_outlives_bound` and `OutlivesEnvironment::with_bounds`. It is only ever used for type outlives constraints from implied bounds.

The `known_type_outlives_obligations` only contain the explicit outlives constraints from the `param_env`. We must not desugar outlives bounds from the environment as otherwise `where &'a &'b: 'c` would imply `'b: 'a` without this getting checked by the user.

## member constraints

Given a member constraint `'m in ['c0, 'c1, 'static]` we have to equate `'m` with one of the choice regions as this is necessary for us to name `'m` in the final `type_of` the opaque. While we could avoid equality by introducing `lub` regions, all regions used by the opaque must be uniquely constrained by its params.

We want to add the weakest constraints necessary to avoid unnecessary errors. We may only add additional outlives constraints, discarding existing ones is unsound.

The choice region will have to be a free region. This means the only additional errors we can get from using a suboptimal one are due to missing `universal_region_relations`.

We filter the choice regions `'c` to regions which are larger than `'m`. For all universal regions `'lower` with `'m: 'lower`, we require the assumption `'c: 'lower`. Equating `'m` with an already smaller universal region does not work.

We also have to make sure that the equality constraints on `'m` does not introduce any new outlives constraints between free regions. For everything that's currently larger than `'m`, make sure it's also larger than the `'c`. For all free regions `'upper` with `'upper: 'm`, we require the assumption `'upper: 'c`. We later constraining `'m: 'c` each `'upper: 'm` results in a transitive `'upper: 'c` bound.

We then take use the smallest unique choice region `'c`.
