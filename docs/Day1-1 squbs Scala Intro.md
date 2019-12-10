# Akka Scala/squbs Training

## Day1: Concepts & Overview

### Akka Overview

#### Why Akka?
   * Actors
      * Simple abstraction
      * High performance
      * Asynchronous
      * No contention
   * Fault tolerance with supervisor hierarchies
   * Streams: resilience & back-pressure
   * Message-oriented architecture

#### The Reactive Manifesto
   * Responsive in all cases
   * Resilient to internal and external failures
   * Elastic to load spikes
   * Message driven, asynchronous architecture enables it all

#### The ActorSystem
   * The base infrastructure to run actors, streams, etc.
   * Has dispatchers to schedule actors
   * Schedules actors to run on dispatchers (thread pool)

#### The Actor Model of Computation
   * Defined by Hewitt et al
   * Most scalable architecture we have at this point
   * **Rules**:
      * Actor, when receive messages can:
         * Send messages
         * Create other actors
         * Change state/behavior to be used by next message

#### Akka Actor instances
   * Have a mailbox
   * Do work on messages in the mailbox
   * Can process multiple messages in one scheduling cycle

#### Mainstream Akka Actors are untyped
   * They can receive/send any message of any type
   * Akka Typed tries to add strong typing to actors

#### Futures
   * Akka API uses Scala `Future` for handle of something completing in the future
   * `Future` is transformable/composable

#### Clustering/Remoting
   * Actors are location transparent
   * Send/receive messages over network

#### Streams
   * Higher-level abstraction than Actors
   * Type safe
   * Provides back-pressure and therefore resilience (system is never overloaded)
   * Provides a programming model of composing stream components like a circuit board
   * Core piece of squbs since squbs 0.9.0 (we're releasing 0.11.0 very soon)
   * **Homework**: Please watch QCon video on squbs and stream processing. This video is available from [http://squbs](http://squbs)

### Some programming rules
* Immutable first
   * I.e. Spring beans are inherently mutable. Java beans are normally mutable
   * Scala collections and case classes are immutable by default, but mutable versions are available
   * If mutability is needed, following is the order of preference
      a. Immutable
      b. Mutable reference to immutable object
      c. Immutable reference to mutable object
   * **Unacceptable**: Mutable reference to mutable object
   * Limit scope of mutability to a minimum
   * DO use builder pattern to build more complicated immutable objects
* Never block
   * I.e. never wait for any kind of futures - such waiting blocks. For Scala `Future`, this would be the `Await` call. Don't use it except for tests.
   * Use functional composition to avoid blocking
   * If you cannot avoid blocking, such as needing to make blocking I/O calls, designate a blocking dispatcher for blocking code. Tune this dispatcher.
* No `null`s. Use `Option`... This is standard convention for Scala but not for Java.
* Don't use concurrency constructs and libraries. Pretty much every feature is off-limits in this architecture except for atomics (members of `java.util.concurrent.atomic` package). Concurrency is already managed for you.
   * DO NOT create threads or thread pools.
* Don't rely on `ThreadLocal`. They don't work and do give intermittent failures.
* Don't create other `ActorSystem`s. Let squbs manage `ActorSystem`s.
* Create stream `Materializer`s sparingly and maximize reuse - they are expensive. squbs provides a facility to access default materializers without creating a new one.
* For HttpClient, **always** consume or discard the `Entity` even in case you receive an error response. Failure to do so causes back-pressure and can get the stream stuck.

### What is squbs?
* Open source project by PayPal
   * https://github.com/paypal/squbs
* Covers
   * Bootstrapping & lifecycle management
   * Module management
   * Service management
   * Extension management
   * HttpClient
   * Pipeline (client/server)
   * Testing tools
   * Orchestrator
   * Stream components
   * Complex marshallers/unmarshallers
   * Validation
   * Admin console
   * Monitoring
   * More goodies

### How to get help?
* Slack #help-squbs
* https://gitter.im/paypal/squbs - for external questions/community
* Documents are at https://squbs.readthedocs.io/en/latest/ (external). There is a link from https://github.com/paypal/squbs.
* Akka documentation: http://akka.io/docs/ with links to JavaDoc

### Lets create your first app
* Seed demo allowing everyone to create app
* Import application into IDE
* Review the anatomy of the default application
