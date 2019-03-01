# Akka Scala/squbs Training

## Day1: Your first Actors

### Create Status Enum and Message Classes

1. Create new Scala class `PaymentActor`

2. In the same file as `PaymentActor`, create case objects for `PaymentStatus`

   ```scala
   sealed trait PaymentStatus
   case object Accepted extends PaymentStatus
   case object Approved extends PaymentStatus
   case object Rejected extends PaymentStatus
   ```

3. Create case classes `RecordId`, `PaymentRequest`, and `PaymentResponse`. Note that amounts are of type `Long` and includes the two decimal points. Divide by 100D to get the actual dollar amount.
   
   ```scala
   case class RecordId(id: Long, creator: Long, creationTime: Long = System.currentTimeMillis)   
   case class PaymentRequest(id: RecordId, payerAcct: Long, payeeAcct: Long, amount: Long)
   case class PaymentResponse(id: RecordId, requestId: RecordId, status: PaymentStatus)
   ```

### Create our Fundamental Actor

We'll expand on this actor a bit later. So lets start with something very basic:

```scala
class PaymentActor extends Actor with ActorLogging {

  val creatorId = 100L
  var id = 1L

  override def receive: Receive = {
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      log.info("Received payment request of ${} accounts {} => {}",
        amount / 100D, payerAcct, payeeAcct)
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
  }
}
```

So far our actor is not doing a lot. It receives a message for a payment and just logs. Nothing very useful. But at least we have a running actor.

### Register Your Actor

So that squbs automatically starts the actor. Add your actor to your cube project's `src/main/resources/META-INF/squbs-meta.conf`.

```
cube-name = com.paypal.myorg.myservcube
cube-version = "0.0.1-SNAPSHOT"
squbs-actors = [
  {
    class-name = com.paypal.myorg.myserv.cube.PaymentActor
    name = paymentActor
  }
]
```

### Testing Your Actor

1. Create folder `scala` under src/test.
2. Under `src/test/scala`, create package with the same name as the package containing your actor.
3. Create the test file `PaymentActorSpec` with a simple call to the actor as follows:

   ```scala
   import akka.actor.{ActorSystem, Props}
   import akka.testkit.{ImplicitSender, TestKit}
   import org.scalatest.{FlatSpecLike, Matchers}

   class PaymentActorSpec extends TestKit(ActorSystem("PaymentActorSpec"))
     with FlatSpecLike with Matchers with ImplicitSender {

     val creator = 200L
     var id = 1L

     private val paymentActor = system.actorOf(Props[PaymentActor])

     "The Payment Actor" should "react to payment requests" in {
       paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 30000)
       id += 1

       // Temporarily have the sleep here so the test does not terminate prematurely.
       Thread.sleep(1000)
     }
   }
   ```

   The `PaymentActorSpec` mixes extends the Akka TestKit and adds in a few traits as follows:
   * `FlatSpecLike` defines the style of the test to use a `FlatSpec` style
   * `Matchers` enables ScalaTest's matcher DSL we'll use later
   * `ImplicitSender` provides a sender within the test for test actors to respond to
   
8. Right-click on your test class name `PaymentActorSpec` in the editor and select option `Run 'PaymentActorSpec'` from the pop-up menu. See your test results. Since our actor does no validations but only logs, we can observe it running by looking at the logs. We'll add assertions in the next step.
9. If you enabled the toolbar, you'll see `PaymentActorSpec` in list of items you can run. Select the test and press the `>` icon to start the test at any future time. Since our actor does nothing but log, we can observe it running by looking at the logs.

### Make the Actor send PaymentResponse back

1. Edit `PaymentActor` and add the following lines:

   ```scala
       case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
         log.info("Received payment request of ${} accounts {} => {}",
           amount / 100D, payerAcct, payeeAcct)
         sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Accepted)
         id += 1
         // ^^^ Add these two line here ^^^
   ```

2. Since `PaymentActor` now responds with a message, lets update our test to obtain response messages and validate its status.
   
   ```scala
     "The Payment Actor" should "react to payment requests" in {
       paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 30000)
       id += 1
       val response = expectMsgType[PaymentResponse]
       response.status shouldBe Accepted
       // ^^^ Add the 2 lines above. ^^^

       // Temporarily have the sleep here so the test does not terminate prematurely.
       // Thread.sleep(1000)
       // ^^^ Remove the sleep, commented lines here ^^^
     }
   ```

### Qualifying Messages

Sometimes we want to treat different messages arriving a bit different. For instance, a cafeteria payment of $12 or less should just be auto-approved. So lets add another case pattern matching.

**Note:** This case needs to be added above the current case as the pattern matching is done in sequence. This case is more specific than the existing one.

```scala
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) if amount < 1200 =>
      log.info("Received small payment request of ${} accounts {} => {}", amount / 100D, payerAcct, payeeAcct)
      sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Approved)
      id += 1
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      ...
```

This matcher has an added qualifier testing that the amount is less than 1200, or $12.00 and it will immediately respond with an `Approved` status.

Lets also add a test to see this in action. In the test, add another test.

```scala
  "The Payment Actor" should "approve small payment requests automatically" in {
    paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)
    id += 1
    val response = expectMsgType[PaymentResponse]
    response.status shouldBe Approved
  }
```

Now run the test again. It should just pass. Also note the log messages.

### Creating a Child Actor

Remember, based on the Actor Model of Computation, an actor, upon receiving a message can do one or more of these three:

* Send messages
* Create other actors
* Change state/behavior to be used by next message

Now, lets explore the second property: Creating another child actor. Note that this actor becomes the parent of this child actor.

Now we create another actor, the `RiskAssessmentActor`. In this case we'll let it approve every request, just for simplicity. Here is the code:

```java
class RiskAssessmentActor extends Actor with ActorLogging {

  val creatorId = 300L
  var id = 1L

  override def receive: Receive = {
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      log.info("Received payment assessment request of ${} accounts {} => {}",
        amount / 100D, payerAcct, payeeAcct);
      sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Approved)
      id += 1
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
  }
}
```

By itself, there is nothing interesting about `RiskAssessmentActor`. It is similar to `PaymentActor` in many ways. But now we'll let `PaymentActor` forward the approval request to `RiskAssessmentActor` and then return it to the client, which is the test code.

Lets enhance the `PaymentActor` to do so. We'll modify the current high risk payment code to do the forward as follows:

```java
    case request @ PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      // Change pattern matching to also provide the request itself by prepending request @
      log.info("Received payment request of ${} accounts {} => {}",
        amount / 100D, payerAcct, payeeAcct)
      sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Accepted)
      id += 1
      val riskActor = context.actorOf(Props[RiskAssessmentActor])
      riskActor forward request
      // ^^^ Add these two lines here ^^^
```

Notice now the client should receive two messages. One with `Accepted` status and one with `Approved` status. So lets modify our first test.

```scala
  "The Payment Actor" should "react to payment requests" in {
    paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 30000)
    id += 1
    val response = expectMsgType[PaymentResponse]
    response.status shouldBe Accepted
    val response2 = expectMsgType[PaymentResponse]
    response2.status shouldBe Approved
    // ^^^ Add the 2 lines above. ^^^
  }
```