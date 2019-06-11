---
layout: post
title: "The place of applicative style in today's Scala"
---
Have you ever used the `<*>` operator from `Apply` typeclass, except just to
try to understand it? It's much more common to use, and even explain applicative functors
via a syntax sugar on tuples, i.e. `(fa, fb).mapN { (a, b) => ??? }`. The "double shark" is
designed to utilize Haskell's strengths - always curried functions and Hindley-Milner type
inference, which Scala does not have, making use of `<*>` require quite a bit of ceremony:

```haskell
f <$> e1 <*> e2 <*> e3
```

vs.

```scala
// Trying to imitate Haskell
{ f[A, B, C](_, _, _) }.curried.pure[F] <*> e1 <*> e2 <*> e3
// Not doing it
(e1, e2, e3).mapN(f)
```

It's evident that there are cases - and probably most of such cases - where version
playing to Scala strengths requires both less reading/writing and can utilize inference better.
But there's at least one scenario where I'd use the other style, if I could, and that is...

## Concurrent data structures

When I fully bought into cats-effect and started using fs2 heavily, lots of my classes began
having constructors that look like this:

```scala
class StatefulObject[F[_], A, B] (
  pending: Ref[F, Map[B, Deferred[F, NonEmptyList[A]]]],
  updates: Topic[F, Option[(B, NonEmptyList[A])]],
  mutex: MVar[F, Unit]
) { ... }
```

<!--more-->
As you can see, the type of the state involved is quite complex and composed from multiple pieces.
Also, classes like this require an effectful constructor, so we make one on companion object:

```scala
object StatefulObject {
  def empty[F[_]: Concurrent, A, B]: F[StatefulObject[F, A, B]] = ...
}
```

And here comes the devil. Scala's inference is left-to-right, loves inferring useless types
like `Ref[F, Map[Nothing, Nothing]]` from initializer, and all these concurrent data structures are
invariant (for a good reason). That means we can't do something naive:

```scala
(
  Ref.of(Map()), // infers Ref[Nothing, Map[Nothing, Nothing]]
  Topic(None)    // infers Topic[Nothing, None.type]
  MVar.of(())    // infers MVar[Nothing, Unit]
).mapN(new StatefulObject(_, _, _)) // fails compilation, because above types don't fit at all.
```

Rather, to use `mapN`, we are forced to repeat most of the types in `apply` too.

```scala
(
  Ref[F].of(Map.empty[B, Deferred[F, NonEmptyList[A]]]), // hell, I can't even read this mess
  Topic[F, Option[(B, NonEmptyList[A])]]](None)    // topic doesn't have a type-param-curried constructor
  MVar[F].of(())    // the best case - we have the initializer of a proper type without any extra BS like in Map case
).mapN(new StatefulObject(_, _, _))
```

And I don't like this. I previously [have tried](/traversing-hlists) to rectify it with type classes, but
that solution is fairly limited in what initializers can be, and require making lots of instances to be useful.

That is where applicative style could help - it's much easier to supply the type parameters for a constructor to get
a function that we can double-shark later:

```scala
val construct = { new StatefulObject[F, A, B](_, _, _) }.curried.pure[F]
construct <*> Ref.of(Map()) <*> Topic(None) <*> MVar.of(())
```

Now _that_ has much less boilerplate. We have managed to mention _just enough_ types (`F`, `A`, `B`) for Scala to infer the
types of all complex expressions involved in constructor, like that `Ref[..]` above (scroll up, I'm not typing it again),
and now it is able to use that information to _dictate_ the types for constructor parameters to match.

Or is it? If you have tried it out, you'd find out that...

## ...it doesn't work (with today's cats)
With cats, you get a glorious error about not being able to prove that some crazy type <:< some other crazy type.
The origin of a problem is how [simulacrum](https://github.com/typelevel/simulacrum) generates syntax extensions.

```scala
// cats' definition of this typeclass
@typeclass trait Apply[F[_]]
  def <*>[A, B](ff: F[A => B])(fa: F[A]): F[B]
}
// gives something like this in macro-generated code
implicit class ApplyOps[F[_], A](self: F[A]) {
  def <*>[A0, B0](fa: F[A0])(implicit ev: A <:< (A0 => B0))
}
```
With this encoding, we end up with three parameter lists, where `A` and `A0` are not related at all, so they would be
inferred from scratch, and only the type of `ev` would be "dictated".

That, unfortunately, renders cats' `<*>` operator absolutely useless to us. Best we can do today is to write a custom one:

```scala
object inferencedAp {
  implicit class ApOps[F[_], A, B](private val self: F[A => B]) extends AnyVal {
    def <|> (fa: F[A])(implicit F: Apply[F]): F[B] = F.ap(self)(fa)
  }
}
```

With it, there are only two distinct parameter lists (one with `self` and one with `fa`), and `self` will be
used to dictate the state of an expression. That is exactly the arrangement we're after:

```scala
object StatefulObject {
  def empty[F[_]: Concurrent, A, B]: F[StatefulObject[F, A, B]] = {
    val construct = { new StatefulObject[F, A, B](_, _, _) }.curried.pure[F]
    construct <|> Ref.of(Map()) <|> Topic(None) <|> MVar.of(())
  }
}
```

As usual, the code is [available on Scastie](https://scastie.scala-lang.org/f2W9OnBkREOdHVFJrBTg3g) to play or copy-paste.

Overall, I'm quite happy with this approach, and it has proven to save a lot of headache as I tinker with the
number and types of data structures needed. Better than waiting for dotty, which might or might not fix it.
I hope the inference for `<*>` will be fixed in some future cats version too.
