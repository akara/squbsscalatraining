## Day 3: Intro to Akka Streams

### Core Concepts

* Source - Source stream components
* Flow - Processing components
* Sink - End stream components
* Materializer - Engine that makes the stream tick
* Materialized Values - The resulting value of a stream
* Back-pressure

### Writing your First Stream

We start with writing our first, very simple stream. Again, we'll expand on this first application later to build a more sophisticated, and more useful stream. You do not necessarily have to build a stream inside an `Actor`, but we will as well start with that.

Create Scala class `PaymentStreamActor` into your package of choice under `src/main/scala` of your cube project with the following content:

```scala
import akka.actor.{Actor, ActorLogging}
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Source

class PaymentStreamActor extends Actor with ActorLogging {

  implicit val mat = ActorMaterializer()

  val src = Source(1 to 100)

  override def receive: Receive = {
    case _ => run()
  }

  private def run() = src.runForeach(log.info("Count: {}", _))
}
```

And to make it tick, we also want to write our test code. Create `PaymentStreamSpec` into your package of choice under `src/test/scala` of your cube project with the following content:

```scala
import akka.actor.{ActorSystem, Props}
import akka.testkit.{ImplicitSender, TestKit}
import org.scalatest.{FlatSpecLike, Matchers}
import org.scalatest.concurrent.ScalaFutures

class PaymentStreamSpec extends TestKit(ActorSystem("PaymentStreamSpec"))
  with FlatSpecLike with Matchers with ImplicitSender with ScalaFutures {

  val streamActor = system.actorOf(Props[PaymentStreamActor])

  "The stream actor" should "Accept a ping request" in {
    streamActor ! "ping"

    // Just temporarily, we just want to see the results
    Thread.sleep(1000)
  }
}
```

Right-click on your test class name `PaymentStreamSpec` in the editor and select option `Run 'PaymentStreamSpec'` from the pop-up menu. See your test results. Since our actor does no validations but only logs, we can observe it running by looking at the logs. You should see it logging the numbers 1 to 100. You got your first stream!

### Adding the Sink

So far, we used a shortcut method `runForeach` that installs a sink and just runs it in one shot. There are a lot of such shortcut methods in Akka Steams to make sure you do not have to write more code than needed. But now lets make it all very clear. Add a sink just below where the source was declared:

```scala
  val src = Source(1 to 100)
  val sink = Sink.foreach[Int](log.info("Count: {}", _))
  // Add this line ^^^^^^^^^^
```

Next we also want to formalize the run a bit better for our own understanding. Lets re-write the `run()` method to be as followings:

```scala
  private def run() = {
    val stream = src.to(sink)
    stream.run()
  }
```

Here we can now see that we essentially create a runnable stream graph `RunnableGraph` where we don't care about the materialized value.

### Adding a Flow component

Now that we have the source and sink components, lets add some processing to the middle of the stream. First we'll only take our even numbers using the filter. Lets create a filtering flow:

```scala
  val src = Source(1 to 100)
  val filter = Flow[Int].filter(_ % 2 == 0)
  // Add this line ^^^^^^^^
  val sink = Sink.foreach[Int](log.info("Count: {}", _))
```

Then lets put our filter to work. Modify our `run()` method as follows:

```scala
    val stream = src.via(filter).to(sink)
    stream.run()
```

And run it again. Now you'll notice half the numbers are no longer printed.

We can also add another stream component, another flow, to calculate the square of each number. Lets do so.

```scala
  val filter = Flow[Int].filter(_ % 2 == 0)
  val square = Flow[Int].map(i => i * i)
  // Add this line ^^^^^^^^^
  val sink = Sink.foreach[Int](log.info("Count: {}", _))
```

Language note: We have been using the shortcut `_` meaning "it" in the `filter`, but we use the full form in the `map`. The shortcut does not serve us well in this case. While we can see some cases in the Scala language using `(_ * _)`, the meaning is different. The first `_` means first input to the closure while the second `_` is the second input. In the shortcut form, each input can only be used only once. Since we do not follow this rule, we need to use the full closure form `(i => i * i)`

Then we also add the `square` to the stream.

```scala
  val stream = src.via(filter).via(square).to(sink)
```

And just run it. Well, there is nothing unexpected to the results.

Here we just created two separate flow components and then composed them together. We could as well define all these as a single component, such as:

```scala
  val squareEven = Flow[Int].filter(_ % 2 == 0).map(i => i * i)
```

