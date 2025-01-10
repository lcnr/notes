# Define in closures

```rust
trait Mirror {
    fn mirror(self) -> Self;
}
impl<T> Mirror for T {
    fn mirror(self) -> Self {
        self
    }
}

struct Inv<'a>(&'a mut &'a ());
fn relate_lts<'a>(x: Inv<'a>, cond: bool) -> impl Mirror + 'a {
    if cond {
        let closure = |x| relate_lts(x, false).mirror();
        closure(x)
    } else {
        x
    }
}
```
The closure desugars to
```rust
fn closure<'aext>(x: Inv<'ext1>) -> Inv<'ext2> {
    let x: Inv<'local1> = relate_tys::<'local2>(x);
    <Inv<'local3> as Mirror>::mirror(x)
}
```
The assignment equates `Opaque<'local2>` with `Inv<'local1>`. After region constraint propagating we get `Opaque<'ext1> = Inv<'ext2>`. The member constraint `'ext2 member ['ext1, 'static]` is ambiguous.

We only know what to do once the caller requires that `'ext1`  and `'ext2` are the same. This requires information from the caller to propagate into the closure.

## Avoiding a fixpoint iterator

This would be a fairly simple solution:

Borrowck of a closure bails after building the constraint graph if there are unsatisfied member constraints. It still returns all the other constraints to the parent.

The parent then uses these constraints and borrowchecks as normal. This ends up computing all the constraints between the external regions of its closures. We then recheck the closure with the external region information from the parent. This time we do not return any closure constraints to the caller and use a post-borrowck typing mode.

This is insufficient in case opaque type normalization results in additional constraints for the caller. If it does, we would be forced to run this to a fixpoint instead.

## Cases where this breaks and implementing a fixpoint algorithm

The closure signature is invariant. Could we store the signature of all closures and then repeatly call `borrowck_typeck_child_with_defining_ty(closure_def_id, current_defining_ty)` until we've reached a fixpoint. This continues to treat all non-universal regions in the parent as external regions and returns their constraints.

I can't think of a concrete test here but expect this to be required for reasonable-ish code.