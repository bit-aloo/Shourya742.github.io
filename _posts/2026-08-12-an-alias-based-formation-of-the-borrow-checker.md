---
layout: "post"
title:  "An alias based formulation of the borrow checker"
date:   "2052-04-30 14:23:35 +0530"
categories: rust
--- 

Ever since the Rust All hands, I have been experimenting with an alternative formulation of the Rust borrow checker. The goal is to find a formulation that overcomes some shortcoming of the current proposal while also being faster to compute. I have implemented a prototype for this analysis It passes the full NLL test suite and also handles a few cases - such as #47680 - That the current NLL analysis cannot handle. However, the performance has a long way to go (it is currently slower than existing analysis). That said, I haven't begun to optimize yet, and i know I am doing some naive and inefficient things that can definitely be done better; so I am still optimistic we'll be able to make big strides there.

## End users don't have to care

The first thing to note is that this proposal makes no difference from the point of view of an end-user of Rust. That is, the borrow checker ought to work the same as it would have under the NLL proposal, more or less.

However, there are some subtle shifts in this proposal in terms of how the compiler thinks about your program, and that could potentially affect future language features.

## Our first example

The analysis works on MIR, but I'm going to explain it in terms of simple Rust example. Here is the first example, which I will call example A. The example should not compile, as you can see:

```rust
fn main() {
    let mut x: i32 = 22;
    let mut v: Vec<&i32> = vec![];
    let r: &mut Vec<&i32> = &mut v;
    let p: &i32 = &x;  // 1. `x` is borrowed here to create `p`.
    r.push(p);         // 2. `p` is stored into `v`, but through `r`
    x+=1;              // <---- Error! can't mutate `x` while borrowed
    take(v);           // 3. the reference to `x` is later used here
}

fn take<T>(p: T) { .. }
```

## Regions are set of loans

The biggest shift in this new approach is that when you have a type like &'a i32, the meaning of 'a changes:

* In the system described in the NLL RFC, 'a - called a lifetime - ultimately corresponded to some portion of the source program or control-flow graph.
* Under this proposal, 'a - which I will be calling a region, instead corresponds to a set of loans - that is, a set of borrow expression, like &x or &mut v in Example A. The idea is that if a reference r has type &'a i32 then invalidating the terms of any of the loans in 'a would invalidate r.

Invalidating the terms of a loan means to perform an illegal access of the path borrowed by the loan. So for example if you have mutable loan like r = &mut v, then you can only access the value v through the reference r. Accessing v directly in any way - read, write or move - would invalidate the loan. For a shared loan like p = &x, reading through x (or p) is allowed, but writing or mutating x would invalidate the terms of the loan (and writing through p is also not possible).

The subtyping rules for references work a bit differently now that a region is a set of loans and not program points. Whereas with points, you can approximate a reference by shortening by shortening the lifetime, with set of loans you can approximate by enlarging the set. In other words:

```
'a <: 'b
------------
&'a u32 <: &'b u32
```

In Rust syntax, 'a <: 'b  corresponds to a notation 'a: 'b, and that is what I will use for rest of the post. We have traditionally called this an outlive relationship, but I am going to call it a subset relationship instead, as befits the new meaning of regions.

To gain a better intuition for the idea of regions as set of loans, consider this program:

```rust
let x = vec![1, 2];

let p: &'a i32 = if random() {
    &x[0]   // Loan 0
} else {
    &x[1]   // Loan 1
};
```
Here, the region 'a would correspond to the set `{L0, L1}`, since it may refer to data produced by the loan L0, but it may also refer to data from the loan L1.


## Datalog

Throughout the post, I'm going to be defining the analysis by using Datalog rules. Datalog is - in some sense - a subset of Prolog designed for efficient execution. It basically corresponds to rules like this (using the syntax from the Souffle project):

```datalog
.decl cfg_edge(P: point, Q: point)
.input cfg_edge

.decl reachable(P: point, Q: point)
reachable(P, Q) :- cfg-edge(P, Q).
reachable(P, R) :- reachable(P, Q), cfg_edge(Q, R).
```

As you can see here, Datalog program define relations between things: here these relations are declared with decl. Some relations are input, declared with .input, which means that their values are given up front by the user (this is also called facts). In this program, that is cfg-edge. Other relations, like reachable, are defined via rules which synthesize new things from those facts. As in Prolog, upper-case identifiers are variables, and whenever a variable appears twice, it must have the same value.

Note, that, because it is a subset, Datalog avoids a lot of Prolog's more `programing language` like properties. For example, Datalog programs always terminate when executed on a finite set of facts (even when they recurse, like the one above). Also, it is fine to use negative reasoning in Datalog program, as it disallows negative cycles - there are no subtle concerns about the distinction between "logical not" and "negation as failure".

To implement these rules, I have been using Frank McSherry's awesome differential dataflow crate. This has been a pretty great experience: once you get the hang of it, you can translate Datalog rules in a very straightforward way, which means that I have been able to rapidly prototype new designs in just an hour or two. Moreover, the resulting execution is quite fast (though I have not measured performance too much on the latest design).