This definitely works. But the filter and square components are no longer easily re-usable. It may be worth grouping some complex logic together this way, though.

But, we have just learned one more thing. A stream is not always one in and one out. On the contrary, the number of elements might shrink (with `filter`) or might grow from processing stage to processing stage (for instance, with `mapConcat` or `flatMapConcat` stages).

### Stream Components are Re-usable

In the later examples, you can clearly see we have separated the declaration of the stream to the running of the stream with the `stream.run()`.

It is to be noted that each of the stream components by themselves are reusable. They can be shipped around in any form, returning from a method call, sent around via actor messages, etc. These logic components can then be composed into the final runnable form and run just about anywhere. Even the final `RunnableGraph` is reusable and can be shipped around. We can think of them as templates of stream logic that is being shipped around and manipulated until finally they get to run via the `run` command. The `src.runForeach` call we had in the beginning is just a shortcut of all this.

### Stream Composition & Componentization

Stream components are composable to build more complex components. Each of them can then be shipped around and built into a final `RunnableGraph`. It is important to understand the result types of such composition, though.

* `Source ~> Flow` ==> `Source`
* `Flow ~> Flow` ==> `Flow`
* `Flow ~> Sink` ==> `Sink`

Lets try out some declarations here. Note: We show the type annotations to clearly display the resulting types of the composition. In your code, this can be inferred by the Scala compiler and may not be needed:

```scala
  val filteredSource: Source[Int, NotUsed] = src.via(filter)
  val filterAndSquare: Flow[Int, Int, NotUsed] = filter.via(square)

  // Use materialized value of flow
  val squareAndLog: Sink[Int, NotUsed] = square.to(sink)

  // Choose materialized value of sink
  val squareAndLogMat: Sink[Int, Future[Done]] = square.toMat(sink)(Keep.right)
```

Each of these now become more complex stream components that also can be shipped around and re-used. This is very handy in declaring and composing more and more complex components that will become part of the main flow.

Language note: You will notice two sets of arguments to `square.toMat`. The first is sink, separated by a different parenthesis from `Keep.right`. This is called "currying" and is often used in functional languages. In essence, passing `sink` to `toMat` creates another function that takes the `Keep` value to produce a real `Sink`. A function that returns another function.

### Materialized Value

The materialized value is the resulting value of a stream when it runs. In our examples so far, we really did not care about the materialized value just yet. but lets say we wanted to calculate the sum of squares as a result of the stream, we can do the `log` as a map stage separately. Then we can have a sink that sums it. Lets do that:

```scala
  val logFlow = Flow[Int].map { i =>
    log.info("{}", i)
    i
  }
  
  val sum = Sink.reduce[Int](_ + _)
```

Language notes:

* In this `map` example, you see the use of curly braces `{` and `}`. In Scala, these curly braces mean the same as normal braces `(` and `)` except they can be used for multi-line closures as you can see here. The last expression `i` is the return value of that closure.
* Here you see the reduce example using the closure `(_ + _)`. This expands to `((a, b) => a + b)`. The second `_` represents the second input value, and so on. That makes it very simple to write and read `reduce` or `fold` closures. When it gets any more complex, use the standard and not this shortcut closure syntax.

Then our run method will just be like this:

```scala
    val stream = src.via(filter).via(square).via(logFlow).toMat(sum)(Keep.right)
    stream.run()
```

Hold on, we just summed up the result of the stream. But where did it go? Of course, that sum would only be available once the stream is done running. That's why we get a `Future[Int]`. We can wait for that. But remember, waiting **is a crime** in this architecture. So we need to be more creative. We can do many things with that `Future` but for now we want to send it back to the test that asked for it. Lets use one of the actor features we have not covered so far, the `pipeTo`. Lets change our `run` method as follows:

```scala
  private def run() = {
    val stream = src.via(filter).via(square).via(logFlow).toMat(sum)(Keep.right)
    stream.run().pipeTo(sender())
  }
```

Now lets change our test case to expect that result back. Go to our test and change it as follows:

```scala
  "The stream actor" should "Accept a ping request" in {
    streamActor ! "ping"
    val sum = expectMsgType[Int]
    sum shouldBe 171700
  }
```

We can also now take out the `sleep` at the end. As this test won't quit until it receives the response, we no longer have to compensate for test quitting before the test logic is done.

Now run the test and have some fun.

### Graph

