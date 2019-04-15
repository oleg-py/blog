---
layout: post
title: "Traverse your HLists for fun and profit"
---

Surprisingly not mentioned often, there's [Typelevel kittens](https://github.com/typelevel/kittens), a library for typeclass derivation for Cats, which also has few extra things. In particular, there's an ability to `sequence` and `traverse` an HList. It doesn't get enough exposure though, so let's explore few illustrative ways this can be used.

**WARNING:** This post will contain a lot of code. Crazy shapeless aux type-level computation kind of code.
<!--more-->

## Intro
Given `Traverse[F]` and `Applicative[G]`, it's possible to "flip" a value of type `F[G[A]]` into `G[F[A]]` using a `.sequence` operation, e.g. with concrete types:

```scala
def printAndGet(i: Int): IO[Int] = IO {
  println(i.toString)
  i
}

val printedNumbers: List[IO[Int]] = List(printAndGet(0), printAndGet(1), printAndGet(2))
val io: IO[List[Int]] = printedNumbers.sequence
```

and then there's `.traverse`, which is like combined `.map` and `.sequence` in a single pass:

```scala
val numbers = List(0, 1, 2)
val io2: IO[List[Int]] = numbers.traverse(printAndGet)
```

And then there's shapeless with its HLists, which are lists with elements of different types known at compile time (and, size is fixed at compile time too):

```scala
val hlist: Int :: String :: Boolean :: HNil = 0 :: "" :: false :: HNil
```

To "sequence" one of those, like in a simple list case, every element needs to be wrapped in some `F` with an `Applicative` instance:

```scala
val hlist: IO[Int] :: IO[String] :: IO[Boolean] :: HNil =
  printAndGet(5) :: IO(StdIn.readLine) :: IO(Random.nextBoolean()) :: HNil

// needs import cats.sequence._
val allResults: IO[Int :: String :: Boolean :: HNil] = hlist.sequence
```

Neat, huh? No? You can already do this with `.tupled`, and it's less convoluted?
```scala
val allResultsTuple: IO[(Int, String, Boolean)] = (printAndGet(5), IO(StdIn.readLine), IO(Random.nextBoolean())).tupled
```

And in fact, due to fixed size and known types, this `HList#sequence` is closer to `.tupled` than to `Traverse[F].sequence`. Is there a point, then? Or is it a convoluted way to write the same thing whilst also slowing down your compile times?

Well, there _are_ cases where HList beats tuples, because of the following three things you can do with shapeless:

- You can freely convert between case classes and HLists of compatible shapes, as well as between FunctionX and Function1 that takes an HList.
- You can _abstract_ over HLists, whatever their contents or lengths are.
- You also have a bunch of predefined operations, e.g. concatenate two HLists, whatever they are, getting the result or wrap every element of HList in a type constructor, getting the type only.

But enough talk. Let's tackle some problems

### Problem #1 - too lazy to write `apply`.

Suppose I have a bunch of tagless algebras that are very stateful:
```scala
class Fooing[F[_]](
  state1: Ref[F, String],
  state2: Ref[F, Int],
  val isDone: Deferred[F, Unit],
  lock: MVar[F, Unit]
) {
 // Methods are omitted for the sake of brevity
}

class Baring[F[_]: Concurrent](
  state: Ref[F, List[String]],
  signal: MVar[F, Int],
  lock: Semaphore[F]
) { 
  // Methods are omitted for the sake of brevity
}
```

Now, there's quite a few things that require construction in `F`, so I have to define companions with `apply` methods:

```scala
object Fooing {
  def apply[F[_]: Concurrent]: F[Fooing[F]] =
    for {
      state1 <- Ref[F].of("")
      state2 <- Ref[F].of(0)
      isDone <- Deferred[F, Unit]
      lock   <- MVar.empty[F, Unit]
    } yield new Fooing(state1, state2, isDone, lock)
}
```

That was quite verbose and repetitive. Let's use tuple syntax for the other class:
```scala
object Baring {
  def apply[F[_]: Concurrent]: F[Baring[F]] =
    (Ref.of[F, List[String]](Nil), MVar.empty[F, Int], Semaphore[F](0))
      .mapN(new Baring(_, _, _))
}
```

Okay, now that's slightly better. But still, we have some reasonable defaults in most of these cases. Would be nice if we could, you know, avoid specifying every single thing over and over.

