---
layout: post
title: "Things to store in a Ref"
---
Cats-effect 1.0.0-RC2 is out, and among various improvements, it brought to us some goodies in `cats.effect.concurrent`, exported from fs2 and Monix with a number of changes:

* Ref - pure mutable reference
* Deferred - a purely functional alternative to `Promise`
* Semaphore - an access control tool
* MVar - a mutable location that can be empty, useful as synchronization and communication channel

This post will focus on `Ref` and to show what interesting techniques it enables.

<!--more-->
You'll need these imports to follow along:
```scala
import cats.implicits._
import cats.effect._, concurrent._
import scala.concurrent.duration._
import scala.concurrent.ExecutionContext.Implicits.global
```

## Mutable state

A simplest and most straightforward use of `Ref` is as a mutable state reference:

```scala

val program = for {
  ref <- Ref[IO].of(1)
  printValue = ref.get.flatMap(i => IO(println(s"Current value is $i")))
  _ <- printValue
  _ <- ref.update(_ + 1)
  _ <- printValue
  _ <- ref.update(_ * 2)
  _ <- printValue
} yield ()

program.unsafeRunSync()
```
Outputs:
```
Current value is 1
Current value is 2
Current value is 4
```

This is somewhat boring, but can serve as a starting point of porting code that uses (local) mutable state.

#### Note on sharing

Note that `Ref` construction in above example happens in for-comprehension. This is intentional: return value of `Ref[IO].of(1)` is `IO[Ref[IO, Int]]`, because creating a mutable variable is an effect, as is updating it.

If you haven't used `Ref`-like data type previously, try to guess what this code will print:

```scala
val ref = Ref[IO].of(0)

for {
 _ <- ref.flatMap(_.update(_ + 42))
 x <- ref.flatMap(_.get)
 _ <- IO(println(s"The value is $x"))
} yield ()
```

If you guessed "42", well, nope. This is purely functional - referentially transparent - code, so we can blindly substitute variable `ref` with its definition, getting:

```scala
for {
  _ <- Ref[IO].of(0).flatMap(_.update(_ + 42))
  x <- Ref[IO].of(0).flatMap(_.get)
  _ <- IO(println(s"The value is $x"))
} yield ()
```

And if you need to share a state, `Ref` needs to be passed as a parameter:

```scala
def periodicReader(ref: Ref[IO, Int]): IO[Unit] =
  IO.sleep(1.second) >> ref.get.flatMap(i => IO(println(s"Current value is $i"))) >> periodicReader(ref)
  
def periodicIncrementer(ref: Ref[IO, Int]): IO[Unit] =
  IO.sleep(750.millis) >> ref.update(_ + 1) >> periodicIncrementer(ref)
  
for {
  ref  <- Ref[IO].of(0)
  read <- periodicReader(ref).start
  incr <- periodicIncrementer(ref).start
  _    <- IO.sleep(10.seconds)
  _    <- read.cancel
  _    <- incr.cancel
} yield ()
```

This puts some restrictions on your ability to share mutable state: all such sharing needs to be stated explicitly, and the piece of state shared needs to be created before each of components that are going to use it. Such limitations may seem annoying and painful (esp. together with inheritance), and yet, from long-term maintenance perspective, it'll make it easy to understand where sharing is happening.

## `IO`-terators

We can use `Ref` and `IO` to turn a sequence into an `IO` that provides `Iterator`-like behavior:

```scala
object StopIteration extends Exception

def mkIterator[A](seq: Stream[A]): IO[IO[A]] = 
  for {
    ref <- Ref[IO].of(seq)
    next = ref.get.flatMap {
      case x #:: xs => ref.set(xs).as(x)
      case _ => IO.raiseError[A](StopIteration)
    }
  } yield next
```

```scala
val prog = for {
  next <- mkIterator(Stream(1, 2, 3))
  a <- next
  b <- next
  c <- next
} yield (c, b, a)

prog.unsafeRunSync() // (3, 2, 1)
```

Not too shabby, eh? The interesting property of that construct is that a number of combinators are already available to us by virtue of `IO` being a monad.

#### map

Mapping values is the most easy and straightforward:

```
val nextString = next.map(_.toString)
```

The fact that each evaluation results in a new value means that mapping function, too, will be applied to each new value.

