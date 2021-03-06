---
id: howto_use_layers
title:  "Use modules and layers"
---

# Unleash ZIO environment with `ZLayer`

`ZIO` is designed around 3 parameters, `R, E, A`. `R` represents the _requirement_ for the effect to run, meaning we need to fulfill
the requirement in order to make the effect _runnable_. We will explore what we can do with `R`, as it plays a crucial role in `ZIO`.

## A simple case for `ZIO` environment
Let's build a simple program for user management, that can retrieve a user (if they exist), and create a user. 
To access the DB we need a `DBConnection`, and each step in our program represents this through the environment type. We can then combine the two (small) steps through `flatMap` or more conveniently through a `for` comprehension.

The result is a program that, in turn, depends on the `DBConnection`.

```scala mdoc:invisible
import zio.{ Has, IO, Layer, UIO, URIO, ZEnv, ZIO, ZLayer }
import zio.clock.Clock
import zio.console.Console
import zio.random.Random

trait DBError
trait Product
trait ProductId
trait DBConnection
case class UserId(value: Long)
```

```scala mdoc:silent
case class User(id: UserId, name: String)
```

```scala mdoc:silent
def getUser(userId: UserId): ZIO[DBConnection, Nothing, Option[User]] = UIO(???)
def createUser(user: User): URIO[DBConnection, Unit] = UIO(???)

val user: User = User(UserId(1234), "Chet")
val created: URIO[DBConnection, Boolean] = for {
  maybeUser <- getUser(user.id)
  res       <- maybeUser.fold(createUser(user).as(true))(_ => ZIO.succeed(false))
} yield res
```

To run the program we must supply a `DBConnection` through `provide`, before feeding it to ZIO runtime.

```scala
val dbConnection: DBConnection = ???
val runnable: UIO[Boolean] = created.provide(dbConnection)

val finallyCreated  = runtime.unsafeRun(runnable)
```

Notice that the act of `provide`ing an effect with its environment eliminates the environment dependency in the resulting effect type, represented by type `Any` of the resulting environment.

In general we need more than just a DB connection though. We need components that enable us to perform different operations, and we need to be able to wire them together. This is what _modules_ are for.

## Our first ZIO module
We will see now how to define modules and use them to create different application layers relying on each other. The core idea is that a layer depends on the layers imediately below but it is completely agnostic about their internal implementation.

This formulation of module pattern is _the way_ ZIO manages dependencies between application components, giving extreme power in terms of compositionality and offering the capability to easily change different implementations. This is particularly useful during the testing/mocking phase.

### What is a module?
A module is a group of functions that deals with only one concern. Keeping the scope of a module limited improves our ability to understand code, in that we need to focus
 only on one topic at a time without juggling with too many concepts together in our head.

`ZIO` iself provides the basic capabilities through modules, e.g. see how `ZEnv` is defined.

### The module recipe
Let's build a module for user data access, following these simple steps:

1. Define an object that gives the name to the module, this can be (not necessarily) a package object
1. Within the module object define a `trait Service` that defines the interface our module is exposing, in our case 2 methods to retrieve and create a user
1. Within the module object define the different implementations of `ModuleName` through `ZLayer` (see below for details on `ZLayer`)
1. Define a type alias like `type ModuleName = Has[Service]` (see below for details on `Has`)

```scala mdoc:silent
import zio.{ Has, ZLayer }

type UserRepo = Has[UserRepo.Service]

object UserRepo {
  trait Service {
    def getUser(userId: UserId): IO[DBError, Option[User]]
    def createUser(user: User): IO[DBError, Unit]
  }

  val testRepo: ZLayer[Any, Nothing, UserRepo] = ZLayer.succeed(???)
}
```

```scala mdoc:reset:invisible
import zio.{ Has, IO, Layer, UIO, URIO, ZEnv, ZIO, ZLayer }
import zio.clock.Clock
import zio.console.Console
import zio.random.Random

trait DBError
trait Product
trait ProductId

case class UserId(value: Long)
case class User(id: UserId, name: String)
```

We encountered two new data types `Has` and `ZLayer`, let's get familiar with them.

