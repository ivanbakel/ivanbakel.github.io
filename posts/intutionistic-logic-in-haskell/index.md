.. title: Intuitionistic logic in Haskell
.. date: 2020-09-19
.. tags:
.. link:
.. description:

# Intuitionistic logic in Haskell

This article assumes a familiarity with Haskell, and some basic knowledge of classical logic (if you don't know about different varieties of logic, the one you *do* know is probably classical).

## Terminating Haskell

### The `Void` datatype

The `Void` datatype is part of the Haskell standard library, and should be well-known by most Haskellers. In short, `Void` has the following declaration
```haskell
data Void
```
That is: it's a datatype, with an empty collection of constructors (you may be surprised this is a valid declaration). The consequence is that it's impossible to construct any value with type `Void`, a fact that both programmers and the compiler can exploit.

Though a `Void` value is *unconstructable*, it is still very simple to write a valid Haskell term which has the `Void` type.
```haskell
aVoidTerm :: Void
aVoidTerm = aVoidTerm

-- Alternatively:
aVoidTerm = undefined
-- Or even:
aVoidTerm = error "Tried to evaluate a `Void` term"
```
These terms all share the property of being **non-terminating**. While lazy evaluation lets them appear in programs without any problem, any attempt to evaluate these terms will fail: either because of an infinite loop or a runtime error.

### Terminating terms

**Terminating** terms are the subset of all Haskell terms which can be evaluated in finite time without error. While it's impossible to decide in general if a particular term is terminating, we can restrict our language so that we can *only* write terminating terms. One way to do that is to require that recursive functions are **strictly positive** - so all recursive calls are with arguments that are "smaller" than the current argument - and **total** - defined for every possible argument.

If we implement those two restrictions in Haskell, we get a subset of the language that can only express terminating Haskell terms - though it can't express every terminating Haskell term. In terminating Haskell, we can *no longer* write terms with a `Void` type: there's no way to define such a term recursively, because it won't get "smaller"; and there's no way to leave the value `undefined`.

If you can force the evaluation of a `Void` term, you can't get some terminating value, so there is no well-defined "after" evaluation. The compiler even lets you exploit this fact to write a terminating term that evaluates a `Void` value, and then returns a value of *any type*.
```haskell
{-# LANGUAGE EmptyCase #-}
useAVoid :: Void -> a
useAVoid void = case void of
```
This code uses the `EmptyCase` language extension to allow us to write a `case` statement with no branches - remembering that `case` statements have to be total, this means that the above code only compiles because `Void` has *no cases* to branch on. While the function definition itself is terminating, the result can have type `a`, for any choice of `a`, because by evaluating the argument it will never terminate and be forced to produce an `a`.

### Inhabited types

`Void` has the property of being **uninhabited**, because it has no "inhabitants" - valid terminating terms which have the `Void` type. Types with inhabitants are, unsurprisingly, said to be **inhabited**.

We have already seen that `Void -> a` is inhabited for any choice of `a`, even uninhabited choices - in fact, this is only because `Void` is uninhabited. For a (terminating) function with type `a -> b`, `b` can be uninhabited only if `a` is uninhabited - otherwise the function could evaluate the argument of type `a`, have it terminate, and be forced to produce a terminating value of type `b`: an impossibility.

A consequence is that the type `a -> Void` is inhabited only for choices of `a` which are uninhabited. Moreover, if `a` is uninhabited, then we can write a terminating term with type `a -> Void` - just like we did for `Void -> a` - by exploiting the fact that `a` has no inhabitants. The result is: `a -> Void` is inhabited if and only if `a` is uninhabited, and vice versa.

We can extend this reasoning about inhabitants to many other basic Haskell types. `Maybe a`, for example, is always inhabited by the terminating term `Nothing`, even for uninhabited choices of `a`. `Either a b` is inhabited provided one of `a` or `b` is inhabited, because you could wrap the terminating term with type `a` (or `b`) in a `Left` (or `Right`) constructor to give a terminating term of type `Either a b`. Conversely, if `Either a b` is inhabited, then *at least one* of `a` or `b` must be inhabited (though the proof is much more difficult to summarize). In a similar vein, the tuple type `(a, b)` is inhabited if and only if *both* `a`, `b` are inhabited.

## Types and logic

Astute readers may have already spotted the point of the above discussion: this reasoning about inhabited types looks a lot like formal logic.

If inhabitedness is "truth", then uninhabitedness is "falsehood". Continuing the comparison, `a -> Void` is the "negation" of `a` - since it is uninhabited if and only if `a` is inhabited, and vice versa; `Either a b` is the "or" of `a` and `b`, since it is inhabited if and only if at least one of `a`, `b` is'; and `(a, b)` is the "and".

It remains to check that `a -> b` follows the expected behaviour of "implies" - that `a -> b` is uninhabited (aka false) if and only if `a` is inhabited (true) and `b` is uninhabited (false). In fact, due to arguments which include the ones made earlier on, we find that this is the case - and that `->`, as well as resembling the "implies" arrow, also seems to act like one.

The similarities don't stop there. If we introduce some type aliases to make our types easier to read:
```haskell
{-# LANGUAGE TypeOperators #-}
type Not a = a -> Void
type a /\ b = (a, b)
type a \/ b = Either a b
```

Then we can wonder if some inhabitants exist for certain types which correspond to logical statements we know to be true. One example would be the commutativity of `/\` - that `a /\ b -> b /\ a`. A programmer could quickly produce such an inhabitant:
```haskell
andComm :: a /\ b -> b /\ a
-- i.e. :: (a, b) -> (b, a)
andComm (x, y) = (y, x)
```

We can also wonder if types which corresponding to false logical statements are *uninhabited*. For example, `a /\ Not a` should never hold, so the type should be uninhabited. It follows that `Not (a /\ Not a)` is inhabited: in fact it is.
```haskell
explosion :: Not (a /\ Not a)
--   i.e. :: (a, a -> Void) -> Void
explosion (x, f) = f x
```
Programmers who struggle with logic may notice that rewriting the type in terms of Haskell types makes for much simpler reading: it's pretty easy to spot that `(a, a -> Void) -> Void` is inhabited, and even to immediately write out the above inhabitant.

### The excluded middle

A pretty important rule for classical logic is the "law of the excluded middle" - that every statement is either definitely true, or definitely false. 

We can express this in our types as `a \/ Not a`. But can we inhabit it? For particular choices of `a`, sure: if we knew that `a` was `Int`, or `Void`, or even `Maybe b`, we would be able to produce an inhabitant easily:
```haskell
intLOEM :: Int \/ Not Int
intLOEM = Left 1

voidLOEM :: Void \/ Not Void
--  i.e. :: Either Void (Void -> Void)
voidLOEM = Right id

maybeLOEM :: (Maybe b) \/ Not (Maybe b)
maybeLOEM = Left (Nothing)
```

But for a *polymorphic* `a`, we struggle. While every type in Haskell is definitely inhabited or uninhabited, we can't in general produce an inhabitant of the type, or an inhabitant for the type's "negation", without knowing *which* type we're dealing with.
```haskell
LOEM :: a \/ Not a
LOEM = ??? -- uh oh
```

Does this mean that our strange similarities are just a coincidence? After all, `a \/ Not a` is certainly true - so we expect to be able to produce an inhabitant. Notably, we also fail to "negate" the law of excluded middle - we can't produce an inhabitant for `Not (a \/ Not a)` either. In the logic of our types, we don't know if `a \/ Not a` is "true" or "false".

What has actually happened is that we've departed from *classical* logic: our types do behave like a logical system, just a different one.

### What makes classical logic?

While most users of logic - particularly mathematicians - take the law of excluded middle as a universal truth, and apply it liberally in proofs, it's not a necessary part of any logical system. Classical logic is actually defined in part by the existence of this law, and double negation: `Not (Not a) -> a` (interested readers can [read more about these laws][2]).

Double negation is another statement that doesn't have an inhabitant in Haskell - and nor does its negation. In a logical system without either law, several statements which are true in classical logic also no longer hold: Peirce's law, `((a -> b) -> a) -> a`; the double contrapositive, `(Not b -> Not a) -> (a -> b)`; the implies equivalence `(a -> b) -> (Not a \/ b)`; de Morgan's laws, such as `Not (Not a /\ Not b) -> (a \/ b)`; and many more. Similarly, none of these Haskell types have inhabitants.

### Intuitionistic logic

If the logic that we see in Haskell types isn't classical logic, what then is it?

The feature of Haskell that prevents all these types from having inhabitants is that, in Haskell, we must *construct terms explicitly*. In classical logic, the statement `a \/ b` holds if and only if at least one of `a`, `b` holds - but our knowledge of `a \/ b` doesn't necessarily tell us which one it is. In Haskell, on the other hand, you can use an `Either a b` to get either an `a` or a `b`, by *deconstructing* the term. In other words, Haskell requires that every term of type `a \/ b` only be *constructed*, from a term with type `a` or `b`.

This limitation doesn't just appear in Haskell. In fact, this exact behaviour describes **intuitionistic logic** - a subset of classical logic. Intuitionistic logic is precisely what you get when you get rid of the laws of the excluded middle and double negation frrom classical logic. The resulting system still makes sense - it's just less powerful. Lots of statements with classical proofs have no equivalent intuitionistic proof.

### Constructive mathematics and decidability

Just like classical logic tries to formalise the reasoning of classical mathematics, intuitionistic logic tries to formalise the reasoning of **constructive mathematics** (for this reason, it is sometimes called *constructive logic*).

Godel's successful proof of the first Incompleteness theorem showed that every system of reasoning about numbers included statements that were **undecidable**. These statements couldn't be proven, and their contradiction couldn't be proven - in other words, it was impossible for *some* statement `a` to prove either `a` or `Not a`.

As a result, several mathematicians became dissatisfied with classical logic: a binary idea of truth no longer seemed correct. They devised a new mathematics without the use of the law of excluded middle or double negation in proofs - since a proof of `Not (Not a)` only showed that `a` wasn't false: not that it was true. This system was termed **constructive mathematics**.

In constructive mathematics, as in intuitionistic logic, proofs are only possible by explicit construction. To prove `a \/ b`, it is necessary to give a proof of `a` or a proof of `b` - it's not enough to argue that at least one must be true. This system ends up being weaker - several statements which are provable in classical mathematics are undecidable in constructive mathematics.

### Decidability as a constraint

Returning to Haskell for a moment, we can use this insight to produce a weaker version of the law of the excluded middle, just for *decidable* statements. Haskell allows us to express decidability as a constraint on a logical statement (a type), by using a typeclass:
```haskell
class Decidable a where
  decide :: a \/ Not a

LOEM :: (Decidable a) => a \/ Not a
LOEM = decide
```

Restricting ourselves to decidable statements is even enough to re-introduce double negation:
```haskell
doubleNegation :: (Decidable a) => Not (Not a) -> a
doubleNegation doubleNegative
  = case (decide :: a \/ Not a) of
      Left a -> a
      Right (notA) -> useAVoid (doubleNegative notA)
```

## The Curry-Howard isomorphism

So Haskell types look like logical statements - and in particular, it looks like statements provable in intuitionistic logic correspond to inhabited Haskell types. But what does that mean for Haskell *terms*?

### Code as proof

To the Haskell compiler, a (terminating) term is a demonstration that a particular type is inhabited. In declaring the type signature for a definition, you claim the type is inhabited: in the definition itself, you *prove* it. When you consider types as a logic, this analogy makes even more sense: where a type is a logical statement, a term is a proof that the type is inhabited, and so that the statement is true. Proving the negation of a statement is done by showing that the negated type is inhabited.

This matchup - types as statements, terms as proof - hasn't gone unnoticed among computer scientists: it's been described in [informal papers][1] since the 70s. Named the **Curry-Howard Correspondence** (or isomorphism), it was first spotted in the lambda calculus - the functional abstract computer that Haskell was inspired by.

### Computing proofs

One of the most interesting parts of the CHC comes from considering `->` - both implication and function. The type `a -> b` is both the logical statement that `a` implies `b`, and the type of transformations of values of type `a` into values of type `b`. But since a value of type `a` is a proof for `a`, such a transformation also *transforms proofs*: given a proof for `a`, it yields a proof for `b`.

And since proofs are code, that means that evaluating a proof involving running those proof transformations. A Haskell term like `f x` with type `b` is made of a proof `f` of `a -> b`, and a proof `x` of `a` (for some `a`). In *running* `f x`, we use the proof of `a -> b` combined with the proof of `a` to get a proof for `b`: and the resulting value has the same type. By evaluating a proof, we get a new proof for the same statement. In other words, computation *simplifies* proofs.

### Cut-free proofs

But what does this "simplification" actually mean in a proof? How can proofs transform other proofs? The answer comes from considering what our logical types actually mean. `a -> b` is a representation of the statement "from the hypothesis a, it is possible to prove b". Inhabitants of `a -> b` are proofs which assume that `a` is true, and use it to construct a proof of `b`. But in order to *use* such a proof to prove `b`, it is necessary to produce a proof of `a`: and the resulting proof has some redundancy. If you can prove `a`, there's no need for another proof to *assume* that `a` is true - it could just use the proof of `a`.

Consider the same situation in Haskell. If you apply a function to an argument
```haskell
(\x -> ...) t
```
then computing the function application substitutes the argument value into the function body.
```haskell
let x = t in ...
```
This term could be written both ways: the behaviour is the same. The only difference is that the first version, when computed, turns into the second.

This construction - proving both an implication and its assumption - is called a **cut** in a proof. When we evaluate a proof as a Haskell term, we actually *eliminate* the cuts in the proof: since a cut is just a function application, and computing function applications is how Haskell code is executed. Since we are limited to terminating terms, executing proofs must eventually give us a simplified proof with *no cuts* in it (because it cannot be simplified any more).

This line of reasoning leads us to a very significant result about intuitionistic (and classical) proofs: the **cut elimination theorem**. Shown by [Gentzen][3] in 1935, the theorem states that every statement with a proof that contains cuts, also has a proof that contains *no* cuts. We have the framework to prove the same result: if every intuitionistic proof corresponds to a terminating Haskell term, where cuts are function application, then computation does the rest of the work. To get a cut-free version of an existing proof, it is sufficient to run the proof fully.

  [1]: http://www.dcc.fc.up.pt/~acm/howard.pdf
  [2]: http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.57.6941
  [3]: https://link.springer.com/article/10.1007/BF01201363
