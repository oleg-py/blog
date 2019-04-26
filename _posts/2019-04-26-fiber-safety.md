---
layout: post
title: "Practical fiber safety (and how Concurrent implies Parallel)"
---

A short post, for a change.

cats-effect has `start`. And `start` has a problem that is outlined right in its docs [(link)](https://github.com/typelevel/cats-effect/blob/c64be615a980027a778615cd9c90c200e397dc34/core/shared/src/main/scala/cats/effect/IO.scala#L356):

```
* {{{
*   def par2[A, B](ioa: IO[A], iob: IO[B]): IO[(A, B)] =
*     for {
*       fa <- ioa.start
*       fb <- iob.start
*        a <- fa.join
*        b <- fb.join
*     } yield (a, b)
* }}}
*
* Note in such a case usage of `parMapN` (via `cats.Parallel`) is
* still recommended because of behavior on error and cancellation —
* consider in the example above what would happen if the first task
* finishes in error. In that case the second task doesn't get canceled,
* which creates a potential memory leak.
 ```
<!--more-->
That doesn't quite feel right. Especially in presense of cancelable `flatMap`s, it's possible that we end up with a more ugly situation:
```scala
fb <- iob.start
// whole operation gets cancelled there
a <- fa.join // fibers have leaked
```

I thought about it for a while, and came up with a simple rule for safe usage of fibers:
* if you don't ever intend to `join` or `cancel`, and just want to spin off a background process, use `io.start.flatMap`
* if you do, use bracket and _always cancel_, e.g. `io.start.bracket(...)(_.cancel)`
  * and if you work with multiple fibers, always join them via `.parTupled`/`.parSequence`.

Always cancelling is a moment that didn't occur to me for a long time, but it has the desired semantics:
* cancellation is a no-op if the fiber has already completed
* cancellation is idempotent, so if something cancels your `join`, it's still safe to call `cancel`
* as a bonus, any errors will trigger cancellation, so no leaks would happen.

So, how would fixed `par2` look like and work with this model?

```scala
def par2[A, B](ioa: IO[A], iob: IO[B]): IO[(A, B)] =
  ioa.start.bracket { fa =>
    iob.start.bracket { fb =>
      for {
        a <- fa.join
        b <- fb.join
      } yield (a, b)
    }(_.cancel)
  }(_.cancel)
```

But that's quite verbose. We can reach for tuple methods:

```scala
def par2[A, B](ioa: IO[A], iob: IO[B]): IO[(A, B)] =
  (ioa.start, iob.start).tupled.bracket { case (fa, fb) =>
    (fa.join, fb.join).tupled
  } { case (fa, fb) => fa.cancel >> fb.cancel }
```

Doing that is also perfectly safe, because acquisition in `bracket` is uncancelable, so `tupled` will never be cancelled.

Alternatively, one might want to reach for `Resource`:

```scala
def startR[A](io: IO[A]): Resource[IO, Fiber[IO, A]] = Resource(io.start.fproduct(_.cancel))

def par2[A, B](ioa: IO[A], iob: IO[B]): IO[(A, B)] = 
  (startR(ioa), startR(iob)).tupled.use { case (fa, fb) => (fa.join, fb.join).tupled }
```

With not much effort, we have achieved the following:
- if the result of `par2` is started and then is cancelled, both `ioa` and `iob` are cancelled
- if `ioa` fails with error, `iob` is cancelled
- if `iob` fails with error, `ioa` is *NOT* cancelled

That non-cancellation of `ioa` on a failure of `iob` is caused due to all implementations above sequencing joins strictly in order. So, if we wanted to rectify that, we need to avoid the sequencing. The simple solution for applications is to always use `parTupled`, `parSequence`, or something of the like.

Obviously, as we are actually trying to _implement_ `parTupled` from scratch, it's not of much help. So let's reach out to the other method on `Concurrent`, namely `racePair`, and materialize/dematerialize errors with `attempt` and `rethrow` respectively:

```scala
def par2[A, B](ioa: IO[A], iob: IO[B]): IO[(A, B)] = 
  (startR(ioa), startR(iob)).tupled.use { case (fa, fb) =>
    IO.racePair(fa.join.attempt, fb.join.attempt).flatMap {
      case Left((Left(ex), _))    => IO.raiseError(ex)
      case Right((_, Left(ex)))   => IO.raiseError(ex)
      case Left((Right(a), fb2))  => (a.pure[IO], fb2.join.rethrow).tupled
      case Right((fa2, Right(b))) => (fa2.join.rethrow, b.pure[IO]).tupled
    }
  }
```

That ain't much, but hey, we've implemented one of key methods of `Applicative` that does computation in parallel, using only methods available on `Concurrent` typeclass. So, we can implement `Parallel[F, G]` for any `Concurrent[F]`. with that too. For some kind of composed applicative of `type G[x] = Resource[IO, Fiber[IO, Either[Throwable, x]]]`. 

That, in turn, means that `Concurrent` from cats-effect can always imply `Parallel` from cats. That is a neat result, but nobody probably wants to deal with the above type, so I'll leave implementing a generic one as an exercise for a reader with a lot of free time and patience.
