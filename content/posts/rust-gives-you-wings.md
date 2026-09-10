+++
date = '2026-09-10T19:32:37+05:30'
draft = false
title = 'Rust give you wings'
slug = 'rust-gives-you-wings'
+++

A week after finishing the [triage tool in Go]({{< ref "cluster-triage" >}}), I wrote it again in Rust.

Not as a port. I did not open the Go source while writing the Rust. The question I wanted answered was which parts of that design were about the problem and which parts were about Go, and the only honest way to find out is to solve it twice and see what moves.

The short answer: the data structure survived intact, and everything I had been holding true by hand turned into a type.

## Why bother

I had already argued myself out of Rust for the original, and with a deadline attached that was the right call. Kubernetes is Go, the ecosystem I needed to read while working out what a real event body looks like is Go, and the problem has no shared mutable state, no lifetime puzzle and nothing running hot.

I want to say up front where this ends, because the ending changed my mind. Having written both, the one I would rather own is the Rust one.

So this was not a language argument. It was a specific question left open at the end of the first build.

The Go design document ends with one admission: the taxonomy is a table of structs with string fields, where an enum would have the compiler check exhaustiveness. I wrote a property test to cover it and noted that this "spends a test where another language spends a keyword".

I wanted to know what else was in that category. Not what Rust would make *faster*, which is nothing here, but what it would make *unnecessary*. Every invariant I was maintaining with discipline, a comment, or a test, is a candidate. Writing it again is the only way to enumerate them, because from inside one implementation they are invisible. They look like the job.

## What survived

The Forest came through unchanged, which is how I know it was about the problem.

It is still one integer per finding, still sorted by first occurrence, still built by scanning backwards so that every parent sits at a lower index than its child. Cycles are still unrepresentable rather than checked. The findings are still topologically ordered because time ordering *is* the topological order.

None of that is a Go idea or a Rust idea. It is a consequence of what causality looks like when you refuse to express joint causes, and it would look the same in any language with an array.

The four rejected alternatives stayed rejected for the same reasons. Union-find still loses because path compression deletes the intermediate hops that are the actual product.

That was worth learning on its own. If the core structure had needed rethinking, the first design was language-shaped in a way I had not noticed.

## What turned into types

Three things. Each one was a rule in Go and is a signature in Rust.

### The sentinel

Go used `Parent == -1` to mean nothing explains this finding. That is a magic number every reader has to learn and every consumer has to remember to check.

```rust
pub type Node<T> = (T, Option<Edge>);
```

The compiler will not let you read the parent without handling the case where there is none. Nothing clever, and it removes a whole category of "did I check for the sentinel here" from review.

### The parallel arrays

In Go, `Findings` and `Edges` have to be the same length and the same order, and nothing enforces it. I mitigated it: only one function constructs the pair, and both slices live in a struct so they travel together. The failure mode if that discipline slipped would be a **wrong diagnosis**, never a crash, which is the worst kind of bug to ship.

The Rust constructor takes them already paired and unzips them itself:

```rust
/// Takes findings already paired with their edges, so the two can never
/// disagree in length.
pub fn new(nodes: Vec<Node<T>>) -> Self {
    let (findings, edges) = nodes.into_iter().unzip();
    Self::from_parts(findings, edges)
}
```

You cannot hand it a mismatched pair, because there is no way to express one. The convention became unrepresentable rather than documented.

### The field that could drift

Go's edge carried a `Kind` alongside its evidence. Two fields that must agree.

```rust
pub struct Edge {
    /// Indexes the forest's findings. Invariant: less than the edge's own index.
    pub parent_idx: usize,
    /// The rule that fired. `kind` is derived from it rather than stored, so an
    /// edge cannot contradict the rule that produced it.
    pub rule: CausalRule,
}
```

That doc comment is the whole argument. Two fields that must agree will eventually disagree. One field and a function cannot.