#### lazy foreach / traverse
If we want to apply a function that performs some effects (`A => IO[B]`) to our *IO-terator*, it's our well-known friend `flatMap`:

```scala
def writeToDb(i: Int): IO[Boolean] = IO { println(s"Wrote $i to DB"); true }
val nextBool = next.flatMap(writeToDb) // iterates over Ints, performs a `writeToDb` for each and gives a `Boolean` back.


(nextBool *> nextBool *> nextBool).unsafeRunSync()
```

#### filter

We also have a predefined operation for filtering. Behold not-so-often used `iterateUntil`:

```scala
val nextOdd = next.iterateUntil(i => i % 2 == 1)
```

#### Grouping and parallelism

By virtues of `Applicative` and `Parallel` we get more combinators for free:

```scala
val everyOdd = next *> next
val nextTriple = (next, next, next).tupled
val next6Par = next.replicateA(6).parSequence
```

## Memoization

Okay, we know we can put some simple values like `Int` into a `Ref[IO, ?]`. What are the *interesting* values we can put into `Ref`?

It's time to recall the first line of `IO` documentation:

> A data type for encoding side effects as pure values

Yep, `IO[A]` is just a pure value. Which means we can have `Ref[IO, IO[A]]`... and inner `IO[A]` can modify the ref itself.

As an example of this technique, consider the following snippet:

```scala
def someMethod[A](io: IO[A]): IO[IO[A]] =
  for {
    ref <- Ref[IO].of(io)
    _   <- ref.update(_.flatTap(a => ref.set(IO.pure(a))))
  } yield ref.get.flatten
```

What it does? I spoiled it in title, of course. It does memoization:

```scala
val program = IO { println("Hey!"); 42 }

val exec = for {
  io <- someMethod(program)
  x  <- io
  y  <- io
} yield (x, y)

// Prints Hey!, then (42,42)
println(exec.unsafeRunSync())
```

To be precise, this memoizes only successful results. It's also not really safe for concurrent access*, but it can be generalized to only require `Sync[F]` constraint. We can make it properly memoize errors by using `.attempt` and `.rethrow` (coming from `MonadError`), and `Semaphore` can help to ensure safe access.

```scala
def memoize[F[_]: Sync, A](fa: F[A]): F[F[A]] =
  for {
    ref <- Ref[F].of(fa.attempt)
    _   <- ref.update(_.flatTap(a => ref.set(a.pure[F])))
  } yield ref.get.flatten.rethrow
```

Notice how little change did it take to make it save exceptions. Now, for the safety:

```scala
def safeMemoize[F[_]: Async, A](fa: F[A]): F[F[A]] =
  for {
    sem <- Semaphore.uncancelable[F](1)
    mem <- memoize(fa)
  } yield sem.withPermit(mem)
```

Another new addition - `Semaphore` - makes an appearance to fix the things.

> \* **Edit:** All methods provided on `Ref` are safe for concurrent access. However, in this case we're separately executing `set` *after* we're doing *modify*, and there's no way to describe the transformation we want as a primitive `Ref` operation.

Let me annoy you with a final code snippet to show that everything is indeed working as intended:

```scala
object Boo extends Exception("Boo!")
val program: IO[String] = IO { println("Executing..."); throw Boo }
safeMemoize(program)
  .flatMap(memo => List.fill(8)(memo.handleError { case Boo => "Got Boo" } ).parSequence)
  .unsafeRunSync()
```

## Conclusion

Thanks to great work of all contributors, cats-effect now has some useful primitives in addition to `IO` type. Now, a lot of things which previously required stepping into side effect land, can be described in pure fashion. And there's more to come - for example, there is a [currently open pull request](https://github.com/typelevel/cats-effect/pull/256) that will help to get rid of `IO { println(...) }` business.

I've barely touched concurrency primitives in this post, but they are more or less straightforward (or maybe I'm yet to discover some interesting things to do with 'em). Their most significant feature is, of course, that they are purely asynchronous, so they don't actually block JVM threads (like e.g. `java.util.concurrent.Semaphore` does) - preventing, however, the following code from executing until possible. This is sometimes referred to as "semantic blocking" or "asynchronous blocking".

Most of the code (with little changes) is available [on Scastie](https://scastie.scala-lang.org/oleg-py/ebHYVfMlSIm4NJBPLJbJlA).