### The `Has` data type

`Has[A]` represents a dependency on a service of type `A`. Two `Has[_]` can be combined _horizontally_ through `++` operator, as in

```scala mdoc:invisible
object Repo {
  trait Service{}
}

object Logger {
  trait Service{
    def log(s: String): UIO[Unit] = UIO(???)
  }
}
```

```scala mdoc:silent
val repo: Has[Repo.Service] = Has(new Repo.Service{})
val logger: Has[Logger.Service] = Has(new Logger.Service{})

val mix: Has[Repo.Service] with Has[Logger.Service] = repo ++ logger
```

At this point you might ask: what's the use of `Has` if the resulting type is just a mix of two traits? Why aren't we just relying on trait mixins?

The extra power given by `Has` is that the resulting data structure is backed by an _heterogeneous map_ from service type to service implementation, that collects each instance that is mixed in so that the instances can be accessed/extracted/modified individually, still guaranteeing supreme type safety.

```scala mdoc:silent
// get back the logger service from the mixed value:
val log = mix.get[Logger.Service].log("Hello modules!")
```

As per the recipe above, it is extremely convenient to define a type alias for `Has[Service]`.
Usually we don't create a `Has` directly, but we do that through `ZLayer`.

### The `ZLayer` data type

`ZLayer[-RIn, +E, +ROut <: Has[_]]` is a recipe to build an environment of type `ROut`, starting from a value `RIn`, possibly producing an error `E` during creation.

In adherence with environmental concepts, the absence of a required input is represented by `RIn = Any`, conveniently  used in the type alias `Layer`.

There are many ways to create a `ZLayer`, here's an incomplete list:
 - `ZLayer.succeed` or `ZIO.asService`  to create a layer from an existing service
 - `ZLayer.succeedMany` to create a layer from a value that's one or more services
 - `ZLayer.fromFunction` to create a layer from a function from the requirement to the service
 - `ZLayer.fromEffect` to lift a `ZIO` effect to a layer requiring the effect environment
 - `ZLayer.fromAcquireRelease` for a layer based on resource acquisition/release. The idea is the same as `ZManaged`
 - `ZLayer.fromServices` to build a layer from a number of required services

Where it makes sense, these methods have also variants to build a service effectfully (suffixed by `M`), resourcefully (suffixed by `Managed`), or to create a combination of services (suffixed by `Many`).

We can compose `layerA` and `layerB`  _horizontally_ to build a layer that has the requirements of both layers, to provide the capabilities of both layers, through `layerA ++ layerB`

We can also compose layers _vertically_, meaning the output of one layer is used as input for the subsequent layer to build the next layer, resulting in one layer with the requirement of the first and the output of the second layer: `layerA >>> layerB`

## Wiring modules together
Here we define a module to cope with CRUD operations for the `User` domain object. We provide also an in memory implementation of the module

```scala mdoc:silent
type UserRepo = Has[UserRepo.Service]

object UserRepo {
  trait Service {
    def getUser(userId: UserId): IO[DBError, Option[User]]
    def createUser(user: User): IO[DBError, Unit]
  }


  // This simple live version depends only on a DB Connection
  val inMemory: Layer[Nothing, UserRepo] = ZLayer.succeed(
    new Service {
      def getUser(userId: UserId): IO[DBError, Option[User]] = UIO(???)
      def createUser(user: User): IO[DBError, Unit] = UIO(???)
    }
  )

  //accessor methods
  def getUser(userId: UserId): ZIO[UserRepo, DBError, Option[User]] =
    ZIO.accessM(_.get.getUser(userId))

  def createUser(user: User): ZIO[UserRepo, DBError, Unit] =
    ZIO.accessM(_.get.createUser(user))
}
```

Then, we define another module to perform some basic logging. We provide also a `consoleLogger` implementation that relies on zio `Console`

