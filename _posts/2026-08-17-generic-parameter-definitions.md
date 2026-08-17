---
layout: "post"
title:  "Generic parameter definitions"
date:   "2052-04-30 14:23:35 +0530"
categories: rust
--- 

# Generic parameter definitions

We will see how rustc tracks what generic parameters are introduced. For example given some `struct Foo<T>` how does rustc track that `Foo` defines some type parameter `T` (and no other generic parameters).

This will not cover how we track generic parameters introduced vai `for<'a>` syntax (eg, in where clause or `fn` types), which is covered when we discuss of binders.

## ty::Generics

The generic parameters introduced by an item are tracked by the `ty::Generics` struct. Sometimes items allow usage of generics defined on parent items, this is accomplished via the `ty::Generics` struct having an optional field to specify a parent item to inherit generic parameters of. For example given the following code:

```rust
trait Trait<T> {
    fn foo<U>(&self);
}
```

The `ty::Generics` used for foo would contain `[U]` and a parent of `Some(Trait)`. `Trait` would have a `ty::Generics` containing `[Self, T]` with a parent of `None`.

The `GenericParamDef` struct is used to represent each individual generic parameter in a `ty::Generics` listing. The `GenericParamDef` struct contains information about the generic parameter, for example its name, defid, what kind of parameter it is (i.e. type, const, lifetime).

`GenericParamDef` also contains a `u32` index representing what position the parameter is (starting from the outermost parent), this is the value used to represent usages of generic parameters.

Interestingly, `ty::Generics` does not currently contain every generic parameter defined on an item. In the case of functions it only contains the early bound parameters.

## EarlyBinder and instantiating parameters

Given an item that introduces a generic parameter `T`, whenever we refer to types inside of `foo` (i.e the return type or argument type) from outside of `foo` we must take care to handle the generic parameters defined on `foo`. As an example:

```rust
fn foo<T, U>(a: T, _b: U) -> T { a }

fn main() {
    let c = foo::<i32, u128>(1, 2);
}
```

When type checking main we cannot just naively look at the return type of `foo` and assign the type `T` to the variable `c`. The function main does not define any generic parameters, `T` is completely meaningless in this context. More generally whenever an item introduces (binds) generic parameters, when accessing types inside the item from outside, the generic parameters must be instantiated with values from the outer item.

In rustc we track this via the `EarlyBinder` type, the return type of `foo` is represented as an `EarlyBinder<Ty>` with the only way to access `Ty` being to provide arguments for any generic parameters `Ty` might be using. This is implemented via the `EarlyBinder::Instantiate` method which discharges the binder returning the inner value with all generic parameters replaced by the provided arguments.

To go back to our example, when type checking `main` the return type of `foo` would be represented as `EarlyBinder(T/#0)`. Then, because we called the function with `i32`, `u128` for the generic arguments, we would call `EarlyBinder::instantiate` on the return type with `[i32, u128]` for the args. This would result in an instantiated return type of `i32` that we can use as the type of the local `c`.

Here are some more examples:

```rust
fn foo<T>() -> Vec<(u32, T)> { Vec::new() }
fn bar() {
    // The return type of `foo` before instantiating it would be:
    // `EarlyBinder(Adt(Vec, &[Tup(&[u32, T/#0])]))`
    // We then instantiate the binder with `[u64]` resulting in the type:
    // `Adt(Vec, &[Tup(&[u32, u64])])`
    let a = foo::<u64>();
}
```

```rust
struct Foo<A, B> {
    x: Vec<A>,
    ...
} 

fn bar(foo: Foo<u32, f32>) {
    // the type of `foo's` `x` field before instantiating it would be:
    // `EarlyBinder(Vec<A/#0>)`
    // We then instantiate the binder with `[u32, f32]` as those are the
    // generic arguments to the `Foo` struct. This results in a type of:
    // `Vec<u32>`
    let y = foo.x;
}
```
In the compiler the instantiate call for this is done with `FieldDef::ty`, at some point during type checking `bar` we will wind up calling `FieldDef::ty(x, &[u32, f32])` in order to obtain the type of `foo.x`.

Notes on indices: It is a bug if the index of a `Param` does not match what the `EarlyBinder` binds. For example, if the index is out of bounds or the index of a lifetime corresponds to a type parameter. These sorts of errors are caught earlier in the compiler during name resolution where we disallow references to generics parameters introduced by items that should not be nameable by the inner item.

As mentioned previously when outside of an item, it is important to instantiate the `EarlyBinder` with generic arguments before accessing the value inside, but the setup for when we are conceptually inside of the binder already is a bit different.

For example:

```rust
impl<T> Trait for Vec<T> {
    fn foo(&self, b: Self) {}
}
```
When constructing a `Ty` to represent `b` parameter's type we need to get the type of `Self` on the impl that we are inside. This can be acquired by calling the `type_of` query with impl's `DefId`, however, this will return a `EarlyBinder<Ty>` as the impl block binds generic parameters that may have to be discharged if we are outside of the impl.

The `EarlyBinder` type provides an `instantiate_identity` function for discharging the binder when you are "already inside of it".  This is effectively a more performant version of writing `EarlyBinder::instantiate(GenericArgs::identity_for_item(..))`. Conceptually this discharges by instantiating it with placeholder in the root universe. In practise though it simply returns the inner value with not modification taking place.