> 💡 **The same move, twice**
>
> The Go version already does this elsewhere: a verdict is a method over its four suppression clauses rather than a field beside them, forced by a test that failed against my own fixtures when the two were separate. Rust did not teach me the principle. It made the compiler enforce it in a place I had not thought to apply it.

## The keyword I finally got to spend

The causal rules are an enum:

```rust
pub enum CausalRule {
    /// Object identity: both findings name the same pod.
    SamePod,
    /// The parent is a node condition and the child's record text names it.
    NodeConditionNamed,
    /// Every pod the child names belongs to the replica set the parent rolled out.
    RolloutReplicaSet,
    /// A node condition on the child's node, but the child's record does not name it.
    NodeConditionUnnamed,
}
```

Whether an edge is proven or merely speculative is an exhaustive match over that. Add a fifth rule and the compiler names the one place you forgot to decide whether it proves anything.

Here is the part I want to be honest about, because the tidy version of this story is wrong.

The taxonomy is **still** a table of `&'static str` in Rust. Reasons arrive off the wire as strings, and turning a hundred wire strings into variants buys nothing at the boundary. The thing I said Go was missing is still missing, because it turns out I was wrong about where the problem was.

What actually changed is the lookup:

```rust
pub fn get_classification_rules(reason: &str) -> Option<&'static [ClassificationRule]>
```

The Go version needed a property test to prove classification could never fall through. The Rust version returns an `Option`, so falling through is a case the caller cannot skip. Same guarantee, moved out of a test I had to think of and into a signature I could not avoid.

Rust did not delete the string table. It deleted the test that was compensating for it. That is a smaller claim than "sum types fixed the taxonomy", and it is the true one.

## The seams

The other real divergence is the stage boundaries.

Go's five stages are packages with consumer-declared interfaces, which is idiomatic and works fine. Rust's are traits, and the pipeline is generic over all of them:

```rust
pub trait Classifier {
    type Item;
    type Classification: Classification;
    type HashKey: Hash + Eq;
    fn classify(&self, item: Self::Item) -> ClassifiedEvent<Self::Item, Self::Classification>;
    fn group_key(classified: &ClassifiedEvent<Self::Item, Self::Classification>) -> Self::HashKey;
}

pub trait Aggregator<ClassificationEngine: Classifier> { type Observation; /* ... */ }
pub trait Resolver<Observation> { type Correlations; /* ... */ }
pub trait Analyzer<Correlations> { type Analysis; /* ... */ }
```

Each stage names the type it produces and the next is generic over it, so the compiler chains them. You cannot wire an aggregator to a resolver expecting a different observation type.

Whether that earns its weight at this size is arguable, and I will argue the other side. Four traits and five type parameters is a lot of machinery for a program with one code path. The Go version wires the same five stages together in a function and is easier to read cold. What the Rust version buys is that the seams are checked rather than agreed, and the seams are exactly where a second implementation of any stage would go.

One comment in the aggregator is a correctness note rather than a style one, and it applies to both languages:

> Must return a deterministic order: resolvers correlate observations by position, so an order taken from hash-map iteration would change the conclusions drawn, not just the presentation.

Go randomises map iteration deliberately, so this bites loudly and early. Rust's `HashMap` is also unordered. Same trap, both languages, and the symptom is a different diagnosis on every run until you notice.

## The measurement

| | Source | Tests | Ratio |
|---|---|---|---|
| Go | 4,326 | 4,283 | 1 to 1 |
| Rust | 6,230 | 1,048 | 6 to 1 |

The Rust version is 44% more source and a quarter of the tests. I did not plan that and only noticed it while writing this, and it is the clearest single number in the whole exercise.

Some of the extra source is real ceremony: trait definitions, associated types, `impl` blocks that Go gets for free with structural interfaces. Some of it is the type system absorbing work that was previously spread thin.