```scala mdoc:silent
type Logging = Has[Logging.Service]

object Logging {
  trait Service {
    def info(s: String): UIO[Unit]
    def error(s: String): UIO[Unit]
  }


  import zio.console.Console
  val consoleLogger: ZLayer[Console, Nothing, Logging] = ZLayer.fromFunction( console =>
    new Service {
      def info(s: String): UIO[Unit]  = console.get.putStrLn(s"info - $s")
      def error(s: String): UIO[Unit] = console.get.putStrLn(s"error - $s")
    }
  )

  //accessor methods
  def info(s: String): URIO[Logging, Unit] =
    ZIO.accessM(_.get.info(s))

  def error(s: String): URIO[Logging, Unit] =
    ZIO.accessM(_.get.error(s))
}
```

The accessor methods are provided so that we can build programs without bothering about the implementation details of the required modules, the compiler will infer fully all the required modules to complete the task

```scala mdoc:silent
val user2: User = User(UserId(123), "Tommy")
val makeUser: ZIO[Logging with UserRepo, DBError, Unit] = for {
  _ <- Logging.info(s"inserting user")  // URIO[Logging, Unit]
  _ <- UserRepo.createUser(user2)       // ZIO[UserRepo, DBError, Unit]
  _ <- Logging.info(s"user inserted")   // URIO[Logging, Unit]
} yield ()
```

Given a program with these requirements, we can build the required layer:
```scala mdoc:silent
// compose horizontally
val horizontal: ZLayer[Console, Nothing, Logging with UserRepo] = Logging.consoleLogger ++ UserRepo.inMemory

// fulfill missing deps, composing vertically
val fullLayer: Layer[Nothing, Logging with UserRepo] = Console.live >>> horizontal

// provide the layer to the program
makeUser.provideLayer(fullLayer)
```

## Providing partial environments
Let's add some extra logic to our program that creates a user

```scala mdoc:silent

val makeUser2: ZIO[Logging with UserRepo with Clock with Random, DBError, Unit] = for {
    uId       <- zio.random.nextLong.map(UserId)
    createdAt <- zio.clock.currentDateTime.orDie
    _         <- Logging.info(s"inserting user")
    _         <- UserRepo.createUser(User(uId, "Chet"))
    _         <- Logging.info(s"user inserted, created at $createdAt")
  } yield ()
```

Now the requirements of our program are richer, and we can satisfy them partially by providing our custom layers, and leaving out the layers that are covered by the standard environment `ZEnv`, in one line of code

```scala mdoc:silent
  val zEnvMakeUser: ZIO[ZEnv, DBError, Unit] = makeUser2.provideCustomLayer(fullLayer)
```

Notice that `provideCustomLayer` is just a special case of `provideSomeLayer`.

## Updating local dependencies
Given a layer, it is possible to update one or more components it provides. One way is through a function to replace an old service with new service

```scala mdoc:silent
val withPostgresService = horizontal.update[UserRepo.Service]{ oldRepo  => new UserRepo.Service {
      override def getUser(userId: UserId): IO[DBError, Option[User]] = UIO(???)
      override def createUser(user: User): IO[DBError, Unit] = UIO(???)
    }
  }
```

Another way is by composing horizontally with a layer that provides the updated service

```scala mdoc:silent
val dbLayer: Layer[Nothing, UserRepo] = ZLayer.succeed(new UserRepo.Service {
    override def getUser(userId: UserId): IO[DBError, Option[User]] = ???
    override def createUser(user: User): IO[DBError, Unit] = ???
  })

val updatedHorizontal2 = horizontal ++ dbLayer
```

## Dealing with managed dependencies
Some components of our applications need to be managed, meaning they undergo a resource acquisition phase before usage, and a resource release phase after usage (e.g. when the application shuts down). `ZLayer` relies on the powerful `ZManaged` data type and this makes this process extremely simple.

For example, to build a postgres-based repository we need a `java.sql.Connection` to be opened at startup, and closed in the release phase.

```scala mdoc:silent
import java.sql.Connection
def makeConnection: UIO[Connection] = UIO(???)
val connectionLayer: Layer[Nothing, Has[Connection]] =
    ZLayer.fromAcquireRelease(makeConnection)(c => UIO(c.close()))
val postgresLayer: ZLayer[Has[Connection], Nothing, UserRepo] =
  ZLayer.fromFunction { hasC =>
    new UserRepo.Service {
      override def getUser(userId: UserId): IO[DBError, Option[User]] = UIO(???)
      override def createUser(user: User): IO[DBError, Unit] = UIO(???)
    }
  }

val fullRepo: Layer[Nothing, UserRepo] = connectionLayer >>> postgresLayer

```

