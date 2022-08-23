+++ 
draft = false
date = 2022-08-21T22:41:04-04:00
title = "Tagless Final: Separating a Program's Logic From its Interpretation"
slug = "" 
tags = []
categories = []
thumbnail = "images/tn.png"
description = ""
+++

# The Premise

![abracadabara](https://static.vecteezy.com/system/resources/thumbnails/000/963/090/small/cartoon-man-with-to-do-list-on-clipboard.jpg)

Suppose you are tasked with implementing a basic to-do application in Scala from scratch; the specifics of this application are not so important at this point, but one way or another, you and the team arrive to the same conclusion that the application must be able to support some persistence capabilities. To that end, you go to the drawing board and brainstorm the following trait expressing some intended persistence functionalities:

```scala
case class Todo(id: UUID, content: String, due: OffsetDateTime)

/**
  Trait that provides persistence capabilities for [[Todo]] entities.
**/
trait TodoRepository {

  update(todo: Todo): Future[Unit]

  find(id: UUID): Future[Option[Todo]]

  /** ...et al **/
}

```

With this basic set of persistence functionalities, your first assignment in the application is to implement a **reschedule** functionality; that is, given a [[Todo]] and a later date, update the former's **due date** value to be the latter. Here's your first stab at it:

```scala
case class RescheduleTodoService(todoRepository: TodoRepository) {
  def reschedule(id: UUID, later: OffsetDateTime): Future[Either[String, Unit]] =
      todoRepository.find(id).flatMap {
        case Some(todo) =>
          val rescheduled = todo.copy(due = later)
          todoRepository.update(rescheduled).map(Right(_))
        case None => Future.value(Left("No matching Todo item"))
      }
}
```

While this compiles just fine, there are a few talking points worth discussing.

##### Return type of Future[Either[String, Unit]]

---

Notice that the return type of the `reschedule` method is `Future[Either[String, Unit]]`. In this case, we are using a `Left(String)` to indicate that some error might have occurred (e.g. _No matching Todo item_). While it is highly advised to consider a better error model using some ADT, our actual focus is on the usage of `Future` here.

We are forced into using `Future` here because the `TodoRepository` returns values wrapped in `Future`. This may appear odd because the usage of `Future` inherently expresses asynchronous operations, but if you think about it, our `reschedule` function can just contain _pure business logic_ (i.e. what it means to reschedule a [[Todo]]). In other words, this design has backed us into a corner and made it so that we've coupled together our program's logic with its interpretation.

##### Future.flatMap and Future.value

---

Expanding on the previous point, if you carefully inspect the `reschedule` implementation, you may realize that you just need to be able to sequence computations with `flatMap` as well as to create _pure_ values with `Future.value`. If this sounds familiar to you, it's because these are the characteristics of a `Monad`! In the next sections, we'll see how we can rewrite the code a bit to abstract away our dependency on the concrete `Future` type; before we get to that, we must familiarize ourselves with the idea of Tagless Final.

# Enter Tagless Final

Tagless Final is a well known pattern that attempts to solve many of the issues we've see. In general, the pattern comprises of three core concepts: _algebras_, _interpreters_, and _programs_.

## Algebra

---

Simply put, an algebra is a trait that enumerates all the capabilities that a component should be able to perform. Most importantly, in order to solve the issue of committing to the specific `Future` type as we've done before, algebras leverage an ingredient that goes by several names: _kind_, _higher-kinded type_, _type constructor_, _generic effect type_, etc. To see this in action, we'll rewrite `TodoRepository` and supply its algebra:

```scala
  trait TodoRepositoryAlgebra[F[_]] {

    def find(id: UUID): F[Option[Todo]]

    def update(todo: Todo): F[Unit]
  }
```

See what we've done here? We abstracted over the _"effect type"_ and delayed the decision to choose which concrete type until later.

## Interpreters

---

The decision to lock in a specific effect type occurs when writing an _interpreter_. Interpreters are simply implementations of an algebra; i.e. they interpret the algebra and define _how_ something works.

Suppose that in our production data centers, we choose to interpret persistence capabilities with SQL. Then, we may have a SQL interpreter as follows:

```scala
  class TodoRepositorySQLInterpreter(client: SQLClient) extends TodoRepository[Future] {
    override def find(id: UUID): Future[Option[Todo]] = ???

    override def update(todo: Todo): Future[Unit] = ???
  }
```

Just as we created an interpreter to be able to persist [[Todo]] entities to some SQL data store, we could just as easily create a _test interpreter_:

```scala
  class TodoRepositoryTestInterpreter() extends TodoRepository[Either[String, *]] {

    override def find(id: UUID): Either[String, Option[Todo]] = {
      val todo = Todo(id, "some content", OffsetDateTime.now())
      Right(Some(todo))
    }

    override def update(todo: Todo): Either[String, Unit] = Right(())
  }
```

This approach makes testing pure business logic really straightforward, as it removes the need to _mock_ out several of our dependencies. After all, we're only interested in testing our business logic, which will more often than not be found in the next section: _programs_.

> Note: a concrete effect type can be used in interpreters, or interpreters can be parameteric on F[_] the whole way. In this case, we chose to commit on Future, since perhaps the SQL client only exposes a Future based implementation,

## Programs

---

Algebras and interpreters by themselves don't achieve much (and that's the point!); we deliberately put business logic inside _programs_, which rely on one or more algebras. Turning full circle, we complete our rewrite of the `reschedule` program we saw earlier:

```scala
  import cats.Monad
  import cats.syntax.all._

  case class RescheduleTodoProgram[F[_]: Monad](todoRepository: TodoRepository[F]) {

    def reschedule(id: UUID, later: OffsetDateTime): F[Either[String, Unit]] = todoRepository.find(id).flatMap {
      case Some(todo) =>
        val rescheduled = todo.copy(due = later)
        todoRepository.update(rescheduled).map(Right(_))
      case None => "No matching to do item".asLeft[Unit].pure[F]
    }
  }
```

In writing our `reschedule` program, we denote with a _typeclass constraint_ that whatever effect type we interpret, it must necessarily be monadic (i.e. support the `flatMap` and `pure` operations).

Observe that this program doesn't leak any information on interpration (i.e. no suggestions of asynchronous operations as we saw previously). What's inside a program describes pure business logic, and that's it.

# Summary

In summary, by following the Tagless Final pattern, we were able to see the following benefits:

1. We saw how unsanitary it can be to couple a program's logic with its interpretation. With Tagless Final algebras, we enumerate a set of functionalities a component should have, but we don't commit on any concrete implementation that point.

2. We saw how we can create production interpreters and test interpreters to facillitate our testing stories.

3. We "Future" proofed our application since in the future, if we decide that we want to use a different effect type (e.g. Cats IO, Monix Task, etc), we can make the switch rather painlessly.
