## Day 2: Functional Composition

### Container Types
* We deal with container types every day: `List[T]`, `Set[T]`
* Even an array is an `Array[T]`
* Common container types we will need: `Option[T]`, `Future[T]`
* A `List[T]` or `Set[T]` contains 0 or more elements
* You can think of an `Option[T]` as a collection containing 0 (empty) or 1 element. `Option[T]` has two subtypes: `Some[T]` or `None`
* A `Future[T]` contains an element or an error that may be filled in the future.

### Transformation, like how we used to do it in Java

```scala
  def intListToString(intList: List[Int]): List[String] = {
    val stringBuffer = new ArrayBuffer[String](intList.size)
    intList.foreach { i =>
      stringBuffer += "count: " + i
    }
    stringBuffer.toList
  }
```

### No! We can do better!
* Given a `List[Int]` and a function that transforms `Int` => `String`, we should be able to apply this transformation to the whole `List` in one shot.

Lets take a brief look at some Scala code (easier to understand):

```scala
def intListToString(intList: List[Int]) = intList.map(i => "count: " + i)
```

So this converts a `List(1, 2, 3)` to a `List("count: 1", "count: 2", "count: 3")`.

### Some theory on `map` operation

* `List`, `Set`, etc. are container types.
* Lets container type: `C` and the elements inside the container type: `E`.
* We can now say that container is of type `C[E]`
* If we apply a function of type `E => F`, the result will be of type `C[F]`
* This application is called the `map` operation.
* So: `C[E].map(E => F) =>> C[F]`

This should be easy!

### Compose using `flatMap`

* For type `C[E]`, if we apply function `E -> C<F>`, what will we get?
* More concrete example: If we have `List[String]` and a function `String => List[Char]` i.e. split the `String` into a list of characters, what will a `map` operation do?
* Answer, `List[List[Character]]`
* Or `C[C[F]]` based on our notation
* So what if we want a list of all characters in all strings? We have to `flatten` it.
* Here's where `flatMap` comes in handy.
* So, in theory: `C[E].flatMap(E => C[F]) =>> C[F]`
* We flatMap the container `C[E]` with a container type `C[F]` and get a new combined result of type `C[F]`.

Clear?

### Try to `flatMap` two lists:

Using the Scala REPL, we can show the effects of `flatMap` vs `map` in the following examples:

```scala
scala> val l1 = List("one", "two", "three")
l1: List[String] = List(one, two, three)

scala> val l2 = l1.flatMap(s => s.toList)
l2: List[Char] = List(o, n, e, t, w, o, t, h, r, e, e)

scala> val l3 = l1.map(s => s.toList)
l3: List[List[Char]] = List(List(o, n, e), List(t, w, o), List(t, h, r, e, e))
```
### Combining with `map` and `flatMap`

* What if we want to operate on multiple containers together, combining them.
* Given containers `C[E]` and `C[F]`, we want to apply a function `(E, F) => T` and get a `C[T]` out of it.
* The combining would look like this: `C[E].flatMap(E => C[F].map(F => [(E, F) => T]))`
* Similarly, combining `C[E]`, `C[F]`, `C[G]` applying `(E, F, G) => T` looks like this: `C[E].flatMap(E -> C[F].flatMap(F => C[G].map(G => [(E, F, G) => T])))` and gives you a `C[T]`.

### Combining & Processing example with `Optional`

```scala
    def combine(o1: Option[String], o2: Option[String]): Option[String] =
      o1.flatMap(s1 => o2.map(s2 => s1 + s2))
```

* Now try `combine(Option("Hello"), Option("World"))`
* And try to make one or the other empty.

### Composing using for-comprehensions

While you may have seen some use of `for` in Scala looking like a loop construct, it is actually not. For-comprehension is actually syntactic sugar to allow easier composition/combination of `map` and `flatMap` described in the previous sections. So our combine function above can be re-written like this:

```scala
    def combine(o1: Option[String], o2: Option[String]): Option[String] =
      for {
        s1 <- o1
        s2 <- o2
      } yield s1 + s2
```

Yes, you spend a few more lines. But the resulting code is much more readable than the flatMap and map composition. The resulting compiled code is exactly the same. This is becoming even more apparent when you have a large number of containers to composition as can be seen in the following composition of 4 containers:

```scala
  for {
    s1 <- o1
    s2 <- o2
    s3 <- o3
    s4 <- o4
  } yield s1 + s2 + s3 + s4
```

Lets try to write this using `flatMap` and `map` and we'll clearly see how much more effort it is.

### Combining Futures

* Why? Because you want to combine futures and not block on each or any future.
* You need to operate on the content of the future.
* And Future is just another container type.
* Same composition rules applies.

### Using Combined Futures in Actors

* Async `Future` operations need to use the `pipe` operation to send an actor message to this or a different actor, without blocking. Ba careful never to block on

### Exercise:

Write a test case that calls `PaymentActor` by `ask` multiple times (multiple `PaymentRequests`) and combine the resulting `Future`s into a single one. Then block on this resulting `Future` to finish the test case using the `Await` call. The `Future` should contain a `List` of all results.
