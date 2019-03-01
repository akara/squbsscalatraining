## Day 4: Serving Http Requests

### Creating a Service with the High Level API

#### Simple Hello Service

Create a new Java class in your `*svc` project named `MockService`. We'll re-use it later. And lets start with something really simple. Just a `hello` service that says `hi` to you:

```scala
import akka.http.scaladsl.server.Route
import org.squbs.unicomplex.RouteDefinition

class MockService extends RouteDefinition {
  override def route: Route =
    path("hello") {
      complete("hi")
    }
}
```

#### Service Registration

Next, register the service to `squbs`. Edit `*svc/src/main/resource/META-INF/squbs-meta.conf`.

```
cube-name = {my-package.cubename}

cube-version = "0.0.1-SNAPSHOT"
squbs-services = [
  {
    class-name = {my-package}.MockService
    listeners = [default-listener, paypal-listener]
    web-context = mock
  }
]
```

Make sure to replace all `{...}` with real values. The `cube-name` is likely generated for you already. No need to change. However, you need to ensure the `class-name` is accurate. To check in IntelliJ press the `Command` button on your Mac or the `Ctrl` button on your Windows system and mouse-over the class name. That name should become a link and IntelliJ should describe that class. If no popup occurs, the class name is wrong.

#### Run the service

1. From `sbt` window,
   * Type `reload` to ensure sbt is up-to-date with the dependencies
   * Run `extractConfig dev` to populate config
2. Configure `App` in IntelliJ - add AspectJ config
   * Click down button and choose “Edit Configurations"
   * Click the `+` sign
   * Choose `application`
   * Name: `App`
   * Main class: `org.squbs.unicomplex.Bootstrap`
   * VM options: `-javaagent:/Users/{my_login}/.ivy2/cache/org.aspectj/aspectjweaver/jars/aspectjweaver-1.8.13.jar`. Don't forget to replace `{my_login}` with your real user name. An equivalent path is needed for Windows.
   * Working directory: `…/{project}servsvc`
   * Use classpath of module: `{project}servsvc`
   * Press `OK`
3. Run the app by pressing the start button with the right arrow
4. Check the app and registered context
   * Point your browser to `http://localhost:8080/admin/v3console/ValidateInternals`
   * Choose the `Component Status` tab
   * Select the link `org.squbs.unicomplex…::Listeners`
   * See the app registered to Default Listener
5. Point your browser to `http://localhost:8080/mock/hello`

This way of setting up and running the service lends itself very well to debugging. There are other ways to run the service directly fromt he `sbt shell` as follows:

1. Assume the application is running from IntelliJ, stop it.
2. Open the `sbt shell` window in your IDE.
3. Type:
   
   `project {project}svc`
   `~re-start`
   
4. Wait until project is up, only a few seconds, then test it out the same way.  The `~` in front of `re-start` causes sbt to watch the project directory and re-start the server anytime a file changes.

5. After done testing, enter `re-stop` to stop the server.

#### Adding Meaning to MockService

So far we have been just serving a `hello` request. Now lets make the service serve some useful data. First, lets create an account class as follows, just in the `MockService.scala` file, above the `MockService` class:

```scala
case class Account(id: Long, name: String, email: String, lastBalance: Long)
```

Next lets make the `MockService` return an account. Add this following `path` directive below the `path(hello) { ...`

```scala
    path("account" / LongNumber) { id =>
      import de.heikoseeberger.akkahttpjson4s.Json4sSupport._
      complete(StatusCodes.OK, Account(id, "Foo Bar", "foobar@foobar.com", 100))
    }
```

We import the `Json4sSupport` in the scope of just this block to ensure it does not interfere with other non-json paths. To ensure we do not have any missing symbols, also add the following imports:

```scala
import akka.http.scaladsl.model.StatusCodes
import org.json4s.{DefaultFormats, native}
```

