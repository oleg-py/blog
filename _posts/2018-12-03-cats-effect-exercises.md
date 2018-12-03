---
layout: post
title: "Scala exercises: cats-effect"
---

There's a lot of info on cats-effect these days, but a lot of it is about concrete use-cases. Yet there's not much for those who know the basics but don't feel confident enough with the library to build full-fledged applications. I've seen a couple of questions, however, which can be well generalized and are more complex and interesting than examples provided in documentation. So, I decided to turn them into exercises.

If you know how to do FP in Scala, and a bit about `cats` and `cats-effect`, the initial solution shouldn't take more than an hour for you to arrive at. If you struggle to find a solution, there will be a couple of hints. And for those who want to dive deeper, there are bonus assignments which require additional generalization and/or refactoring.

Currently, there are just two. I plan to slowly build up the list as I solve more interesting problems, mine or others'.
<!--more-->
So, all solutions:
- Must work on both JVM and JS. Don't use platform-specific APIs, in particular blocking APIs.
- Must not use impure methods (e.g. no `unsafeRun`).
  - I recommend to use IOApp to run everything, but in a worksheet environment it's fine to do unsafeRun at the very end.
  - Any call to stdlib impure method (e.g. `Random.nextInt()`) must be suspended in `IO`.
  - Don't use `var`s, stdlib atomics, mutable collections and `Future`s, even suspended.
- Must not use extra libraries besides `cats` and `cats-effect`.
  - Exception: you can use other effect types like `SyncIO` (where applicable), monix `Coeval` / `Task` with `TaskApp` or `ZIO`, either instead of cats' `IO` or to test generalizations. However, all concurrency primitives (`Ref`, `Deferred`, `MVar`, etc.) should come from `cats.effect` package.
  
To jumpstart everything, a source sample to start with is provided for each exercise. You are free to change anything, there are no restrictions to code style as long as it is readable. I recommend sticking to provided type signatures / interface structure for those you will be implementing, unless bonuses require you to change these.

To prevent spoilers, post your answers as github repos or gists, and post them in comments or @ me on twitter (link in the footer). Gists are easy to leave comment on, too!

## Worker pool with load balancing
#### Objective
Do parallel processing, distributed over a limited number of workers, each with its own state (counters, DB connections, etc.).

#### Requirements
- Processing jobs must run in parallel
- Submitting a processing request must wait if all workers are busy.
- Submission should do load balancing: wait for the first worker to finish, not for a certain one.
- Worker should become available whenever a job is completed successfully, with an exception or cancelled.

Assume the number of workers is not very large (<= 100).

#### Start [(on Scastie)](https://scastie.scala-lang.org/KrOJRxq9SUuGW1I7aPPjQw)

```scala
import scala.util.Random
import scala.concurrent.duration._
import cats._
import cats.implicits._
import cats.effect.{IO, Timer}
import cats.effect.concurrent.Ref

// To start, our requests can be modelled as simple functions.
// You might want to replace this type with a class if you go for bonuses. Or not.
type Worker[A, B] = A => IO[B]

// Sample stateful worker that keeps count of requests it has accepted
def mkWorker(id: Int)(implicit timer: Timer[IO]): IO[Worker[Int, Int]] =
  Ref[IO].of(0).map { counter =>
    def simulateWork: IO[Unit] =
      IO(50 + Random.nextInt(450)).map(_.millis).flatMap(IO.sleep)

    def report: IO[Unit] =
      counter.get.flatMap(i => IO(println(s"Total processed by $id: $i")))

    x => simulateWork >>
      counter.update(_ + 1) >>
      report >>
      IO.pure(x + 1)
  }

trait WorkerPool[A, B] {
  def exec(a: A): IO[B]
}

object WorkerPool {
  // Implement this constructor, and, correspondingly, the interface above.
  // You are free to use named or anonymous classes
  def of[A, B](fs: List[Worker[A, B]]): IO[WorkerPool[A, B]] = ???
}

// Sample test pool to play with in IOApp
val testPool: IO[WorkerPool[Int, Int]] =
  List.range(0, 10)
    .traverse(mkWorker)
    .flatMap(WorkerPool.of)
```

#### Hints

