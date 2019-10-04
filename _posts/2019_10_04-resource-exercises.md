---
layout: post
title: "Scala exercises: abusing Resource"
---

So, cats-effect has `Resource`. It's nice. Some cats-effect users are considering making a `Resource` fanclub. Why? Well, probably because of some _neat things_ you can do with it, and two exercises here are to help you see what can be done outside of just flat-mapping some library ones and using them in the end.

These exercises are harder than previous ones. To make matters worse, I also now require
using cats-effect typeclasses instead of choosing a concrete `F`. You can test it with the
effect of your choice, of course, but it must work for everything.

On a positive note, I've provided scastie links with test cases that I used myself - it
is not enough to find all possible bugs, but should get you started quickly and check more
edge cases than my previous ones.

## Tearing resources apart
There's one unsafe method on Resource, and it's called `allocated`. It gives you
access to allocator and finalizer directly. In fact, originally `Resource` didn't
have it - that's how dangerous it is. However, it is required for some advanced
usages, like embedding a `Resource` into some other datastructure.

<!--more-->

For example, one can imagine a scope that manages lifetime of several resources:

```scala
sealed trait Scope[F[_]] {
  def open[A](ra: Resource[F, A]): F[A]
}

object Scope {
  // Implement this. Add context bounds as necessary.
  def apply[F[_]]: Resource[F, Scope[F]] = ???
}
```


The usefulness of that construct is two-fold. First, unlike using `Resource.liftF`, it can
preserve cancelability of other operations:

```scala
def foo: IO[Unit] = Scope[F].use { s =>
  for {
    socket <- s.open(getSocket)
    _      <- use(socket)
  } yield ()
}
```

```scala
def foo: IO[Unit] = {
  for {
    socket <- getSocket
    _      <- Resource.liftF(use(socket)) // will not be cancelable
  } yield ()
}.use(IO.pure)
```

Second, it's easier to write code that both creates and uses Resources, as you can use regular
for-comprehension without needing to wrap it in giant braces and call `.use(IO.pure)` at
the end

```scala
object Runner extends IOApp {
  def run(args: List[String]): IO[ExitCode] = Scope[IO].use { s =>
    for {
      httpClient <- s.open(BlazeClientBuilder(ExecutionContext.global).resource)
      proxy <- Proxy(httpClient)
      _  <- s.open(BlazeServerBuilder.bindHttp(80, "localhost").withHttpApp(proxy).resource)
      _  <- IO.sleep(10.minutes) // use IO.never to run forever
    } yield ExitCode.Success
  }
}
```

The requirements are as follows:
- All `open`ed resources should be properly finalized when `use` is finished,
  in order of acquisition. Even if one finalizer crashes, the rest should be
  attempted.
- `open` should be atomic WRT cancellation: resource is either opened or not, and
  cannot be cancelled in the middle of acquisition.
  
Both guarantees are provided to you with standard usage of Resource, but `allocated` removes them.

Here's a [scastie](https://scastie.scala-lang.org/ZAFgElmeRjOBHILNJH6TCQ) to get you started

## Supervising fibers
`Fiber` is a great tool, but very leaky: if you don't `.join` it, you can lose errors
(not even get them logged), and if you get canceled before you do something with `Fiber`
after `.start`, it can be undesirably kept running - possibly forever! - without any
way to stop it.

It's a common knowledge that you should use something like `Parallel` typeclass operations where possible.
It has some very nice properties (I'll use "par-fiber" to denote a logical fiber that is spawned by `Parallel` ops
without giving you actual `Fiber` handle):
- All par-fibers execute concurrently
- if a whole operation is cancelled, every its par-fiber is cancelled
- if at least one par-fiber errors, every other par-fiber is cancelled and error is propagated
- a whole operation completes sucessfully if, and only if, all par-fibers are completed successfully.

But it's not enough for many other applications, e.g. when you want to maybe start an unknown number of fibers that fetch
some data periodically, based on HTTP requests that you don't know ahead of time.

What can be good enough for most usages, though? Well, Nathaniel J. Smith suggests a limited version
of `.start` in [his blog post](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/)
and even has made a Python library implementing his vision.

I highly recommend to read it before getting started, but the core idea is simple - you only allow fibers to be started
in some scope, and they are guaranteed to be completed or cancelled when the scope ends.

You'll be implementing this approach yourself with the following API:

```scala
trait Nursery[F[_]] {
  def startSoon[A](fa: F[A]): F[Fiber[F, A]] 
}

object Nursery {
  // Implement this. You need `Concurrent` since you'd be using `.start` internally,
  // and you should NOT need anything else
  def apply[F[_]: Concurrent]: Resource[F, Nursery[F]] = ???
}
```

And since it should be more powerful than `Parallel`, you should be able to do all same things
with the same safety guarantees that you have from `Parallel`:

```scala
def concSequence[F[_]: Concurrent, G[_]: Traverse, A](gfa: G[F[A]]): F[G[A]] =
  Nursery[F].use { n =>
    gfa.traverse(n.startSoon(_)).flatMap(_.traverse(_.join))
  }
```

Here's how the `Resource[F, Nursery[F]]` should behave:
- The resource is finalized only when all fibers produced by `startSoon` are finished or explicitly
  cancelled.
- If the `use` is cancelled or results in error, all fibers produced by `startSoon` should be cancelled.
- `startSoon` should be atomic - there should be no potential for fiber to leak outside the control of 
  the `Nursery`. It's either started and will not leak, or not started, if `use` is cancelled.
- If any fiber has errored, the error should be propagated. Existing fibers should be cancelled
  and no new ones should be accepted. (optional - hard, but needed for parity with `Parallel`)

To help us provide the 1st guarantee, we will be relying on the fact that in the following construct:
```scala
for {
  _ <- someResource.use(whatever)
  _ <- doSomethingElse
} yield ()
```
The finalizer of `someResource` will always be finished _before_ `doSomethingElse` begins - even if
it's doing some async action, like waiting on a `Semaphore`/`Deferred`/`MVar` (this is not _yet_ a law of Bracket - possibly will be added as one in cats-effect 3).

For the 2nd guarantee, we will need to use a Resource constructor that lets us distinguish between
different exit cases - `Resource.makeCase`.

Here's a [scastie](https://scastie.scala-lang.org/wnu0vq3uQCu66dQm7Om7ZA) for this one, too.