Also, `Json4sSupport` requires two implicit arguments. We can declare these two implicit fields in the `MockService` class. These allow you to control the serialization. We'll stay with defaults for now:

```scala
implicit val formats = DefaultFormats
implicit val serialization = native.Serialization
```

Putting it together, your `MockService.scala` will looks like this:

```scala
import akka.http.scaladsl.model.StatusCodes
import akka.http.scaladsl.server.Route
import org.json4s.{DefaultFormats, native}
import org.squbs.unicomplex.RouteDefinition

case class Account(id: Long, name: String, email: String, lastBalance: Long)

class MockService extends RouteDefinition {

  implicit val formats = DefaultFormats
  implicit val serialization = native.Serialization

  override def route: Route =
    path("hello") {
      complete("hi")
    } ~
    path("account" / LongNumber) { id =>
      import de.heikoseeberger.akkahttpjson4s.Json4sSupport._
      complete(StatusCodes.OK, Account(id, "Foo Bar", "foobar@foobar.com", 100))
    }
}
```

Restart your service and try this URL: `http://localhost:8080/mock/account/100`

So far we have learned how to do path matching and use an implicit Jackson  marshaller to send a marshalled object back.

The route DSL is very versatile and allows all kinds of matching and extractions. We just touched the surface of it using `path` and `complete`. For a full list of directives, please visit [Directives](https://doc.akka.io/docs/akka-http/current/routing-dsl/directives/index.html?language=scala) in the Akka-HTTP documentation.

### Creating a Service with the Low Level API

So far we have used the high-level route API to serve requests through `MockService`. This high-level route API is very versatile and useful for complicated REST API matchings.

There is another low-level API that is based on Akka Streams `Flow` concept and component. It may be a little more cumbersome to use for complex REST requests. But in exchange, it delivers higher performance, lower latency, consumes less memory, and allows us to provide end-to-end back-pressure. It is extremely suitable for high-volume, highly resilient services.

The `Flow` API is defined as `Flow<HttpRequest, HttpResponse, NotUsed>` telling us to build a flow component that processes an `HttpRequest` into an `HttpResponse`. Lets try create a `ValidationService` with the low-level API. Again, we start simple. First create a Scala class `ValidationService` that just provides a simple `"Hello"` flow:

```java
import akka.NotUsed
import akka.http.scaladsl.model.{HttpRequest, HttpResponse, StatusCodes}
import akka.stream.scaladsl.Flow
import org.squbs.unicomplex.FlowDefinition

class ValidationService extends FlowDefinition {
  override def flow: Flow[HttpRequest, HttpResponse, NotUsed] =
    Flow[HttpRequest].map { req =>
      HttpResponse(status = StatusCodes.OK, entity = "Hello")
    }
}
```

#### Register the Service

Similar to registering `MockService`, we need to register `ValidationService` in `*svc/src/main/resources/META-INF/` in order to have it being recognized and installed 

```yaml
cube-name = {my-package.cubename}

cube-version = "0.0.1-SNAPSHOT"
squbs-services = [
  {
    class-name = {my-package}.MockService
    listeners = [default-listener, paypal-listener]
    web-context = mock
  }
  {
    class-name = {my-package}.ValidationService
    listeners = [default-listener, paypal-listener]
    web-context = validate
  }
  # ^^^^^^^^^^^^^^^^^^^^^^^^^
  # Add this block as another service with another context, "validate"
]
```

Remember to check the classes referenced here are actually valid.

Restart the server and try the URL: `http://localhost:8080/validate`

### Calling Another Service

Similar to the `Flow`-based low-level service, we use streams to make HttpClient calls. `squbs` provides a beefed up HttpClient with all required enterprise functionality through the `ClientFlow` API. An example of the `ClientFlow` API can be seen in the followings:

```java
import akka.actor.ActorSystem
import akka.http.scaladsl.model.{HttpRequest, HttpResponse}
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.{Sink, Source}
import org.squbs.httpclient.ClientFlow

import scala.concurrent.Future
import scala.util.Try

object ClientSample {
  
  implicit val system = ActorSystem("ClientSample")
  implicit val mat = ActorMaterializer()
  
  val clientFlow = ClientFlow[Int]("sample")
  
  def clientCall(request: HttpRequest): Future[(Try[HttpResponse], Int)] =
    Source.single(request -> 42).via(clientFlow).runWith(Sink.head)
}
```

Lets take a closer look at each part of the API. First we need an `ActorSystem` to access the `ClientFlow`. Like all `Akka Streams`-based facilities, we also need a materializer, both get passed implicitly and need to be declared with `implicit`. We can create new ones as seen in this example, or we can use an existing one. The actual `ClientFlow` instance is created by calling `ClientFlow(String name)`. The name refers to a service name that gets resolved to a real service endpoint behind the scenes.

As we now have the `ClientFlow` we can go into its usage. Lets start with the HttpClient architecture. The HttpClient holds a pool of persistent connnections, for which the size can be specified in the configuration. Some http calls may be slower than the other. In order to maximize the stream throughput and response time of all `ClientFlow` calls, the `ClientFlow` does not guarantee the order of  messages. Some http calls may be faster and respond before preceding calls.

For this reason, the `ClientFlow` takes a context of arbitrary type (`Int` 42 in this case) in addition to the `HttpRequest` itself. It allows users to identify which request a response corresponds to. While `ClientFlow` does not require this context to be unique, it is in the best interest of the developer to keep it unique.

In the example above, the context is just an `Int` of value `42`. In the simplest case, we create a source of a tuple of `HttpRequest` and `Int` and pass it to the `ClientFlow`. Then we obtain the `Sink.head()` which is the first and only value coming out. That would be the materialized value for this stream, which gives us a `Future[](Try[HttpResponse], Int)]` handing us the response asynchronously.

Well, this is just the simplest use case. When we build an end-to-end application, we may use the stream in a very different manner. Lets try it out.

## E2E flow + Testing

### Building an End-To-End Flow

In this section, we will modify our `ValidationService` to do something more useful and make a client REST call to `MockService`, asking for accounts given an account number range. Lets get started:

Design:

* We send an HttpRequest to do mass account fetches based on a begin id and end id as follows: `http://localhost:8080/validate/{startAcct}/{endAcct}`
* In the server, we extract the `startAcct` and `endAcct` and expand it to each account id in the stream.
* The stream then creates an `HttpRequest` from each account id and sends it along with the account id as a context to the `ClientFlow` which calls the `MockService`. The response is then aggregated into a JSON and sent back to the sender.

Let's get our hands dirty and build this service, in full. We start with defining some `case class`es inside of `ValidationService` as follows:

```scala
  case class MessageContext(origRequest: HttpRequest, beginAcct: Long, rangeSize: Int, accountId: Long)
  case class AccountStatus(acctId: Long, status: String)
```

Now we define each of our flow components that we'll compose at the end. First the `pathMatchFlow`:

```scala
  val pathMatchFlow = Flow[HttpRequest].map { req =>
    import PathMatcher._
    val matcher: PathMatcher[(Long, Long)] = "validate" / LongNumber / LongNumber
    matcher(req.uri.path.tail) match {
      case Matched(_, (beginAcct, endAcct)) => MessageContext(req, beginAcct, (endAcct - beginAcct + 1).toInt, -1)
      case _ => MessageContext(req, 0, 0, -1)
    }
  }
```

This flow uses the `PathMatcher` utility to extract the begin and end account ids from the URL path. We declare the `PathMatcher` using a DSL `"validate" / LongNumber / LongNumber`. Since it returns 2 `Longs` the type parameter must be `PathMatcher[(Long, Long)]`.

Next, since `PathMatcher` does not match the leading slash, we need to strip that from the `Path` by calling `.tail`, the `.head` being the `/` itself.

When it does not match, we create an empty `MessageContext` with `rangeSize` of 0 that we'll deal with later.

Next we need to have a routing that routes valid vs empty `MessageContext`s. The `Partition` stage is a great router that takes a function to help decide which route the message should go.

```scala
  val validateInput = Partition[MessageContext](2, {
    case messageContext if messageContext.rangeSize == 0 => 1
    case _ => 0
  })
```

Then we deal with the `invalidInputFlow` which responds the invalid input.

```scala
  val invalidInputFlow = Flow[MessageContext].map { ctx =>
    HttpResponse(StatusCodes.BadRequest, entity = s"Request ${ctx.origRequest.uri.path} invalid")
  }
```

On the happy path side, we now need to expand the range into individual elements. We can do so using `mapConcat` or `flatMapConcat`. We choose the latter for this exercise, for which we have to return a `Source`.

```scala
  val expandIdsFlow = Flow[MessageContext].flatMapConcat { messageContext =>
    val begin = messageContext.beginAcct
    val end = begin + messageContext.rangeSize
    Source(begin until end).map(id => messageContext.copy(accountId = id))
  }
```

Remember, to make an HTTP request, we need to build a tuple with the `HttpRequest` and a context.

```scala
  val buildRequestFlow = Flow[MessageContext].map(ctx => (HttpRequest(uri = s"/mock/account/${ctx.accountId}"), ctx))
```

Then that tuple has to be passed through the `ClientFlow`. We next define that `ClientFlow`.

```scala
  val clientFlow = {
    import context.system
    ClientFlow[MessageContext]("mock")
  }
```

The client flow needs the `ActorSystem`, so we just have to import it in this context.

Next we need to handle the individual responses:

```scala
  val handleResponseFlow = Flow[(Try[HttpResponse], MessageContext)].mapAsync(10) {
    case (Success(response), ctx) if response.status == StatusCodes.OK =>
      response.entity.toStrict(1.second).map(entity => (AccountStatus(ctx.accountId, entity.data.utf8String), ctx))
    case (Success(response), ctx) =>
      response.entity.discardBytes()
      Future.successful((AccountStatus(ctx.accountId, response.status.toString), ctx))
    case (Failure(e), ctx) =>
      Future.successful((AccountStatus(ctx.accountId, e.toString), ctx))
  }
```

The `handleResponseFlow` is a little more involved. It needs to deal with 3 cases. One is the happy path, one for a error response from the server, and one where the server cannot even be reached. In all cases, we come up with the `AccountStatus` and pass the context along as a tuple. Also note that accessing the entity incurs a `Future` call. That's why we need to use `mapAsync` for this stage. Cases that don't need to respond with a `Future` can just wrap their response with `Future.successful` to fit into `mapAsync`'s requirements. We also need to pay special attention to discard the entity if we do not intend to use it. Failure to do so will create back-pressure on the stream.

The last step we need to build the final response.

```scala
  val buildResponseFlow = Flow[(AccountStatus, MessageContext)]
    .scan(Queue.empty[(AccountStatus, MessageContext)]) { (q, elem) =>
      if (q.nonEmpty && q.head._2.reqId != elem._2.reqId) Queue(elem)
      else q :+ elem
    }
    .collect {
      case q if q.nonEmpty && q.size >= q.head._2.rangeSize =>
        q.map { case (acctStatus, _) => acctStatus.status }
    }
    .map(q => HttpResponse(entity = HttpEntity(ContentTypes.`application/json`, q.mkString("[", ",", "]"))))
```

This one is a little tricky. We need to determine whether we have all the input for the response. For that we use the `scan` stage to create a potential response as a `Queue`. We also need to check whether the `Queue` we have is from a previous request and discard accordingly. Every `Queue` then gets checked whether it has all the elements. We only keep the complete queue and filter everyone else out using the `collect` stage which can pattern match and transform the stream elements in one shot. Lastly, we turn the accepted queue into an `HttpResponse`

Then we compose all components together into a single `Flow` using the `GraphDSL`.

```scala
  override def flow: Flow[HttpRequest, HttpResponse, NotUsed] =
    Flow.fromGraph(GraphDSL.create() { implicit b =>
      import GraphDSL.Implicits._
      val pathMatch = b.add(pathMatchFlow)
      val validate = b.add(validateInput)
      val merge = b.add(Merge[HttpResponse](2))

      pathMatch ~> validate
      validate ~> expandIdsFlow ~> buildRequestFlow ~> clientFlow ~> handleResponseFlow ~> buildResponseFlow ~> merge
      validate ~> invalidInputFlow ~> merge

      FlowShape(pathMatch.in, merge.out)
    })

```

Note that we need to add the `pathMatchFlow` the the builder `b` for it to become a `FlowShpe` so we can access its input port `.in`.

Putting the whole picture together now looks like the following: 

```scala
package com.paypal.squbs.straining.svc


import java.util.concurrent.atomic.AtomicLong

import akka.NotUsed
import akka.http.scaladsl.model._
import akka.http.scaladsl.server.PathMatcher
import akka.http.scaladsl.server.PathMatchers.LongNumber
import akka.stream.scaladsl.{Flow, GraphDSL, Merge, Partition, Source}
import akka.stream.{ActorMaterializer, FlowShape}
import org.squbs.httpclient.ClientFlow
import org.squbs.unicomplex.FlowDefinition

import scala.collection.immutable.Queue
import scala.concurrent.Future
import scala.concurrent.duration._
import scala.util.{Failure, Success, Try}

object ValidationService {
  private[svc] val currentId = new AtomicLong(0L)
}

class ValidationService extends FlowDefinition {

  import ValidationService._

  case class MessageContext(origRequest: HttpRequest, beginAcct: Long, rangeSize: Int, accountId: Long,
                            reqId: Long = currentId.getAndIncrement())
  case class AccountStatus(acctId: Long, status: String)

  import context.dispatcher

  implicit val mat = ActorMaterializer()

  val pathMatchFlow = Flow[HttpRequest].map { req =>
    import PathMatcher._
    val matcher: PathMatcher[(Long, Long)] = "validate" / LongNumber / LongNumber
    matcher(req.uri.path.tail) match {
      case Matched(_, (beginAcct, endAcct)) => MessageContext(req, beginAcct, (endAcct - beginAcct + 1).toInt, -1)
      case _ => MessageContext(req, 0, 0, -1)
    }
  }

  val validateInput = Partition[MessageContext](2, {
    case messageContext if messageContext.rangeSize == 0 => 1
    case _ => 0
  })

  val invalidInputFlow = Flow[MessageContext].map { ctx =>
    HttpResponse(StatusCodes.BadRequest, entity = s"Request ${ctx.origRequest.uri.path} invalid")
  }


  val expandIdsFlow = Flow[MessageContext].flatMapConcat { messageContext =>
    val begin = messageContext.beginAcct
    val end = begin + messageContext.rangeSize
    Source(begin until end).map(id => messageContext.copy(accountId = id))
  }

  val buildRequestFlow = Flow[MessageContext].map(ctx => (HttpRequest(uri = s"/mock/account/${ctx.accountId}"), ctx))

  val clientFlow = {
    import context.system
    ClientFlow[MessageContext]("mock")
  }

  val handleResponseFlow = Flow[(Try[HttpResponse], MessageContext)].mapAsync(10) {
    case (Success(response), ctx) if response.status == StatusCodes.OK =>
      response.entity.toStrict(1.second).map(entity => (AccountStatus(ctx.accountId, entity.data.utf8String), ctx))
    case (Success(response), ctx) =>
      response.entity.discardBytes()
      Future.successful((AccountStatus(ctx.accountId, response.status.toString), ctx))
    case (Failure(e), ctx) =>
      Future.successful((AccountStatus(ctx.accountId, e.toString), ctx))
  }

  val buildResponseFlow = Flow[(AccountStatus, MessageContext)]
    .scan(Queue.empty[(AccountStatus, MessageContext)]) { (q, elem) =>
      if (q.nonEmpty && q.head._2.reqId != elem._2.reqId) Queue(elem)
      else q :+ elem
    }
    .collect {
      case q if q.nonEmpty && q.size >= q.head._2.rangeSize =>
        q.map { case (acctStatus, _) => acctStatus.status }
    }
    .map(q => HttpResponse(entity = HttpEntity(ContentTypes.`application/json`, q.mkString("[", ",", "]"))))

  override def flow: Flow[HttpRequest, HttpResponse, NotUsed] =
    Flow.fromGraph(GraphDSL.create() { implicit b =>
      import GraphDSL.Implicits._
      val pathMatch = b.add(pathMatchFlow)
      val validate = b.add(validateInput)
      val merge = b.add(Merge[HttpResponse](2))

      pathMatch ~> validate
      validate ~> expandIdsFlow ~> buildRequestFlow ~> clientFlow ~> handleResponseFlow ~> buildResponseFlow ~> merge
      validate ~> invalidInputFlow ~> merge

      FlowShape(pathMatch.in, merge.out)
    })
}
```

Remember, we referred to a service called `"mock"`. Before we can get this running, we also have to edit our `application.properties` file. This is in `*svc/src/main/resources/META-INF/configuration/Dev/env.conf`. Add the resource as follows:

```
topo-connections {
  mock_host = localhost
  mock_port = 8080
  mock_protocol = http
}
```

After changing the configurations, make sure to re-run `extractConfig Dev` from your `sbt shell` window.

### Testing Http(s) Services

Since we're using JUnit in this our session, we also need to make sure `JUnit` is in your dependencies. Edit the service project's `build.sbt` and add the necessary `JUnit` dependencies as follows:

squbs provides a powerful `CustomTestKit` API used for creating tests. Just extend the `CustomTestKit` API. You can pass configuration to this API through calling the superclass constructor. A picture is worth a thousand words, so lets get straight down to code.

* In your service project under `src/test/scala` create the same package as your mock service. If the `scala` directory does not exist, you may want to create that, too.
* Under this package, create a class called `MockServiceSpec`.

The code for `MockServiceSpec	` is as follows:

```java
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.HttpRequest
import akka.stream.ActorMaterializer
import org.scalatest.concurrent.ScalaFutures
import org.scalatest.{FlatSpecLike, Matchers}
import org.squbs.testkit.CustomTestKit

import scala.concurrent.duration._

class MockServiceSpec extends CustomTestKit(withClassPath = true) with FlatSpecLike with Matchers with ScalaFutures {

  implicit val patience = PatienceConfig(5.seconds, 50.millis)
  implicit val mat = ActorMaterializer()

  "The mock service" should "respond to simple request" in {
    val responseF = Http(system).singleRequest(HttpRequest(uri = s"http://localhost:$port/mock/hello"))
    val content = responseF.futureValue.entity.toStrict(1.second).futureValue.data.utf8String
    content shouldBe "hi"
  }
}
```

The test is straightforward. We mix in ScalaTest's trait `ScalaFutures` to allow us to wait for the `Future`s. Since we're doing an actual service call in this test, we need to allow for a larger timeout. This is set through the `PatienceConfig`. The settings above polls every 50 milliseconds and times out after 5 seconds.

The test configuration is not the same as the Dev configuration. You can pass the configuration in through the CustomTestKit constructor. Or in this case, we'll add the test configuration to `/src/test/resources`. Lets create `/src/test/resources/application.conf` with the following content:

```
default-listener  {
  aliases = [dev-listener]
}

topo-connections {
  mock_host = localhost
  mock_port = 8080
  mock_protocol = http
}
```

Now the test should run without an issue. Just run the JUnit test as usual.
