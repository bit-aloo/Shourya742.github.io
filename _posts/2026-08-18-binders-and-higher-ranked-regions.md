---
layout: "post"
title:  "Binders and higher ranked regions"
date:   "2052-04-30 14:23:35 +0530"
categories: rust
--- 

## Binder and Higher ranked regions

Sometimes, we define generic parameters not on an item but as part of a type or a where clause. As an example, the type `for<'a> fn(&'a u32)` or the where clause `for<'a> T: Trait<'a>` both introduce a generic lifetime named `'a`. Currently, there is no stable syntax for `for<T>` or `for<const N: usize>`, but on nightly `feature(non-lifetime-binders)` can be used to write where clauses(but not types) using `for<T>/for<const N: usize>`.

The `for` is referred to as a "binder" because it brings new names into scope. In rustc we use the `Binder` type to track where these parameters are introduced and what the parameters are (i.e, how many and whether the parameter is a type/const/region). A type such as `for<'a> fn(&'a u32)` would be represented in rustc as:

```rust
Binder (
    fn(&RegionKind::Bound(DebruijnIndex(0), BoundVar(0)), u32) -> (),
    $[BoundVariableKind::Region(...)]
)
```

Usages of these parameters is represented by the `RegionKind::Bound` (or `TyKind::Bound/ ConstKind::Bound` variants). These bound regions/types/const are composed of two main pieces of data:

* A DebruijnIndex to specify which binder we are referring to.
* A `BoundVar` which specifies which of the parameters that the `Binder` introduces we are referring to.

We also sometimes store some extra information for diagnostics reasons via the `BoundTypeKind/BoundRegionKind`, but this is not important for type equality, or, more generally, the semantics of `Ty`. (omitted from the above example).

In debug output (and  also informally when talking to each other), we tend to write these bound variables in the format of `^DebruijnIndex_BoundVar`. The above example would instead be written as `Binder(fn(&'^0_0), &[BoundVariableKind::Region])`. Sometimes, when the `DebruijnIndex`is `0`, we just omit it and would write `^0`.

Another concrete example, this time a mixture of `for<'a>` in a where clause and a Type:

```rust
where 
    for<'a> Foo<for<'b> fn(&'a &'b T)>: Trait
```

This would be represented as:

```rust
Binder(
    Foo<Binder {
        fn(&^1_0 &'^0 T/#0),
        [BoundVariableKind::Region(...)]
    }>: Trait,
    [BoundVariableKind::Region(...)]
)
```

Note how the `'^1_0` refers to the `'a` parameter, We use a `DebruignIndex` of `1` to refer to the binder one level up from the innermost one, and a var of `0` to refer to the first parameter bound which is `'a`. We also use `'^0` to refer to the `'b` parameter, the `DebruijnIndex` is `0` (referring to the innermost binder) so we omit it, leaving only the boundvar of `0` referring to the first parameter bound which is `'b`.

We did not always explicitly track the set of bound vars introduced by each `Binder`, and this caused a number of bugs. By tracking these explicitly, we can assert when constructing higher ranked where clauses/types that there are no escaping bound variables or variables from a different binder. See the following example of an invalid type inside of a binder:

```rust
Binder (
    fn(&`^1_0 &`1 T/#0),
    &[BoundVariableKind::Region(...)]
)
```

This would cause all kinds of issues as the region `'^1_0` refers to a binder at a higher level than the outermost binder i.e. it is an escaping bound var. The `'^1` region (also writeable as `'^0_1`) is also ill formed as the binder it refers to does not introduce a second parameter. Modern day rustc will ICE when constructing this binder due to both of those reasons. In the past, we would have simply allowed this to work and then ran into issues in other parts of the codebase. 

## Instantiating Binders

Much like `EarlyBinder`, when accessing the inside of a `Binder`, we must first discharge it by replacing the bound vars with some other value. This is for much the same reason as with `EarlyBinder`, types referencing parameters introduced by the `Binder` do not make any sense outside of the binder. See the following erroring example:

```rust
fn foo<'a>(a: &'a u32) -> &'a u32 {
    a
}

fn bar<T>(a: fn(&u32) -> T) -> T {
    a(&10)
}

fn main() {
    let higher_ranked_fn_ptr = foo as for<'a> fn(&'a u32) -> &'a u32;

    // Attempt to infer `T=for<'a> &'a u32` which is not satisfiable
    let references_bound_vars = bar(higher_ranked_fn_ptr);
}
```

In this example, we are providing an argument of type `for<'a> fn(&'^0 u32) -> &'^0 u32` to `bar`. We do not want to allow `T` to be inferred to the type `&'^0 u32` as it would be rather nonsensical and likely unsound if we did not happen to ICE. `main` doesn't know about `'a` so the borrow checker would not be able to handle a borrow with lifetime `'a`.

Unlike `EarlyBinder` we typically do not instantiate `Binder` with some concrete set of arguments from the user, i.e, `['b, 'static]` as arguments to a `for<'a1, 'a2> fn(&'a1 u32, &'a2 u32)`. Instead we usually instantiate the binder with inference variables or placeholders.

## Instantiating with inference variables

We instantiate binders with inference variable when we are trying to infer a possible instantiation of the binder, e.g. calling higher ranked function pointers or attempting to use a higer ranked where clause to prove some bound. For example, given the `higher_ranked_fn_ptr` from the example above, if we were to call it with `&10_u32` we would:

* Instantiate the binder with infer vars yielding a signature of `fn(&'?0 u32) -> &'?0 u32`.
* Equate the type of the provided argument argument `&10_u32` (&'static u32) with the type in the signature, `&'0 u32`, inferring `'?0` = `'static`.
* The provided arguments were correct as we were successfully able to unify the types of the provided arguments with the types of the arguments in fn ptr signature.

As another example of instantiating with infer vars, given some `for<'a> T: Trait<'a>`  where clause, if we were attempting to prove that `T: Trait<'static>` holds we would:

* Instantiate the binder with infer vars yielding a where clause of `T: Trait<'?0>`
* Equate the goal of `T: Trait<'static>` with the instantiated where clause, inferring `'?0 = 'static`.
* The goal holds because we were successfully able to unify `T: Trait<'static>` with `T: Trait<'?0>`

Instantiating binders with inference variables can be accomplished by using the `instantiate_binder_with_fresh_vars` method on `inferCtxt`. Binders should be instantiated with infer vars when we only care about one specific instantiation of the binder, if instead we wish to reason about all possible instantiations of the binder then placeholders should be used instead.

## Instantiating with placeholders

Placeholders are very similar to `Ty/ConstKind::Param/ReEarlyParam`, they represent some unknown type that is only equal to itself. `Ty/Const` and `Region` all have a Placeholder variant that is comprised of a `Universe` and a `BoundVar`.

The `Universe` tracks which binder the placeholder originated from, and the `BoundVar` tracks which parameter on said binder that this placeholder corresponds to. Equality of placeholders is determined solely by whether the universes are equal and the `BoundVar`s are equal.

When talking with other rustc devs or seeing `Debug` formatted `Ty/Const/Region`s, `Placeholder` will often be written as `!UNIVERSE_BOUNDVARS`. For example, given some type `for<'a> fn(&'a u32, for<'b> fn(&'b &'a u32))`, after instantiating both binders (assuming the `Universe` in the current `InferCtxt` was `U0` beforehand), the type of `&'b &'a u32` would be represented as `&'!2_0 &1_0 u32`.

When the universe of the placeholder is `0`, it will be entirely omitted from the debug output i.e `!0_2` would be printed as `!2`. This rarely happens in practice though as we increase the universe in the `InferCtxt` when instantiating a binder with placeholders, so usually the lowest universe placeholders encounterable are ones in `U1`.

`Binder`s can be instantiated with placeholders via the `enter_forall` method on `inferCtxt`. It should be used whenever the compiler should care about any possible instantiation of the binder instead of one concrete instantiation.

## Why have both RePlaceholder and Rebound?

You may be wondering why we have both of these variants, afterall the data stored in Placeholder is effectively equivalent to that of `ReBound`: something to track which binder and an index to track which parameter the `Binder` introduced.

The main reason for this is that `Bound` is a more syntactic representation of bound variables whereas Placeholder is a more semantic representation. As a concrete example:

```rust

```