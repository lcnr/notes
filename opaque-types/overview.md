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
fn recur(b: bool) -> impl Id {
    if b {
        let delay = recur(false);
        delay.id()
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

```rust
fn define_in_closure<'a>() -> impl Sized + use<'a> {
    let _ = || drop::<u32>(define_in_closure());
    loop {}
}
```

```rust
fn tup_in_closure<'a, 'b>(x: &'a (), y: &'b ()) -> impl Sized {
    let _ = || drop(tup_in_closure(x, y));
    (x, y)
}
```

We can likely break
```rust
trait ImplementedNoDefine: Sized {
    fn method(self) {}
}

impl<T> ImplementedNoDefine for T {
    fn method(self) {}
}

pub fn foo() -> impl Sized {
    if false {
        foo().method()
    }
}
```

We may want to break
```rust
use std::ops::Deref;
struct Foo<T>(T);
impl<T> Deref for Foo<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
impl<T> Foo<T> {
    fn method(&self) {}
}

fn rpit() -> impl Sized {
    if false {
        let x = Foo(rpit());
        x.method()
    }
}
```
and
```rust
trait Trait {
    fn foo(self: &&Self) {}
}

impl Trait for u32 {}

fn define() -> impl Sized {
    if false {
        define().foo(); // constrains `impl Sized` to `&_`
    }
    &1u32
}
```

test-ish
```rust
fn foo() -> impl Trait {
  let a = Default::default();

  if false {
    return a.clone();
  }

  a.foo()
}
```

- we must make sure that the external regions of the opaque are all unique, can't do this inside of the closure

## Goals

- nested bodies may define opaque types
- the root is the only thing checked by `type_of_opaque`
- figure out how `recur().alias_bound()` should work in the new solver
- figure out why borrowck is separate between closures and parent

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
- aliemjay general cleanup: https://github.com/rust-lang/rust/pull/116891
- considering liveness for opaque-type captures: https://github.com/rust-lang/rust/pull/116040
- eagerly normalize opaque types https://github.com/rust-lang/rust/pull/120798