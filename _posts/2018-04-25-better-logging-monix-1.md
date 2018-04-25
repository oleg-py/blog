---
layout: post
title: "Better logging with Monix 3, part 1: MDC"
---
### Problem:
I want to figure out which log entries belong to which request.

Also, being a lazy person, I want to get away with as little work as possible. In particular that means:
- I don't want to pass extra function parameters / implicits (solutions like `Logger.takingImplicit` of scala-logging won't cut)
- I don't want to pollute my domain signatures with something like `ReaderT[Task, RequestId, A]` instead of plain `Task`s.
- I don't want to manually insert request ID to every call to log function. I want to set it once and be done with it.

Java SLF4J API already support all this functionality in form of [MDC](https://www.slf4j.org/api/org/slf4j/MDC.html). However, the existing implementations use a `ThreadLocal` variable, which doesn't work if you're not creating a separate thread per each request - which I do not.

### Solution:
<!--more-->
Subclass your backend's (Logback here) MDC adapter and override everything to use a `Local`:

```scala
import monix.execution.misc.Local
import ch.qos.logback.classic.util.LogbackMDCAdapter

import java.{util => ju}

class MonixMDCAdapter extends LogbackMDCAdapter {
  private[this] val map = Local[ju.Map[String, String]](ju.Collections.emptyMap())

  override def put(key: String, `val`: String): Unit = {
    if (map() eq ju.Collections.EMPTY_MAP) {
      map := new ju.HashMap()
    }
    map().put(key, `val`)
    ()
  }
  
  override def get(key: String): String = map().get(key)
  override def remove(key: String): Unit = {
    map().remove(key)
    ()
  }

  // Note: we're resetting the Local to default, not clearing the actual hashmap
  override def clear(): Unit = map.clear()
  override def getCopyOfContextMap: ju.Map[String, String] = new ju.HashMap(map())
  override def setContextMap(contextMap: ju.Map[String, String]): Unit =
    map := new ju.HashMap(contextMap)

  override def getPropertyMap: ju.Map[String, String] = map()
  override def getKeys: ju.Set[String] = map().keySet()
}
```

Plug it in using reflection:

```scala
// ... somewhere in a main ...
import org.slf4j.MDC
val field = classOf[MDC].getDeclaredField("mdcAdapter")
field.setAccessible(true)
field.set(null, new MonixMDCAdapter)
```

Use MDC like you would normally (example showing Log4s):

```scala
import org.log4s._

// ...somewhere in a method...
for {
  user <- authenticate(token)
  _    <- Task.eval { MDC("user") = user.email }
} yield user
```

Read the current issues section.

### Explanation

Monix 3 has a new feature called "local context propagation", which is essentially `ThreadLocal`s being tied to an execution of a single `Task` instead of a thread. Two APIs are provided:

- monix.execution.misc.Local - a side-effecting fully synchronous version
- monix.eval.TaskLocal - a pure version based on Monix Task.

They are both interfaces to the same context, i.e. both `Local` and `TaskLocal` values get transmitted together. You can also go from `TaskLocal` to `Local` by using `TaskLocal#local` (a mouthful of locals!).

`Local`, due to its synchronous and side-effecting nature, makes a perfect replacement for Java APIs that are using `ThreadLocal`, like Logback's MDC adapter. Unfortunately, MDC is tied to a backend, so we are forced to use reflection to plug in our version.

Most of the code above is straightforward, but there's one caveat that you should be aware of. See how I dance with Java's `emptyMap` below:

```scala
  private[this] val map = Local[ju.Map[String, String]](ju.Collections.emptyMap())

  override def put(key: String, `val`: String): Unit = {
    if (map() eq ju.Collections.EMPTY_MAP) {
      map := new ju.HashMap()
    }
    map().put(key, `val`)
    ()
  }
```

This is required because default values are not immediately set into the local context. Instead, it happens only on first write - and I mean doing `:=` on local, not mutating the value inside! Had I used a mutable map as a default, it would work like a global variable.

### Current issues

Monix Local is not yet fleshed out completely. There are bugs in Monix 3.0.0-RC1:

- Using `executeOn` breaks propagation [(ticket)](https://github.com/monix/monix/issues/612)
  * Workaround: use `Task.shift(ec)` or `Task#asyncBoundary(ec)` for switching execution context
- Local context might not get cleared properly if mutated before async boundaries [(ticket)](https://github.com/monix/monix/issues/624)
  * Workaround: perform a `Task.shift` before reading/writing any locals (e.g. in http4s middleware)


---

As you can see, the local context propagation is very powerful. I'm sure more interesting uses will surface later, but even now we are able to adapt the old-fashioned Java APIs that assume thread-per-request model. Stay tuned for part 2, where I get meaningfull call traces abusing instrumentation because I'm still too lazy to change my business logic for the sake of logging.
