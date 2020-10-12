<!--
.. title: Theorem proving in Haskell
.. slug: theorem-proving-in-haskell
.. date: 2020-09-20 12:35:13 UTC+01:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

# Theorem proving in Haskell

This article follows on from [the previous one](link://slug/intuitionistic-logic-in-haskell) on intuitionistic logic in Haskell. Unlike that one, this is not cover any theory - instead, it's a technical exploration of a Haskell library that simulates a theorem prover.

## Theorem provers

Since Haskell terms are proofs, we can try to use the Haskell compiler as a **theorem prover** - a program that allows the user to describe a proof as a series of steps, and then checks that the proof is correct. In Haskell's case, we describe proofs as terms, and know they are correct when the term compiles and has the expected type (the statement we are trying to prove).

### Coq

This approach is one taken by many theorem provers: [Coq][2] is a theorem prover with an associated dependently-typed language. Coq's type system is powerful enough to encode much more than just the propositional logic we've seen so far: it can be used to make statements about the behaviour of functions, the existence of certain kinds of values, and even other (quantified) statements.

The Coq compiler is similarly powerful: it is able to enforce that functions are *strictly positive* - the property that gave us terminating Haskell terms - while still allowing for a wide range of expressible terms. The power of Coq is such that you can use it to describe and prove complex statements and theorems: but its basic concepts are very similar to the ones we've seen already. Coq proofs are just code; implication terms are just functions.

### Interactive theorem proving

Coq is designed around *interactive theorem proving* - proofs can be described as a series of steps, and an editor can step through a proof to see the effect of each step in arriving at the final result. Each step can modify the **environment** - the set of variables and facts known to the compiler - and the **goals** - the set of statements which need to be proven. For example, the type `forall A B : Prop, A -> B -> A /\ B` says that, from the hypotheses `A` and `B`, it is possible prove `A /\ B` (this is equivalent to the Haskell type `a -> b -> a /\ b`). A proof of that statement might look like:
```coq
Proof.            (* env: {},                              goals: { forall A B, A -> B -> A /\ B } *)
  intros A B a b. (* env: { A, B : Prop ; a : A ; b : B }, goals: { A /\ B } *)
  split.          (* env: { A, B : Prop ; a : A ; b : B }, goals: { A ; B } *)
  - exact a.      (* env: { A, B : Prop ; a : A ; b : B }, goals: { B } *)
  - exact b.      (* env: { A, B : Prop ; a : A ; b : B }, goals: {} *)
  Qed.
```
At the start of the proof, the environment is empty and there is one goal - the whole statement. The first step, `intros`, introduces names into the environment: `A`, `B`, for the types, and `a`, `b` for the proofs - and changes the goal to `A /\ B`. The second step `split`s one goal into two - `A` and `B`. The third and fourth steps each prove a single goal by providing a term which is `exact`ly its proof.

Coq also allows for such terms to be expressed as functions directly: and vice versa. There's no distinction between what's expressible in the "code style" or the "proof style" of Coq terms.

## Proof style with monads

Looking at the proof style available in Coq and other provers, one immediate candidate for a Haskell implementation springs to mind: do-notation. Haskell's do-notation already gives syntactic support for sequences of steps and name-binding, through `x <- ...`.

In order to leverage do-notation, we then have to find an appropriate Haskell type which supports it (or otherwise implement one ourselves). While a `Monad` instance would be the most obvious choice, it wouldn't be powerful enough; in order to support a changing goal, something about the `Monad` type must change over time. We also want to have type-level *restrictions*: the `split` step, which turns a `A /\ B` goal into two subgoals, shouldn't be usable on a goal which isn't an `/\`.

### An indexed monad

It turns out that we can overcome these problems by adding some *type-level state* to our monad. The result is called an [**indexed monad**][1], which (along with definitions for indexed functors and applicatives), has the declaration:
```haskell
class (IxApplicative m) => IxMonad m where
  ibind :: (a -> m j k b) -> m i j a -> m i k b
```

An `IxMonad` instance is a type constructor with 3 arguments: the old typestate, the new typestate, and the monad argument. When sequencing indexed monad values, the typestates must "align": the new typestate of the first value must be the old typestate of the second. This is easier to see in the `IxApplicative` definition (slightly modified here for readability):
```haskell
class IxFunctor m => IxApplicative m where
  ipure :: a -> m i i a
  iap :: m i j (a -> b) -> m j k a -> m i k b
```
`iap`, as well as applying the function type, also "composes" the typestates: the resulting monad value has the old typestate of the first argument, and the new typestate of the second.

### The goal as typestate

So we might want an indexed monad; but what should its typestate be? The type-level information we care about are the environment and the goal. However, the environment (a collection of variables and definitions) is already handled by Haskell as a programming language: do-notation even has a way to bind variables, with the arrow `<-`.

It follows that the typestate of our indexed monad turns out to be the *goal*. Proof steps, as monad values, are really then *goal transformations*. Such a monad value is itself a proof that, in a certain environment, some goal transformation is valid: a proof step that goes from a goal of `a` to a goal of `b` must also prove that from a proof of `b` it's possible to get a proof of `a`.

### The `Tactic` monad

How then do we actually define the monad? We can do that by thinking about what valid goal transformations we will want to have.

If a monad value has some final type variable `a`, then it will be possible to bind that value to a variable which then has type `a`:
```haskell
x <- myTransformation -- myTransformation has type m i j a
...                   -- from here on, x has type a
```
The remaining proof with goal `j` will have access to a proof of `a` through the variable `x`. Such a goal transformation *introduces* `a` as a hypothesis for the rest of the proof; if the goal from that point on is `j`, then the proof shows that `a -> j`.

But the original goal was `i`, not `j` - so the goal transformation has to justify why a proof of `a -> j` gives a proof of `i`. In other words, the goal transformation is an inhabitant of `(a -> j) -> i`.

This gives our monad definition:
```haskell
data Tactic i j a
  = Tactic ((a -> j) -> i)
```
The name `Tactic` comes from the term used for such proof steps in many theorem provers.

### Some tactics

Now that we know the shape of the monad, it's very easy to start writing tactic instances like the ones we would expect to see in any theorem prover (if you're not interested in examples, skip this section).