<details>
<summary><strong>Show hints</strong></summary>
<ul>
  <li> Relying on a concurrent queue might be a good idea. And <code>MVar</code> is essentially a one-element queue.</li>
  <li> Because our workers are functions of type <code>A => IO[B]</code>, we can freely do anything effectful before and after running function.</li>
  <li> Our factory method (<code>apply</code> on companion) returns <code>IO</code> too. This lets us create a shared <code>MVar</code> and do pre-initialization, if needed.</li>
</ul>
<details>
<summary><strong>Show heavy spoilers</strong></summary>

Put free workers into <code>MVar</code>. All workers should be free on init. Once they are done processing, <i>guarantee</i> that they put themselves back onto <code>MVar</code>. And we need to NOT wait on that <code>put</code> to complete, so use <code>start</code> and discard the resulting fiber.

</details>
</details>

#### Bonus
- Generalize for any `F` using `Concurrent` typeclass.
- Add methods to `WorkerPool` interface for adding workers on the fly and removing all workers. If all workers are removed, submitted jobs must wait until one is added.

## Race for success
#### Objective
Quickly obtain data which can be requested from multiple sources of unknown latency (databases, caches, network services, etc.).

#### Requirements
- The function should run requests in parallel.
- The function should wait for the first request to complete _successfuly_.
- Once a first request has completed, everything that is still in-flight must be cancelled.
- If all requests have failed, all errors should be reported for better debugging.

Assume that there will be <= 32 providers and they all don't block OS threads for I/O.

#### Start [(on Scastie)](https://scastie.scala-lang.org/SPVZqEbGRSK27nEoxcteXQ)

```scala
import scala.util.Random
import scala.concurrent.duration._
import cats._
import cats.data._
import cats.implicits._
import cats.effect.{IO, Timer, ExitCase}

case class Data(source: String, body: String)

def provider(name: String)(implicit timer: Timer[IO]): IO[Data] = {
  val proc = for {
    dur <- IO { Random.nextInt(500) }
    _   <- IO.sleep { (100 + dur).millis }
    _   <- IO { if (Random.nextBoolean()) throw new Exception(s"Error in $name") }
    txt <- IO { Random.alphanumeric.take(16).mkString }
  } yield Data(name, txt)
  
  proc.guaranteeCase {
    case ExitCase.Completed => IO { println(s"$name request finished") }
    case ExitCase.Canceled  => IO { println(s"$name request canceled") }
    case ExitCase.Error(ex) => IO { println(s"$name errored") }
  }
}

// Use this class for reporting all failures.
case class CompositeException (ex: NonEmptyList[Throwable]) extends Exception("All race candidates have failed")

// Implement this function:
def raceToSuccess[A](ios: NonEmptyList[IO[A]]): IO[A] = ???

// In your IOApp, you can use the following sample method list

val methods: NonEmptyList[IO[Data]] = NonEmptyList.of(
  "memcached",
  "redis",
  "postgres",
  "mongodb",
  "hdd",
  "aws"
).map(provider)
```
#### Hints
<details>
<summary><strong>Show hints</strong></summary>

There are two operators we're interested in: <code>race</code> and <code>racePair</code>. Both run two tasks in parallel, the difference being what happens after one of them is completed. In case of <code>race</code>, the loser is automatically cancelled. In case of <code>racePair</code>, we get to choose what to do, where the still running process is represented by <code>Fiber</code>.

<details>
<summary><strong>Show heavy spoilers</strong></summary>

Using <code>racePair</code>, try folding/reducing the list: race previous result with <code>attempt</code>, then, if we got a successful (as in, <code>Right</code>) result from one, cancel the other and return the result. Otherwise, fall back to the second one, all while accumulating the errors. The result should be something like <code>Either[List[Throwable], A]</code>. Then transform list into an exception and use <code>.rethrow</code> to lift it back to <code>IO</code>.

</details>
</details>

#### Bonus
- If returned `IO` is cancelled, all in-flight requests should be properly cancelled as well.
- Refactor function to allow generic effect type to be used, not only cats' `IO`. (e.g. anything with Async or Concurrent instances).
- Refactor function to allow generic container type to be used (e.g. anything with Traverse or NonEmptyTraverse instances).
  - Don't use `toList`. If you have to work with lists anyway, might as well push the conversion responsibility to the caller.
  - If you want to support collections that might be empty (List, Vector, Option), the function must result in a failing `IO`/`F` when passed an empty value.