So far, we have been dealing only with relatively linear streams. But most applications, stream segments or components cannot be linear. The stream itself does not at all need to be linear. And while it is possible to compose non-linear streams using the current stream syntax, it can become hard to deal with. That's where the stream graph syntax comes in very handy.

The graph is defined by a GraphDSL, which introduces a certain boilerplate as follows:

```scala
    val graph = RunnableGraph.fromGraph(GraphDSL.create(sum) { implicit builder =>
      out =>
        import GraphDSL.Implicits._

        val bCast = builder.add(Broadcast[Int](2))
        val merge = builder.add(Merge[Int](2))

        src ~> bCast ~> filter ~> merge ~> logFlow ~> out
               bCast ~> square ~> merge
        
        ClosedShape
    })
```

The `GraphDSL` block looks a little fuzzy, but well worth it when it comes to the stream graph declaration itself. Lets break it down into the components:

1. The declaration and the factory. The case above shows a `RunnableGraph`, which means this graph is a full template and can be run immediately. However, a `GraphDSL` does not always need to create a `RunnableGraph`. It can as well create a `Flow`, `Source`, or `Sink`. These can again be composed with other components using `GraphDSL` or simple stream operations. To construct a `RunnableGraph` from a `GraphDSL`, you'd construct it with `RunnableGraph.fromGraph(...)`. Similarly, to construct a `Flow`, you'd start with `Flow.fromGraph(...)`. Similar methods exist for `Source.fromGraph(...)` and `Sink.fromGraph(...)`.

2. The graph itself is created by `GraphDSL.create(...)`. The `GraphDSL.create(...)` is a highly overloaded method. It can take no arguments to a very large number of arguments. The format shown in this example passes only the `sum` sink in. By doing so, we're saying we want this `RunnableGraph`/`Flow`/`Source`/`Sink` to use the materialized value of `sum` as the materialized value of this graph. If no parameter is passed, the materialized value is simply a `Future[Done]`.

   Life gets more interesting when we pass multiple stream components into `GraphDSL.create(...)` In that case we have multiple materialized value for the graph. This is of course not valid. So we need to pass another closure in called `combineMat`. This lambda will take *n* arguments where *n* is the number of stream stages passed in. The output is then the materialized value based on combining the input materialized values. This allows and enforces developers to define how to produce the final materialized value from the multiple values in question. The easiest case is to create a tuple to return each of the materialized value as part of the result.

   The last argument of the `GraphDSL.create` is the builder lambda. This is a layered lambda, the first one takes the `builder` as an argument. We need to mark this builder `implicit` to be used further below in the graph. The next lambda in takes an imported version of the stream stages passed into `GraphDSL.create`. For instance, if you have 3 stages passed in, you'll have 3 inputs in this lambda representing those imported stages. These will be used to compose the stream graph. In this case the `out` represents the `sum` passed in.
   
3. `builder.add(...)` calls. Every non-linear stream stage not passed in (and hence no materialized value captured) can be made available to the composition by passing it to the `builder.add(...)` calls. Linear stages from outside this block are added implicitly so we have to only worry about our non-linear stages. 
4. The graph composition. We go through all the trouble just for this alone. This is where you describe how the stages/components connect to each others.

5. The return value. These need to match the type of graph you're composing. Depending on the type of composition, you want to return the relevant shape for that composition. Here is the return value for common stage types:

   | Type            | Return value                          |
   | --------------- | ------------------------------------- |
   | `RunnableGraph` | `ClosedShape`           |
   | `Source`        | `SourceShape(outputPort)`          |
   | `Sink`          | `SinkShape(inputPort)`             |
   | `Flow`          | `FlowShape(inputPort, outputPort)` |


Enough said. Lets get some code going. We'll create a new Scala class that looks very much like our `PaymentStreamActor`. Lets call it `PaymentGraphActor`:

```scala
import akka.actor.{Actor, ActorLogging}
import akka.pattern._
import akka.stream.{ActorMaterializer, ClosedShape, FlowShape}
import akka.stream.scaladsl.{Broadcast, Flow, GraphDSL, Keep, Merge, RunnableGraph, Sink, Source}

class PaymentGraphActor extends Actor with ActorLogging {
  implicit val mat = ActorMaterializer()
  import context.dispatcher

  val src = Source(1 to 100)
  val filter = Flow[Int].filter(_ % 2 == 0)
  val square = Flow[Int].map(i => i * i)
  val logFlow = Flow[Int].map { i =>
    log.info("{}", i)
    i
  }

  val sum = Sink.reduce[Int](_ + _)

  override def receive: Receive = {
    case _ => run()
  }

  private def run() = {
    val graph = RunnableGraph.fromGraph(GraphDSL.create(sum) { implicit builder =>
      out =>
        import GraphDSL.Implicits._

        val bCast = builder.add(Broadcast[Int](2))
        val merge = builder.add(Merge[Int](2))

        src ~> bCast ~> filter ~> merge ~> logFlow ~> out
               bCast ~> square ~> merge
        
        ClosedShape
    })

    graph.run().pipeTo(sender())
  }
}
```