Coq's `intro`, for example, allows you to give a name to the left side of an `->`:
```haskell
intro :: Tactic (a -> b) b a
```
Such a goal transformation requires an inhabitant for `(a -> b) -> (a -> b)` - so the definition is pretty obvious:
```haskell
intro = Tactic id
```

The `left` tactic simplifies proving an `\/`:
```haskell
left :: Tactic (a \/ b) a ()
```
This tactic doesn't introduce any hypothesis - and that won't be a problem later, since we can always construct a value of type `()` when necessary.
```haskell
left = Tactic \f -> Left (f ())
```

## Canonical truth

The `Tactic` monad (once you define the necessary typeclass instances) lets you do proof steps with do notation: but we still can't describe a whole proof. Specifically, we don't know when we are "done" proving a statement - after all, we could always apply further goal transformations.

We could be satisfied by reducing the proof of a statement to the proof of some other statement which we've already proved i.e. producing a term of type `Tactic a b ()`, where we know `b` and want to prove `a`. Such a proof would inhabit `(() -> b) -> a)` - and since `() -> b` is inhabited (since we know `b` to be true), then we can get an inhabitant for `a`.

In practice, however, it's useful to have a *single* choice for such a known statement - and that statement represents "truth" in the type system. Additionally, it would be nice for this choice of truth to have a single, **canonical** constructor - so that all proofs of truth are the same. Together, these justify choosing `()` as the representation of truth.

### Even more tactics

Using this representation of truth, we can describe tactics that *solve* goals - that is, transform goals into the goal of `()`. (Again, if you don't want examples, skip this section.)

Coq's `exact` tactic solves a goal by giving a term of the exact type of the goal:
```haskell
exact :: a -> Tactic a () ()
exact proof = Tactic \_ -> proof
```

The `split` tactic in Coq turns a goal of `A /\ B` into two subgoals of `A` and `B`. However, the `Tactic` monad doesn't support multiple goals: so how can we represent it? The answer is through *subproofs*:
```haskell
split :: Tactic a () () -> Tactic b () () -> Tactic (a /\ b) () ()
```
These subproofs require solving their respective subgoals. The tactic then allows you to solve a goal of `a /\ b` by providing a proof for each of `a` and `b`. The resulting proofs can finally be combined:
```haskell
split (Tactic getA) (Tactic getB) = Tactic \trivial -> (getA trivial, getB trivial)
```

This technique of subproofs allows for defining lots of other useful tactics. The `assert` tactic in Coq allows for stating a subgoal, proving it, and giving the resulting proof a name to use later.
```haskell
assert :: Tactic a () () -> Tactic i i a
```
`assert` takes the proof of the subgoal, and allows it to be bound in do-notation. Using the bound variable later on just uses the subproof.
```haskell
assert (Tactic getSubproof) = Tactic \_ -> getSubproof id
```

## A complete proof

We can then define a complete proof as one which solves its goal:
```haskell
data Proof a = Proof (Tactic a () ())
```
Of course, such a definition is justified only if we can use it to get an inhabitant of `a`. In fact, this defines a complete proof as inhabitant of `(() -> ()) -> a`, which can very easily be used to obtain an inhabitant of `a`:
```haskell
useProof :: Proof a -> a
useProof (Proof (Tactic transformation)) = transformation id
```

 [1]: https://hackage.haskell.org/package/indexed
 [2]: https://coq.inria.fr/