The missing tests are not missing coverage. They are tests I never had to write, because the property they would assert is now a thing that does not compile. A meaningful fraction of the Go suite exists to prove that two parallel things stayed parallel, that a sentinel was checked, that a table has no fall-through. Every one of those is a test standing in for a type.

## Design the types and the code mostly writes itself

There is a bigger claim underneath everything above, and it is the one I actually believe.

Almost all of the difficulty in a program like this is in the data model. Once the model is right, the implementation is close to determined. Writing the Rust version felt less like authoring code and more like being walked through the consequences of decisions I had already made. The compiler kept naming the next thing that did not follow yet, I fixed it, and eventually there was nothing left to name and the program worked.

That is not a Rust superpower. It is what happens whenever a data model is precise. What Rust does is make the precision expressible, and then refuse to let you drift from it.

That second half is where the shortlist gets very short. To model a domain properly you need a specific set of tools, and they have to be present together:

- **Sum types with exhaustive matching**, so "one of these four things" is a type rather than a convention plus a test.
- **No null**, so absence is a case you handle rather than a landmine you remember.
- **Cheap newtypes**, so a node name and a namespace stop being interchangeable strings.
- **Ownership and moves**, so the lifecycle of a value is in the signature rather than in a comment about who frees it.
- **Traits over inheritance**, so a type declares what it can do without inheriting what it should not.
- **Constructors you can make the only door**, so an invariant established once cannot be sidestepped later.

Plenty of languages have some of that. The ML family has most of it and has had it for decades, and the reason I am not writing this in OCaml is reach rather than merit. Kotlin and Swift have sum types and real nullability, and both are largely bound to one platform. Go, Java, Python and TypeScript are missing the first item outright, which is the one the rest hangs from.

Rust is the only language in wide production use that hands you the whole set at once, on any platform, with no runtime to carry. That is the actual argument for it here, and it has nothing to do with speed or memory safety.

Design the types well and Rust becomes tedious in the best way. Design them badly and it will fight you at every line, which is also correct behavior, and the most common reason people bounce off the language is that they are feeling their data model being rejected and reading it as the borrow checker being difficult.

## Which one I would keep

The Rust one. That is not where I expected to land, and it is worth being precise about why, because none of the usual reasons apply.

Not speed. It is a 130 millisecond batch job in both languages and neither number matters.

Not safety in the memory sense. There is one thread, no aliasing, and nothing unsafe to get wrong. Borrow checking earned nothing here and I said so before I started.

Not the ecosystem, which still favors Go and always will for anything Kubernetes-shaped. If this grew into something that talks to the API server tomorrow, that argument would win again and I would go back.

What tipped it is narrower than any of those. In the Go version, a specific set of facts were true because I kept them true. The findings and edges stay parallel. The sentinel gets checked. The kind agrees with the rule that produced it. The taxonomy has no fall-through. Every one of those is correct in the Go code, and every one is correct because I was careful, reviewed it, and wrote a test standing guard over it.

In the Rust version most of them are not facts I maintain. They are shapes the compiler will not let me express wrongly. That difference costs 44% more source and buys back three quarters of the test suite, and the part I actually care about is not the line count. It is that the failure mode I was most afraid of, a **wrong diagnosis rather than a crash**, is the exact failure mode that class of invariant produces when it slips.

A tool nobody trusts at 3am is worse than no tool. Discipline is how you get correctness in Go, and my discipline is fine, but it is a thing that has to hold every time I come back to this after six months away. The Rust version does not ask me to remember.

So: Go was the right decision for the brief, and I would make it again under the same constraints. Rust is the version I would keep, extend, and hand to someone else.

The exercise also answered the question I actually started with. The Forest was about the problem and survived untouched. The invariants around it were about Go, and every one of them turned into either a type or nothing at all.

Writing the same thing twice is expensive and I would not do it often. But it is the only way I have found to tell which of your careful decisions hold weight and which are scaffolding you built because the compiler would not.
