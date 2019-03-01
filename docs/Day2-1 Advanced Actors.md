## Day 2: Advanced Actors

### Using `ask`

So far we have learned about `tell`, the fire-and-forget sender, and `forward`. There is another way to send an actor a message and expect a response back, even in production and not test code, by using `ask`. Lets add another test using just this to `PaymentActorSpec`:

First we need to deal with a few imports:

```scala
import akka.pattern._
import org.scalatest.concurrent.ScalaFutures
import scala.concurrent.duration._
```

The `import akka.pattern._` enables the use of the ask `?`, and `maptTo` we'll discuss below. And since ask deals with Scala `Future`s the `ScalaFutures` trait above helps with testing futures. The `scala.concurrent.duration._` import provides convenient ways to deal with time, like `3.seconds` in the example below.

In addition, we have to mix the `ScalaFutures` trait into the test class:

```scala
class PaymentActorSpec extends TestKit(ActorSystem("PaymentActorSpec"))
  with FlatSpecLike with Matchers with ImplicitSender with ScalaFutures {
```

Then the test itself:

```scala
  "The Payment Actor" should "approve small payment requests through ask automatically" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000))
      .mapTo[PaymentResponse]
    id += 1
    responseF.futureValue.status shouldBe Approved
  }
```

Wow, this looks really short. Lets cover this test case using ask one-by-one:

1. We set a timeout, implicitly. Since ask `?` returns a future, we cannot wait for this future indefinitely. The timeout tells how long ask should wait. The `3.seconds` ge=ot converted to the `Timeout` type implicitly, too.
2. The ask, or `?` itself. Since our actors are not typed, it won't know what the response type of the `PaymentActor` would be. So it returns a `Future[Any]`
3. `.mapTo[PaymentResponse]` casts the `Future`s value to the target type.
4. The `futureValue` operation we use in the validation is a facility provided by ScalaTest's `ScalaFutures` trait. It does indeed block. But by using a ScalaTest trait here we ensure that such blocking code would only be confined to tests and does not leak into production code. Production code does not have access to ScalaTest.

The `Future` may succeed or fail. To signal a failure to an `ask`, the actor must respond back with a `akka.actor.Status.Failure`. Lets try that with our `matchAny` case. Modify `PaymentActor`'s `matchAny` to send back a `Failure` when that case is matched:

Once again, the import:

```scala
import akka.actor.Status
```

Then in the `PaymentActor` modify the `matchAny` to be as follows:

```scala
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
      sender() ! Status.Failure(new IllegalArgumentException("Message type not understood"))
      // ^^^ This line added ^^^
```

Lets try this out in our test. Lets add a test that checks for such failure to `PaymentActorSpec`:

```scala
  "The Payment Actor" should "fail the future with the right exception" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = paymentActor ? "Pay me!"
    responseF.failed.futureValue shouldBe an [IllegalArgumentException]
  }
```

The only notable part of this test is that we use the `.failed` projection of a `Future` to validate the future has failed with a certain exception.

### Changing Actor Behavior

Now we come to the last property of actors. It can change behavior based on a message. Lets try having an `offline` behavior for our actors. If we send an `Offline` message to our `PaymentActor`, it will only send an error back. Once we send an `Online` message to the actor, it will start resume processing. Here we change the state of the actor to off-line and have it behave differently in offline mode. For that, we need our `Offline` and our `Online` messages first. Lets also make them singletons to save on any GC cost. They are just signals and don't contain state.

```scala
case object Offline
case object Online
```

Next, lets add a Receive matcher to our `PaymentActor` to build the offline behavior. This is a final variable to our receive and should never change:

```scala
  def offlineBehavior: Receive = {
    case Online =>
      context.unbecome()
      sender() ! Online
    case _: PaymentRequest =>
      log.error("Received PaymentRequest but still Offline")
      sender() ! Offline
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
      sender() ! Status.Failure(new IllegalArgumentException("Message type not understood"))
  }
```

Add the receive match for `Offline` to take this actor to its `offlineBehavior`. This match can go anywhere before the `rcvBuilder.matchAny`:

```scala
    case Offline =>
      context.become(offlineBehavior, discardOld = false)
      sender() ! Offline
```