And for that, we also create the test `PaymentGraphSpec` class:

```scala
import akka.actor.{ActorSystem, Props}
import akka.testkit.{ImplicitSender, TestKit}
import org.scalatest.concurrent.ScalaFutures
import org.scalatest.{FlatSpecLike, Matchers}

class PaymentGraphSpec extends TestKit(ActorSystem("PaymentGraphSpec"))
  with FlatSpecLike with Matchers with ImplicitSender with ScalaFutures {

  val graphActor = system.actorOf(Props[PaymentGraphActor])

  "The graph actor" should "Accept a ping request" in {
    graphActor ! "ping"
    val sum = expectMsgType[Int]
    sum shouldBe 340900
  }
}
```

### `BidiFlow` - Bidirectional Flows

In many cases, it is useful to model a stream or stream component as bidirectional flows. More commonly, protocol stacks can be modeled as bidirectional flows. Even the squbs pipeline in 0.9 is modeled as a set of `BidiFlow`s stacked on top of each others.

###### Anatomy of a `BidiFlow`

```scala
        +------+
  In1 ~>|      |~> Out1
        | bidi |
 Out2 <~|      |<~ In2
        +------+
```

###### Stacking of `BidiFlow`s

```scala
       +-------+  +-------+  +-------+
  In ~>|       |~>|       |~>|       |~> toFlow
       | bidi1 |  | bidi2 |  | bidi3 |
 Out <~|       |<~|       |<~|       |<~ fromFlow
       +-------+  +-------+  +-------+

       bidi1.atop(bidi2).atop(bidi3);
```

But, the `BidiFlow` is not only useful for "literally" bidirectional flows. In some cases we need to gate off a part of the flow with a single entry and exit point of control at those places. `BidiFlow`s are extremely useful in such cases. The following squbs flow components are `BidiFlow`s:

* TimeoutBidiFlow
* CircuitBreakerBidi

As we can see, these stages fend off a section of the stream to test whether a message passed through that stream part in timely or not, in error or not. It also has the capability to short circuit the flow with alternatives paths (like timeout messages) at its entry/exit point as can be seen in the following sample:

```scala
       +---------+  +------------+
  In ~>|         |~>|            |
       | timeout |  | processing |
 Out <~|         |<~|            |
       +---------+  +------------+
       
       timeout.join(processing);
```

Here, timeout is a `BidiFlow` where processing is just a regular flow stage (or combination of flow stages). The arrow is just pointed backwards to make it connect. It only has one input and one output port.

### Stream Stages

So far we have shown only a few most common stages, such as `Flow.map`, `Flow.filter`, `Broadcast`, `Merge`, etc. Akka Streams comes with a multitude of stages. Lets point your browser to [Overview of built-in stages and their semantics](https://doc.akka.io/docs/akka/current/stream/operators/index.html?language=scala) and look at some of the many stages provided by Akka.

If the Akka-provided stages are not adequate, there are more stages provided as part of [Alpakka](https://developer.lightbend.com/docs/alpakka/current/). squbs (listed in the Alpakka catalog) also provides several interesting stream stages:

* PersistentBuffer (with and without commit stage)
* BroadcastBuffer (with and without commit stage)
* Timeout
* CircuitBreaker

#### Custom Stages

If stages we can find still do not fit requirements, we can build our own stage. This is documented at [Custom stream processing](https://doc.akka.io/docs/akka/current/stream/stream-customize.html?language=scala). Since testing and qualification of custom stream stages are pretty elaborate, plese engage the squbs team. Also, if you think your requirements are more generic than just your use case, squbs always welcomes your contributions!

### Stream Cookbook

For ideas how to satisfy your requirements with Streams, the [Streams Cookbook](https://doc.akka.io/docs/akka/current/stream/stream-cookbook.html?language=scala) is an invaluable resource to get your ideas.
