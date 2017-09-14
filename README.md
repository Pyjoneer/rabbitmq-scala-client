# RabbitMQ client

[![](https://teamcity.int.avast.com/app/rest/builds/buildType:(id:CloudSystems_RabbitMQClient_Build)/statusIcon)] (https://teamcity.int.avast.com/viewType.html?buildTypeId=CloudSystems_RabbitMQClient_Build)

![](http://badges.ff.int.avast.com/image/maven-local/com.avast.clients/rabbitmq-client_2.11?title=RabbitMQClient)

This client is lightweight wrapper over standard [RabbitMQ java client](https://www.rabbitmq.com/java-client.html).
It's API may be difficult to use for inexperienced RabbitMQ users. Goal of this library is to simplify basic use cases and shadow the programmer
from the underlying client.

It uses [Lyra library](https://github.com/jhalterman/lyra) for better recovery ability.

Author: [Jenda Kolena](mailto:kolena@avast.com)

## Dependency
`compile 'com.avast.clients:rabbitmq-client-core_?:x.x.x'`
For most current version see the [Teamcity](https://teamcity.int.avast.com/viewType.html?buildTypeId=CloudSystems_RabbitMQClient_ReleasePublish).

## Usage

### Configuration

#### Structured config
Since v 6.x, it's necessary to have the config structured as following:
```hocon
rabbitConfig {
  // connection config
  
  consumer1 {
    //consumer config
  }
  
  consumer2 {
    //consumer config
  }
  
  producer1 {
    //producer config
  }
  
  producer2 {
    //producer config
  }
}

```

#### Config example

```hocon
myConfig {
  hosts = ["localhost:5672"]
  virtualHost = "/"
  
  name="Cluster01Connection" // used for logging AND is also visible in client properties in RabbitMQ management console

  ssl {
    enabled = false // enabled by default
  }

  credentials {
    //enabled = true // enabled by default

    username = "guest"
    password = "guest"
  }

  connectionTimeout = 5s // default value

  networkRecovery {
    enabled = true // default value
    period = 5s // default value
  }


  // CONSUMERS AND PRODUCERS:

  // this is the name you use while creating; it's recommended to use something more expressive, like "licensesConsumer"
  consumer {
    name = "Testing" // this is used for metrics, logging etc.

    consumerTag = Default // string or "Default"; default is randomly generated string (like "amq.ctag-ov2Sp8MYKE6ysJ9SchKeqQ"); visible in RabbitMQ management console

    queueName = "test"

    prefetchCount = 100 // don't change unless you have a reason to do so ;-)

    // should the consumer declare queue he wants to read from?
    declare {
      enabled = true // disabled by default

      durable = true // default value
      autoDelete = false // default value
      exclusive = false // default value
    }

    // bindings from exchanges to the queue
    bindings = [
      {
        // all routing keys the queue should bind with
        // leave empty or use "" for binding to fanout exchange
        routingKeys = ["test"]

        // should the consumer declare exchange he wants to bind to?
        exchange {
          name = "myclient"

          declare {
            enabled = true // disabled by default

            type = "direct" // fanout, topic
          }
        }
      }
    ]
  }

  // this is the name you use while creating; it's recommended to use something more expressive, like "licensesProducer"
  producer {
    name = "Testing" // this is used for metrics, logging etc.

    exchange = "myclient"

    // should the consumer declare exchange he wants to send to?
    declare {
      enabled = true // disabled by default

      type = "direct" // fanout, topic
      durable = true // default value
      autoDelete = false // default value
    }
  }
}
```
For full list of options please see [reference.conf](src/main/resources/reference.conf).

### Scala usage

```scala
  val config = ConfigFactory.load().getConfig("myRabbitConfig")

  // you need both `ExecutorService` (optionally passed to `RabbitMQFactory`) and `ExecutionContext` (implicitly passed to consumer), both are
  // used for callbacks execution, so why not to use a `ExecutionContextExecutionService`?
  implicit val ex: ExecutionContextExecutorService = ???

  val monitor = new JmxMetricsMonitor("TestDomain")

  // here you create the factory; it's shared for all producers/consumers amongst one RabbitMQ server - they will share a single TCP connection
  // but have separated channels
  // if you expect very high load, you can use separate connections for each producer/consumer, but it's usually not needed
  val rabbitFactory = RabbitMQFactory.fromConfig(config, Some(ex))

  val receiver = rabbitFactory.newConsumer("consumer", monitor) { delivery =>
    println(delivery)
    Future.successful(true)
  }

  val sender = rabbitFactory.newProducer("producer", monitor)
```

### Java usage

The Java api is placed in subpackage `javaapi` (but not all classes have their Java counterparts, some have to be imported from Scala API,
depending on your usage).  
Don't get confused by the Java API partially implemented in Scala.

```java
final RabbitMQJavaFactory factory = RabbitMQFactory.newBuilder(config).withExecutor(executor).build();

final RabbitMQConsumer rabbitMQConsumer = factory.newConsumer(
    "consumer",
    NoOpMonitor.INSTANCE,
    executor,
    ExampleJava::handleDelivery
);

final RabbitMQProducer rabbitMQProducer = factory.newProducer("producer",
    NoOpMonitor.INSTANCE
);

```

See [full example](/src/test/java/ExampleJava.java)

##Notes

### DeliveryResult
The consumers `readAction` returns `Future` of [`DeliveryResult`](src/main/scala/com/avast/clients/rabbitmq/DeliveryResult.scala). The `DeliveryResult` has 4 possible values
(descriptions of usual use-cases):
1. Ack - the message was processed; it will be removed from the queue
1. Reject - the message is corrupted or for some other reason we don't want to see it again; it will be removed from the queue
1. Retry - the message couldn't be processed at this moment (unreachable 3rd party services?); it will be requeued (inserted on the top of
the queue)
1. Republish - the message may be corrupted but we're not sure; it will be re-published to the bottom of the queue (as a new message and the
original one will be removed). It's usually wise  to prevent an infinite republishing of the message - see [Poisoned message handler](#Poison_message_handler) below.

####Difference between _Retry_ and _Republish_
When using _Retry_ the message can effectively cause starvation of other messages in the queue
until the message itself can be processed; on the other hand _Republish_ inserts the message to the original queue as a new message and it
lets the consumer handle other messages (if they can be processed).

### Extras
There is an extra module available with some optional functionality.
`compile 'com.avast.clients:rabbitmq-client-core_?:x.x.x'`

#### HealthCheck
The library is not able to recover from all failures so it provides [HealthCheck class](/src/main/scala/com/avast/clients/rabbitmq/HealthCheck.scala)
 that indicates if the application is OK or not - then it should be restarted.
To use that class, simply pass the `rabbitExceptionHandler` field as listener when constructing the RabbitMQ classes. Then you can call `getStatus` method.

If you are using [Yap](https://git.int.avast.com/ff/yap) then you can use it in this way:
```scala
object YapHealthCheck extends HealthCheck with (HttpRequest[Bytes] => CompletableFuture[HttpResponse[Bytes]]) {
  override def apply(req: HttpRequest[Bytes]): CompletableFuture[HttpResponse[Bytes]] = {
    val resp = getStatus match {
      case Ok => HttpResponses.Ok()
      case Failure(msg, _) => HttpResponses.InternalServerError(Bytes.copyFromUtf8(msg))
    }
    CompletableFuture.completedFuture(resp)
  }
}
```

#### Poisoned message handler
It's quite often use-case we want to republish failed message but want to avoid the message to be republishing forever. Wrap your handler (readAction)
[PoisonedMessageHandler] with to solve this issue. It will count no. of attempts and won't let the message to be republished again and again
(above the limit you set).  
```scala
val newReadAction = new PoisonedMessageHandler(3)(myReadAction)
```
Java:
```java
newReadAction = PoisonedMessageHandler.forJava(3, myReadAction, executor);
```
You can even pretend lower number of attempts when you want to rise the republishing count (for some special message):
```scala
Republish(Map(PoisonedMessageHandler.RepublishCountHeaderName -> 1.asInstanceOf[AnyRef]))
```