Enter Shapeless and Kittens.

```scala
object Fooing {
 def apply[F[_]: Concurrent]: F[Fooing[F]] =
   in[F] autowire (new Fooing[F](_, _, _, _))
}

object Baring {
  def apply[F[_]: Concurrent]: F[Baring[F]] =
    in[F] autowire (new Baring[F](_, _, _))
}
```

Obviously, I just made this `in[F] build` thing up, it's not in the library. So we will make one. And as I'm writing code for humans to use (I hope), I often focus on how nice it is to use and read.

Of course, I already tried several variants, so I know that this one would work :). And it cannot be made much nicer, especially since you can't eta-expand constructors (`new Baring[F] _` isn't valid).

#### Making of autowire
HList is the core structure of shapeless mental model. It helps to think in HList manipulations we want to go through, so let's pick a route:

1. We have a function `(A, B, C, D) => Z` as a parameter (of any arity, not only 4).
2. We need to know how to make `F[A] :: F[B] :: F[C] :: F[D] :: HNil` out of thin air (?)
3. Then we can sequence it to become `F[A :: B :: C :: D :: HNil]`. I'll call it `fArgsList`
4. Then we need to convert our parameter function to accept a single HList, `(A :: B :: C :: D :: HNil) => Z`. That thing, I'll call `listFun`.
5. Then, if `F` is a `Functor`, we just do `fArgsList.map(listFun)` and we're done.

Okay, sounds easy enough. We just need to figure out the (?) bit and, of course, how the hell this is encoded with Shapeless/Kittens.

For (?), let's introduce a custom implicit (lawless typeclass, if you prefer).

```scala
trait Autowire[A] {
  def make(): A
}

object Autowire {
  implicit def mkRef[F[_]: Sync, A: Monoid]: Autowire[F[Ref[F, A]]] =
    () => Ref[F].of(Monoid[A].empty)

  implicit def mkSemaphore[F[_]: Concurrent]: Autowire[F[Semaphore[F]]] =
    () => Semaphore[F](0)

  implicit def mkDeferred[F[_]: Concurrent, A]: Autowire[F[Deferred[F, A]]] =
    () => Deferred[F, A]

  implicit def mkMVar[F[_]: Concurrent, A]: Autowire[F[MVar[F, A]]] =
    () => MVar.empty[F, A]
}
```

#### Traverse that!

Now, if we have an HList type like `F[Ref[IO, Int]] :: F[Ref[IO, String]] :: F[Semaphore[IO]] :: HNil`, we can ask if there's an implicit `Autowire` for every element, giving us, out of thin air, a value of type:
```scala
Autowire[IO[Ref[IO, Int]]] :: Autowire[IO[Ref[IO, String]]] :: Autowire[IO[Semaphore[IO]]] :: HNil
```

So, what do we do with such a list? Turns out, we can use HList's traverse to skip straight to the point 3, yielding us a value of type

```scala
IO[Ref[IO, Int] :: Ref[IO, String] :: Semaphore[IO] :: HNil]
```

All whe have to do is to define the function that we will traverse with. Except it's not our regular Scala FunctionX. It needs to be polymorphic, taking any value of type `Autowire[F[A]]`, for all `F[_]` and for all `A`s, and providing us `F[A]` back, depending on the input value. Such a function doesn't have a common representation in Scala, and this is where Shapeless `Poly` comes into play:

```scala
object AutowireTraverser extends Poly1 {
  //                                 ^ Polymorphic version of Function1 - takes 1 arg only
  implicit def theCase[F[_], A] =
    at[Autowire[F[A]]] { _.make() }
  //  ^ given that     ^ apply this transformation
}
```
#### The code
Now, we can reformulate the task we put in the very beginning, to eliminate all the question marks:

1. We have a function `(A, B, C, D) => Z` as a parameter (of any arity, not only 4).
2. We need to know how what is `F[A] :: F[B] :: F[C] :: F[D] :: HNil`. I'll call this type `ArgsFList`
3. We need to get the implicit `Autowire[F[a]]` for every component of the above list. I'll call it `instances`
4. We can _traverse_ the HList of implicits to become `F[A :: B :: C :: D :: HNil]`. That's our `fArgsList`
5. We still need to convert our parameter function to accept a single HList, `(A :: B :: C :: D :: HNil) => Z`. That thing, I'll call `listFun`.
6. Then, if `F` is a `Functor`, we just do `fArgsList.map(listFun)` and we're done.