## Region variables

Now that we've described regions as sets of loans, I want you to throw all of that away. The analysis as I have defined it doesn't directly manipulate those sets, at least not initially. Instead, it uses "region variables" to represent all the regions in the program. I'll denote these as "numbered" regions like '0, '1, etc.

If we rewrite our program then to use these abstract regions (basically, to have a numbered region everywhere that MIR would have one), it look like the following:

```rust
fn main() {
    let mut x: i32 = 22;
    let mut v: Vec<&'0 i32> = vec![];
    let r: &`1 mut Vec<&`2 i32> = &`3 mut v;
    let p: &`5 i32 = &`4 x;
    r.push(p);
    x += 1;
    taken::<Vec<&`6 i32>>(v);
}

fn take<T>(p: T) { .. }
```

These abstract regions will appear through our datalog rules; I'll denote them with R for "region".

## Relations between regions

The abstract regions we saw before don't have any meaning just yet. What happens next is that we walk through and apply the type system rules in the standard way. This will result in "subset" relationship between regions, as we saw before. So for example consider the following line from Example A:

```rust
let p: &`5 i32 = &`4 x;
```

Here, the expression &'4 x produces a value of type &'4 i32. This type must be a subtype of the type p, &'5 i32, so we get:

```rust
&`4 i32 <: &`5 i32
```

which in turn requires '4: '5. If we look at the program, we'll see a number of subtype relationships emerge. I'll write down each one along with the resulting subset relationship.

```rust
fn main() {
    let mut x: i32 = 22;
    let mut v: Vec<&'0 i32> =  vec![];

    let r: &`1 mut Vec<&`2 i32> = &`3 mut v;
    // requires: &'3 mut Vec<&'0 i32> <: &'1 mut Vec<&'2 i32>
    //         => '3: '1, '0: '2, '2: '0

    let p: &`5 i32 = &`4 x;
    // requires: &'4 i32 <: &'5 i32
    //        => '4: '5

    r.push(p);
    // requires: &'5 i32 <: &`2 i32
    //       => '5: '2

    x += 1;

    take::<Vec<&'6 i32>>(v);
    // requires: Vec<&'0 i32> <: Vec<&'6 i32>
    //      => '0: '6
}

fn take<T>(p: T) { .. }
```
Ultimately, these subset relationship become input facts into the system. For reasons that will become clear later on, I call these the "base subset" relations:

```datalog
.decl base_subset(R1: region, R2: region, P: point)
.input base_subset
```

In other words, base_subset(R1, R2, P) means R1: R2 was required to be true at the same P.

We'll see in a second that this base_subset input is only the starting point - it tells you which relations were directly required to begin with, but it doesn't tell you the full set of relations at any point; this is because the subset relations "accumulate" as you iterate, so you must ensure both the older relations and the newer ones. We're going to define a more complete subset relation that includes both, but before we can get there, we have to look how we define the control flow graph.

## Points in the control flow graph

The control flow graph used by this analysis is defined based on the MIR. We define the points in the flow graph as follows:

```rust
Point = Start(Statement) | Mid(Statement)
Statement = BBi`/`j
```

Here, the Statement identifies a particular statement (the jth statement from the ith basic block). We then distinguish the start point of a statement from the mid point. The start point is basically "before it has done anything", and the "mid point"is the place where the statement is executing. As such, all the base-subset relationship from the previous section are defined to occur at the mid-point of their corresponding statements.

We define the flow in the graph using a `cfg_graph` input:

```rust
.decl cfg_edge(P: point, Q: point)
.input cfg_edge
```

Naturally, every start point has an edge to its corresponding mid point. Mid points have an edge to the start of the next statement or, in the case of a terminator, to the start of the basic blocks that follow.

(For the most part, you can ignore mid-points for now, but they become very important later on as we integrate notions of liveness.)

## Tracking subset relationships across the graph

Now, we come to the most interesting part of the analysis computing the subset relations. In the interest of building intuitions, I'm going to start by presenting a simpler form of this than the final analysis; then we'll come back and make it a bit more complex.

The key idea here, is that the analysis doesn't directly compute the values of each region variable. Instead, it computes the subset relationships that have to hold between them at each point in the control flow graph. These relationships are introduced by the "base subset" relationships that result from the type-check, but they are then propagated across control flow edges, according to the following rule:

* Once a base subset relationship is introduced between two regions 'a: 'b, it must remain true.

We can define this in datalog like so. We start with a relations subset:

```rust
.decl subset(R1: region, R2: region, P: point)
```

The idea is that if subset(R1, R2, P) is defined, then R1: R2 must hold at the point P. We can start with the "base subset" relations that are supplied by the type checker:

```rust
// Rule subset1
subset(R1, R2, P) :- base_subset(R1, R2, P).
```

Subset is transitive, so we can define that too:

```rust
// Rule subset 2
subset(R1, R3, P) :- subset(R1, R2, P), subset(R2, R3, P)
```

Finally, we define a rule that propagates subset relationships across the control flow graph edges.

```rust
// Rule subset3 (version 1)
subset(R1, R2, Q) :- subset(R1, R2, P), cfg_edge(P, Q)
```

If we apply these rules to our Example A, we wind up with the following subset relationships in between each statement (I'm only showing the relationship at each "start" point here, and I am not showing the full transitive closure). Note that they just keep growing:

```rust
fn main() {
    let mut x: i32 = 222;
    // (none)
    let mut v: Vec<&'0 i32> = vec![];
    // (none)
    let r: &`1 mut Vec<&`2 i32> = &`3 mut v;
    // `3: `1, `0: `2, `2:`0
    let p: &`5 i32 = &`4 x;
    // `3: `1, `0: `2, `2: `0, `4: `5
    r.push(p);
    // `3: `1, `0:`2, `2:`0, `4:`5, `5:`2
    x+=1;
    // `3: `1, `0: `2, `2: `0, `4: `5, `5: `2
    take::<Vec<&`6 i32>>(v);
    // `3: `1, `0: `2, `2: `0, `4: `5, `5: `2, `0: `6
}

