# Method selection on opaque types

We want to support method selection on opaque types in their defining scope while before their defining use.
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
We structurally resolve opaque types while computing the `method_autoderef_steps` and for method resolutions.

We have to error 