So, remember how I said there are operations for a lot of things in Shapeless? The way you use them in generic functions is by requiring an implicit of a certain type. That is also why I listed the steps above in such a verbose way.

We also need to mention every single intermediate HList type as a type parameter. And some of them have to be `<: HList` to meet the constraints of operations we're going to use next.

```scala
def in[F[_]] = new { //                                                      step 1 -v
  def autowire[Func, ArgsList <: HList, ArgsFList <: HList, Instances <: HList, Out](func: Func)(implicit
    fn2p: FnToProduct.Aux[Func, ArgsList => Out],                       // <- step 5, convert our function to Function1 of HList
    fArgs: Mapped.Aux[ArgsList, F, ArgsFList],                          // <- step 2, every value of ArgsList in F gives ArgsFList
    prov: LiftAll.Aux[Autowire, ArgsFList, Instances],                  // <- step 3, get all the implicits
    trav: Traverser.Aux[Instances, AutowireTraverser.type, F[ArgsList]],// <- step 4, ensure we can traverse and it gives F[ArgsList]
    F: Functor[F]                                                       // <- step 6, just a plain Functor, as we need its .map
  ): F[Out] = {
    val instances = prov.instances
    val fArgsList = instances.traverse(AutowireTraverser)
    val listFun = fn2p(func)
    fArgsList.map(listFun)
  }
}
```

> Notice how step 5 is oddly out of order. Why is that? Implicit resolution works, effectively, left-to-right, and therefore the order could make a difference in everything working or not.

Okay, we're done, and it should work. You can see the [code with required imports in Scastie](https://scastie.scala-lang.org/oleg-py/nrAErXNkQGipiCMaDn9QNg/2).

But still, the point of that example is more to show you what you _could_ do. It's up to you (and your teammates) to decide if you _should_. Arguably, for creating several Refs and all using tuple and `mapN` would be easier.

But not so much in the next problem we're about to tackle.

### Problem #2 - recombining Streams, in parallel

Suppose you're crazy enough to use cats-effect and fs2 on Scala.js React frontend. Streams are quite good and natural at representing things that change over time, and with React we can optimize away unnecessary rerenders, so it doesn't really matter if same thing accidentally arrives twice.

But one day (possibly even _day 1_) you need a combinator to combine two streams in parallel - to emit a value when any of two streams emits. In Rx, it's usually named `withLatestFrom` [(illustration)](https://rxmarbles.com/#withLatestFrom). Helpful folks at fs2 gitter tell you that there's an applicative instance for `Signal` which basically exhibits that behavior. And also we can convert any `Stream` to `Signal` using `.hold`. Okay.

Soon you realize that `hold` requires initial value, so you need `holdOption`. And also they return signals inside of singleton streams.

```scala
implicit final class StreamOps[F[_], A](private val self: Stream[F, A]) {
  def withLatestFrom[B](other: Stream[F, B])(implicit F: Concurrent[F]): Stream[F, (A, B)] =
    for {
      sig1 <- self.holdOption
      sig2 <- other.holdOption
      both =  (sig1, sig2).tupled
      ab   <- both.discrete.collect { case (Some(a), Some(b)) => (a, b) }
    } yield ab
}
```

Now this works, so we continue with a refactoring stage. Let's try to simplify the code using several observations.

- Since `Stream` is a monad, it's an `Applicative`, so with `cats.data.Nested` we might be able to work on both layers
- Since `Option` is an applicative too, we can use `_.tupled` instead of pattern matching.
- And, thanks to `cats.FunctorFilter` that basically abstracts over `.collect`, we can use a helper `.mapFilter`, which takes a function of shape `A => Option[B]`, which is exactly what `_.tupled` gives us.

Putting everything together, we get the following:

```scala
implicit final class StreamOps[F[_], A](private val self: Stream[F, A]) {
  def withLatestFrom[B](other: Stream[F, B])(implicit F: Concurrent[F]): Stream[F, (A, B)] =
    (Nested(self.holdOption), Nested(other.holdOption)).tupled.value
      .flatMap(_.discrete).mapFilter(_.tupled)
}
```

Okay, we have completely failed in our attempts to simplify the code. But it's somewhat shorter, so that's better, right? Right?

#### Handling four streams