The `context.become(...)` call moves the actor to `offlineBehavior`. And we can see in the `offlineBehavior` the `context.unbecome()` will set the behavior back to original. If the `discardOld` parameter to `context.become(...)` is set to `true`, we can only keep shifting to new behavior. This is to prevent memory leaks caused by behavior change.

Lets build a test for this. This test is a bit longer, though. It checks the behavior before going off-line, while off-line, and after coming back on-line. Here is the test code:

```scala
  "The Payment Actor" should "respond only with `Offline` while offline" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF.futureValue.status shouldBe Approved

    // Now take the actor offline
    val responseF2 = paymentActor ? Offline
    responseF2.futureValue shouldBe Offline

    // Lets try to send in 5 payment requests. Each should return Offline
    (0 until 5).foreach { _ =>
      val responseF3 = paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)
      id += 1
      responseF3.futureValue shouldBe Offline
    }

    // Next we turn the PaymentActor back on-line
    val responseF4 = paymentActor ? Online
    responseF4.futureValue shouldBe Online

    // Now we should be back in business
    val responseF5 = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF5.futureValue.status shouldBe Approved
  }
```

### Stashing Messages

In some cases we don't just want to say we're offline. We want to stash the received messages till we're back online and deal with them at that time. This is often used in actors that act as caches. There is a short time while loading/reloading caches we just want to hold onto requests until our data is fully loaded.

For this exercise, we're going to replace some logic we have built. In order to deal with this scenario, we want to make a copy of our `PaymentActor` into a new class called `StashingPaymentActor`. You'll need to copy and modify all references to `PaymentActor` to `StashingPaymentActor`.

We will then mix in the `Stash` trait to our `StashingPaymentActor`:

```scala
class StashingPaymentActor extends Actor with Stash with ActorLogging {
```

Next lets go to the `offlineBehavior` and make it stash the messages instead of just sending an off-line response. Similarly, we need to unstash everything at the time we go back on-line.:

```scala
  def offlineBehavior: Receive = {
    case Online =>
      unstashAll()
      // ^^^ Here's your unstash change ^^^
      context.unbecome()
      sender() ! Online
    case _: PaymentRequest =>
      log.error("Received PaymentRequest but still Offline")
      stash()
      // ^^^ Here's your stash change ^^^
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
      sender() ! Status.Failure(new IllegalArgumentException("Message type not understood"))
  }
```

Then we want to handle all stashed messages when we get back online.

Now it should all work. Similarly, we want to make a copy of `PaymentActorSpec` into `StashingPaymentActorSpec` to make the test modifications.

We'll now modify the `testOffline()` method to test the stashing behavior. Here is is our new `testOffline()` method in `StashingPaymentActorSpec`:

```scala
  "The Payment Actor" should "respond only with `Offline` while offline" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF.futureValue.status shouldBe Approved

    // Now take the actor offline
    val responseF2 = paymentActor ? Offline
    responseF2.futureValue shouldBe Offline

    // Lets try to send in 5 payment requests. Each should return Offline
    (0 until 5).foreach { _ =>
      paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)
      id += 1
      expectNoMessage(1.second)
    }

    // Next we turn the PaymentActor back on-line
    val responseF4 = paymentActor ? Online
    responseF4.futureValue shouldBe Online

    // Receive the stashed messages
    receiveN(5, 5.seconds) foreach {
      case PaymentResponse(_, _, status) =>
        status shouldBe Approved
      case _ => fail("Message received is not a PaymentResponse")
    }

    // Now we should be back in business
    val responseF5 = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF5.futureValue.status shouldBe Approved
  }
```

### Actor Lifecycle

An actor keeps on living, until they are sent a `PoisonPill`, call `getContext().stop(self())` internally, or `getContext().stop(actorRef)` externally.

It is important to terminate actors when no longer in use. If we keep creating actors and not terminate them, we have a memory leak.

![Image showing actor lifecycle](https://doc.akka.io/docs/akka/current/images/actor_lifecycle.png)

### What we have not covered

* Piping results from async operations
* FSM
* Supervisor Policy

Please do read up on these in your own time.

Now we are done with Actors. Yeah!
