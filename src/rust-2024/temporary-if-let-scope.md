# `if let` temporary scope

🚧 The 2024 Edition has not yet been released and hence this section is still "under construction".
More information may be found in the tracking issue at <https://github.com/rust-lang/rust/issues/124085>.

## Summary

- In an `if let $pat = $expr { .. } else { .. }` expression, the temporary values generated from evaluating `$expr` will be dropped before the program enters the `else` branch instead of after.

## Details

The 2024 Edition changes the drop scope of [temporary values] in the scrutinee[^scrutinee] of an `if let` expression. This is intended to help reduce the potentially unexpected behavior involved with the temporary living for too long.

Before 2024, the temporaries could be extended beyond the `if let` expression itself. For example:

```rust,edition2021
// Before 2024
# use std::sync::RwLock;

fn f(value: &RwLock<Option<bool>>) {
    if let Some(x) = *value.read().unwrap() {
        println!("value is {x}");
    } else {
        let mut v = value.write().unwrap();
        if v.is_none() {
            *v = Some(true);
        }
    }
    // <--- Read lock is dropped here in 2021
}
```

In this example, the temporary read lock generated by the call to `value.read()` will not be dropped until after the `if let` expression (that is, after the `else` block). In the case where the `else` block is executed, this causes a deadlock when it attempts to acquire a write lock.

The 2024 Edition shortens the lifetime of the temporaries to the point where the then-block is completely evaluated or the program control enters the `else` block.

<!-- TODO: edition2024 -->
```rust
// Starting with 2024
# use std::sync::RwLock;

fn f(value: &RwLock<Option<bool>>) {
    if let Some(x) = *value.read().unwrap() {
        println!("value is {x}");
    }
    // <--- Read lock is dropped here in 2024
    else {
        let mut s = value.write().unwrap();
        if s.is_none() {
            *s = Some(true);
        }
    }
}
```

See the [temporary scope rules] for more information about how temporary scopes are extended. See the [tail expression temporary scope] chapter for a similar change made to tail expressions.

[^scrutinee]: The [scrutinee] is the expression being matched on in the `if let` expression.

[scrutinee]: ../../reference/glossary.html#scrutinee
[temporary values]: ../../reference/expressions.html#temporaries
[temporary scope rules]: ../../reference/destructors.html#temporary-scopes
[tail expression temporary scope]: temporary-tail-expr-scope.md

## Migration

It is always safe to rewrite `if let` with a `match`. The temporaries of the `match` scrutinee are extended past the end of the `match` expression (typically to the end of the statement), which is the same as the 2021 behavior of `if let`.

The [`if_let_rescope`] lint suggests a fix when a lifetime issue arises due to this change or the lint detects that a temporary value with a custom, non-trivial `Drop` destructor is generated from the scrutinee of the `if let`. For instance, the earlier example may be rewritten into the following when the suggestion from `cargo fix` is accepted:

```rust
# use std::sync::RwLock;
fn f(value: &RwLock<Option<bool>>) {
    match *value.read().unwrap() {
        Some(x) => {
            println!("value is {x}");
        }
        _ => {
            let mut s = value.write().unwrap();
            if s.is_none() {
                *s = Some(true);
            }
        }
    }
    // <--- Read lock is dropped here in both 2021 and 2024
}
```

In this particular example, that's probably not what you want due to the aforementioned deadlock! However, some scenarios may be assuming that the temporaries are held past the `else` clause, in which case you may want to retain the old behavior.

The `if_let_rescope` lint cannot deduce with complete confidence that the program semantics are preserved when the lifetime of such temporary values are shortened. For this reason, the suggestion from this lint is *not* automatically applied when running `cargo fix --edition`. It is recommended to manually inspect the warnings emitted when running `cargo fix --edition` and determine whether or not you need to apply the suggestion.

If you want to manually inspect these warnings without performing the edition migration, you can enable the lint with:

```rust
// Add this to the root of your crate to do a manual migration.
#![warn(if_let_rescope)]
```

[`if_let_rescope`]: ../../rustc/lints/listing/allowed-by-default.html#if-let-rescope