Anyway, the code works, but you often want even more streams than two. And tuples aren't very beautiful to use, custom case classes are much more readable. So you end up with a boilerplatey code like this:

```scala
case class State(int: Int, string: String, boolean1: Boolean, boolean2: Boolean)

stream1
  .withLatestFrom(stream2)
  .withLatestFrom(stream3)
  .withLatestFrom(stream4)
  .map { case (((a, b), c), d) => State(a, b, c, d) }
```
That sucks, and I hate writing code that sucks. Can shapeless help us there? Yes, very much so. Let's first imagine a perfect syntax for our use-case:

```scala
val stream: Stream[F, State] = combine[State].from(stream1 :: stream2 :: stream3 :: stream4 :: HNil)
// Or, even better
combine[State].from(stream1, stream2, stream3, stream4) // some kind of varargs-to-hlist?
```

Obviously, that call should compile if and only if such combination is possible. It will be order-sensitive so we don't mess up cases we can't sensibly distinguish between, like those Booleans.

Remember, in the very beginning I said `tupled` is pretty much the same as `sequence`ing an HList? That's why we are going to break down version 2 and use it as a base of our generalization:

1. We get an HList of Streams, and we also know which type A we're trying to build. I'll call this `streams` of type `Streams`
2. We need to map every element of the HList with `s => Nested(s.holdOption)`.
3. We sequence the resulting HList, to get a Stream of a Signal of an HList of Options.
4. We unwrap the resulting Nested and do the same `flatMap(_.discrete)` to get a plain Stream of an HList of Options.
5. We `_.sequence` the inner HList of an Options, and do it in `mapFilter`, so we can just drop `None` values. We get a stream of an HList, of type `fs2.Stream[F, Repr]`.
6. We reconstruct the `A` from each `Repr` that is in the stream, giving us `fs2.Stream[F, A]`, just like we wanted.

And, similar to regular lists, `hlist.map(f).sequence` is same as `hlist.traverse(f)`, and might even be somewhat lighter on compiler and runtime. So, let's fuse steps 2 and 3 together.

Now, we need is to know how we do step 6. And that is done using the most common class you see in any shapeless tutorial - `Generic`. That's the thing that encodes the relationship between a case class and a certain HList, equipped with two methods - `to`, to go from case class to HList.

Finally, there _is_ a way to get that varargs-to-HList magic. For this, we need to have a method called `xxxProduct` that takes an HList a parameter, in a class that extends a `ProductArgs` trait. Then, calls to `.xxx(a, b, c)` will be rewritten in compile-time to calls to `.xxxProduct(a :: b :: c :: HNil)` - exactly what we wanted!.

So, let's put everything together:

```scala
object combine {
  def apply[A]: CombineCurried[A] = new CombineCurried[A]

  // Function to traverse with
  object holdFn extends Poly1 {
    implicit def holdStream[F[_]: Concurrent, A] =
      at[Stream[F, A]](s => Nested(s.holdOption))
  }

  class CombineCurried[A] extends ProductArgs {
    def fromProduct[F[_], Streams <: HList, Opts <: HList, Repr <: HList](streams: Streams)(implicit
      tr: Traverser.Aux[Streams, holdFn.type, Nested[Stream[F, ?], Signal[F, ?], Opts]],  // fused 2-3
      seq2: Sequencer.Aux[Opts, Option, Repr],                                            // 5
      gen: Generic.Aux[A, Repr]                                                           // 6
    ): Stream[F, A] =
      streams.traverse(holdFn).value
      .flatMap(_.discrete)
      .mapFilter(_.sequence)
      .map(gen.from)
  }
}
```