## Layers are shared in the dependency graph
One important feature of `ZIO` layers is that they are acquired in parallel wherever possible, and they are shared. For every layer in our dependency graph, there is only one instance of it that is shared between all the layers that depend on it. If you don't want to share a module, create a fresh, non-shared version of it through `ZLayer.fresh`.

Notice also that the `ZLayer` mechanism makes it impossible to build cyclic dependencies, making the initialization process very linear, by construction.

# Hidden Versus Passed Through Dependencies
One design decision regarding building dependency graphs is whether to hide or pass through the upstream dependencies of a service. `ZLayer` defaults to hidden dependencies but makes it easy to pass through dependencies as well.

To illustrate this, consider the postgres-based repository discussed above:

```scala mdoc:silent
val connection: ZLayer[Any, Nothing, Has[Connection]] = connectionLayer
val userRepo: ZLayer[Has[Connection], Nothing, UserRepo] = postgresLayer
val layer: ZLayer[Any, Nothing, UserRepo] = connection >>> userRepo
```

Notice that in `layer` the dependency of `UserRepo` on `Connection` has been "hidden" and is no longer expressed in the type signature. From the perspective of a caller, `layer` just outputs a `UserRepo` and requires no inputs. The caller does not need to be concerned with the internal implementation details of how the `UserRepo` is constructed.

This achieves an encapsulation of services and can make it easier to refactor code. For example, say we want to refactor our application to use an in memory database:

```scala mdoc:silent
val updatedLayer: ZLayer[Any, Nothing, UserRepo] = dbLayer
```

No other code will need to be changed, because the fact that the previous implementation used a `Connection` was hidden from users and so they were not able to rely on it.

However, if an upstream dependency is used by many other services it can be convenient to "pass through" that dependency and include it in the output of a layer. This can be done with the `>+>` operator, which provides the output of one layer to another layer, returning a new layer that outputs the services of _both_ layers.

```scala
val layer: ZLayer[Any, Nothing, Connection with UserRepo] = connection >+> userRepo
```

Here, the `Connection` dependency has been passed through and is available to all downstream services. This allows a style of composition where the `>+>` operator is used to build a progressively larger set of services, with each new service able to depend on all the services before it.

```scala mdoc:invisible
type Baker = Has[Baker.Service]
type Ingredients = Has[Ingredients.Service]
type Oven = Has[Oven.Service]
type Dough = Has[Dough.Service]
type Cake = Has[Cake.Service]

object Baker {
  trait Service
}

object Ingredients {
  trait Service
}

object Oven {
  trait Service
}

object Dough {
  trait Service
}

object Cake {
  trait Service
}
```

```scala mdoc
lazy val baker: ZLayer[Any, Nothing, Baker] = ???
lazy val ingredients: ZLayer[Any, Nothing, Ingredients] = ???
lazy val oven: ZLayer[Any, Nothing, Oven] = ???
lazy val dough: ZLayer[Baker with Ingredients, Nothing, Dough] = ???
lazy val cake: ZLayer[Baker with Oven with Dough, Nothing, Cake] = ???

lazy val all: ZLayer[Any, Nothing, Baker with Ingredients with Oven with Dough with Cake] =
  baker >+>       // Baker
  ingredients >+> // Baker with Ingredients
  oven >+>        // Baker with Ingredients with Oven
  dough >+>       // Baker with Ingredients with Oven with Dough
  cake            // Baker with Ingredients with Oven with Dough with Cake
```

`ZLayer` makes it easy to mix and match these styles. If you pass through dependencies and later want to hide them you can do so through a simple type ascription:

```scala mdoc:silent
lazy val hidden: ZLayer[Any, Nothing, Cake] = all
```

And if you do build your dependency graph more explicitly you can be confident that layers used in multiple parts of the dependency graph will only be created once due to memoization and sharing.