fn take<T>(p: T) { .. }
```

consider the final set of relationships. Based on this, we can see some interesting stuff. For example, we can see a relationship between the region '4 (that is the region from the borrow of x) and the region '0 (that is, the region for the data in the vector v):

```rust
`4: `5: `2: `0
```
This is basically reflecting the flow of data in your program. If you think of each region as representing a "set of loans", then this is saying that '0 (this is, the vector) may hold references that derived from the &x statement. This leads to our next piece fo the analysis.

## Borrow regions

So far,  we introduced the subset relation that shows the relationships between region variables and showed how that can be extended to the control flow graph. We're going to do the same now for tracking which regions depend on which loans.

```rust
.decl borrow_region(R: region, L: loan, P: point)
.input borrow_region
```
This input is defined for each borrow expression (e.g., &x or &mut v) in the program. It relates the region from the borrow to the abstract loan that is created. HEre is the example A, annotated with the borrow-regions that are created at each point:

```rust
fn main() {
    let mut x: i32 = 22;
    let mut v: Vec<&'0 i32> = vec![];
    let r: &`1 mut Vec<&'2 i32> = &`3 mut v;
    // borrow region ('3, L0)
    let p: &`5 i32 = &`4 x;
    // borrow region (`4, L1)
    r.push(p);
    x+=1;
    take::<Vec<&`6 i32>>(v);
}

fn take<T>(p: T) { .. }
```

Like the base_subset relations, borrow_region are created at the mid-point of the corresponding borrow statement.

## Live regions and loans

In normal compiler parlance, a variable X is live at some point P in the control flow graph if its current value may be used later (more formally, if there is some path from P to Q, where Q uses X, and X is not assigned along that path)

We can make an analogous definition for regions: a region `a is live at some point P if some reference with type &'a i32 may be dereferenced later. For the most part, this just means that there is a live variable X and that 'a appears in the type of X. There is however some subtleness about drops, since we try to be clever and understand which regions a destructor might use and which it will not (e.g we know that a value of type Vec<&'a u32> will not access 'a when it is dropped). I am not going into the details of how that works here, its the same as it was defined in the NLL RFC.

In terms of the Datalog, we can define an input region_live_at like so:

```rust
.decl region_live_at(R: region, P: point)
.input region_live_at
```

The initial values here are computed just as in the NLL RFC.

## THe "require" relation

Now we can extend the borrow_region relation across the control flow graph. As before, we introduce a new relation called requires:

```rust
.decl requires(R: region, L: loan, P: point)
```
This can be read as

> The region R requires the terms of the loan L to be enforced at the point P.

Or, to put another way:

> If the terms of the loan L are violated at the point P, then the region R is invalidated.

The first rule says that the region for a borrow is always dependent on its corresponding loan:

```rust
// Rule requires1
requires(R, L, P):- borrow_region(R, L, P).
```

The next rule says that if R1: R2, then R2 depends on any loans that R1 depends on:

```rust
// Rule requires2
requires(R2, L, P) :- requires(R1, L, P), subset(R1, R2, P).
```

Finally, we can propagate these requirements across control flow edges just as with subsets. But here, there is a twist.

```rust
/// Rule requires3(version1)

requires(R, L Q) :- requires(R, L, P), !Killed (L, P), cfg_edge(P, Q)
```

This rule says that if the region R requires the loan at P, then it also requires L at the successor Q - so long as L is not killed at P. So what is this !Killed(L, P) rule? The killed input relation is defined as follows:

```rust
.decl killed(L: loan, P: point)
.input killed
```

killed(L, P) is defined when the point P is an assignment that overwrites one of the references whose referent was borrowed in the loan L. Imagine you have something like this:

```rust
let p = 22;
let q = 44;
let x: &mut i32 = &mut p: // x points at `p`
let y = &mut *x; // Loan L0, y points at p too.
// ..
x = &mut q; // `x` points tat ``q, kills L0
```