It works, and you can play with it [on Scastie](https://scastie.scala-lang.org/oleg-py/BBnpyc0aQdy5Wgt20DPUXA/15)

### Questions
##### WTF is `new { def apply[...] }`?
In Java, you can create anonymous objects with an extra method:
```java
var myObject = new Object() {
  public void newMethod() { System.out.println("I'm a new method"); }
};
myObject.newMethod();
```

However, there's no real way to give this variable a static type. And before Java introduced `var` keyword, you couldn't even do it locally. However, this works in old versions of Java:

```java
new Object() {
  public void newMethod() { System.out.println("I'm a new method"); }
}.newMethod()
```

Now, in Scala, you can assign this value a structural type. Also, scala allows you to omit Object.

```scala
val object: { def method(): Unit } = new {
  def newMethod() = println("I'm a new method")
}

object.method()
```

And in fact, Scala's inferencer can figure out structural types as well, so the type ascription above is redundant.

It's not something I recommend in production code, esp. running on JVM, as it uses reflection and therefore causes unnecessary slowdowns, but for quick one-offs, test fixtures and so on it's quite good. Just don't use it on your hot path.

##### A semaphore of capacity 0? Is that even useful?
Yep. You can add more permits to the semaphore than it started with. That 0 isn't an _upper bound_, but starting quantity. You can add e.g. extra 100 permits by doing `semahore.releaseN(100)`

##### What if I want some MVars to not be empty? What if I have plain value parameters, or fs2 Topics, or...
You can do wiring by defining an implicit, if you need a lot of it:
```scala
implicit def mkTopic[F[_], A: Monoid]: Autowire[F[fs2.Topic[F, A]] = () => Topic(Monoid[A].empty)
```
By implicit resolution rules, this would take precedence over whatever is defined in companion of `Autowire`. You can also use partial application AND regular flatMap/flatTap for further customizations:

```scala
Topic[F, String]("Started") flatMap { topic =>
  in[F] autowire (new MyClass(_, topic, _, 42))
} flatTap { instance =>
  instance.semaphore.releaseN(4)
}
```

But, I'm not saying that because you can use that technique, you should. Prefer regular instantiation where feasible. I came up with this while looking for an option to purely initialize 12 refs and 2 topics, which _is_ a bit of a pain, but I'm still not sure I'm going to use it.

##### Why are there so many type parameters?
In short, we need to link the intermediate results between each other. Ideally, we would like to write something like:

```scala
def fromProduct[F[_], Streams](streams: Streams)(implicit
      tr: Traverser[Streams, holdFn.type, Nested[Stream[F, ?], Signal[F, ?]]]
      seq2: Sequencer[tr.Result, Option],
      gen: Generic[A],
      backwardsLink: gen.Repr =:= seq2.Result
    ): Stream[F, A]
```

The catch here is that in Scala, you can't have path-dependent types in one parameter list depending on values in the same parameter list, and you also can't have multiple implicit parameter lists.

In a future Scala version, that might be very much possible. There [were already efforts](https://github.com/scala/scala/pull/5108) to fix the latter problem. But for now, we need to lift every intermediate result into a type parameter and use `Aux`es to link them together.

##### Why is there no return type ascription on Poly case? Shouldn't implicit defs always have return types?
It's not strictly necessary and, if done incorrectly, can ruin the inference. OTOH, sometimes (e.g. in problem 1) it helps.

For example, this erases important type information about the result type:
```scala
object holdFn extends Poly1 {
  implicit def holdStream[F[_]: Concurrent, A]: Case[Stream[F, A]] =
    at[Stream[F, A]](s => Nested(s.holdOption))
}
```

And the proper type would be:
```
Case.Aux[Stream[F, A], Nested[Stream[F, ?], Signal[F, ?], Option[A]]
```

I just don't bother, treating these as a mini-DSL.


##### Why can you avoid requiring `Concurrent[F]` everywhere?

When we require a `Traverser` for a certain `Poly`, the resolution of said `Traverser` - at the call site - has already required `Concurrent`. So, it's redundant. That also means you can use `Autowired` with only `Sync[F]` if all you construct are `Ref`s.

You can, however, require `Concurrent[F]` too. It will make no difference if the method call can compile, but will make one if it doesn't - the compile error about implicit not found would mention it, and not just the `Traverser`. Not a big improvement, but it might be worth it.

##### Shapeless is still too magic for me to work it out.

I can relate to that sentiment. It takes time to learn to work with shapeless, and even then it can take frustratingly long to come up with working code. While the article might give impression that I wrote everything in several minutes, each shapeless snippet took several hours of trial and error, and making sure my desired use cases compile and work as I want. All I can say is - don't give up if you think it's worth your time. Shapeless is not easy, but it lets you do awesome things as well in a very type-safe fashion without dropping down to writing macros by hand.

### More reads
- [Question on SO](https://stackoverflow.com/questions/11825129/are-hlists-nothing-more-than-a-convoluted-way-of-writing-tuples) about the point of HLists.
- [Shapeless Guide](https://books.underscore.io/shapeless-guide/shapeless-guide.html), a book by underscore.io
