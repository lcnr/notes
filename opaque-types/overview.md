# Opaque types and `-Znext-solver`

Tests which have to compile
```rust
trait Id: Sized {
    fn id(self) -> Self { self }
}
impl<T> Id for T {}
fn recur(b: bool) -> impl Id {
    if b {
        // Method resolution happens while
        // we don't yet know the hidden type
        // of `recur`.
        recur(false).id()
    } else {
        1u32
    }
}
```

```rust
trait Id: Sized {
    fn id(self) -> Self { self }
}
impl<T> Id for T {}
fn recur_in_closure() -> impl Id {
    let _ = || recur_in_closure().id();
    1u32
}
```

```rust
trait Id: Sized {
    fn id(self) -> Self { self }
}
impl<T> Id for T {}
fn recur_in_closure<'a>() -> impl Id + 'a {
    #[cfg(explicit)]
    let _ = || recur_in_closure::<'a>().id();
    #[cfg(unconstrained)]
    let _ = || recur_in_closure().id();
    1u32
}
```

## Goals

- nested bodies may define opaque types
- the root is the only thing checked by `type_of_opaque`
- figure out how `recur().alias_bound()` should work in the new solver

## Current behavior

- the opaque type gets defined in the nested body
- the defining use in the closure must not use external regions as we don't know their relationships which breaks member constraints 
- member constraints?

## Related refactors

- eagerly normalize opaques in the old solver as well
    - https://github.com/rust-lang/rust/pull/120798
        - blocked on `recur().alias_bound()`
- merge `TypeVerifier` and `TypeChecker`
- figure out why the canonical closure signature is not just a user type annotation.
- eagerly create nll vars for late bound and then ICE on failing lookup
- remove `struct OpaqueTypeDecl`

## Links

- aliemjay min-choice PR: https://github.com/rust-lang/rust/pull/105300/
- considering liveness for opaque-type captures: https://github.com/rust-lang/rust/pull/116040