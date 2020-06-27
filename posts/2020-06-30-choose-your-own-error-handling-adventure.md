---
layout: post
title: Choose Your Own Error-Handling Adventure

meta:
  nav: blog
  author: mtomko
  pygments: true
  mathjax: false

tut:
  scala: 2.13.2
  binaryScala: "2.13"
  scalacOptions:
  dependencies:
    - org.scalatest::scalatest:3.1.2
    - org.scalacheck::scalacheck:1.14.1
    - org.typelevel::cats-core:2.1.0
    - org.typelevel::cats-effect:2.1.3
---

```tut:invisible
import cats.data.NonEmptyList
import cats.effect.{ContextShift, IO}
import cats.implicits._
import cats.MonadError

implicit val cs: ContextShift[IO] = IO.contextShift(scala.concurrent.ExecutionContext.global)
```

# Error Handling
Proper error handling is possibly one of the least glamorous aspects
of software development. Although the situation is much better when
using functional techniques like `MonadError` and `ApplicativeError`,
dealing with errors can still be a bit confusing or hard to do well.

In this blog post, we will look at some general guidelines for purely
functional error handling. Then we will provide some more detailed
descriptions of the various error handling methods provided by common
error types `ApplicativeError` and `MonadError` and discussion of when
each one is appropriate. Finally we will provide a "choose your own
adventure" style table for choosing the appropriate error handling
technique to help guide you to the right method.

## How to handle errors
Dealing with failures can be a frustrating task in any programming
environment. Deciding when and where to raise an error, as well as
when and how to handle an error can be difficult. If we include
error-handling deep down in our code, we may pollute an otherwise
general method with specific details that pertain only in specific
situations. However, by handling errors too far out, we may lose
access to critical context that we need for our own debugging, or to
provide users with information they need to solve problems.

Unfortunately, these kinds of questions are beyond the scope of this
blog post. However, by making it easy to choose the right
error-handling techniques, I hope to make it easier to focus your
efforts on solving the truly hard problems inpvolved in
error-handling, and to remove doubts about what methods to call in
what situations given a desired result.

## Preliminaries
We would like to use the simplest `MonadError` possible for the rest
of this post, so let's put in a few preliminaries:

```tut:silent
import cats.MonadError

// partially apply the type for Either
type ME[A] = Either[Throwable, A]

// summon an MonadError for `ME` that we can refer to later
val me: MonadError[ME, Throwable] = MonadError[ME, Throwable]
```

## Raise vs. Throw
One of the first questions that's worth addressing in the context of
typeclasses like `ApplicativeError` and `MonadError` is when to
`throw` and when to use `raiseError`. In the case of `Either` (and,
thus, `ME`), we always need to capture non-fatal errors explicity:
```tut:fail
me.catchNonFatal(throw new RuntimeException("doh!"))

me.pure(throw new RuntimeException("doh!"))
```

If you are using an `IO` monad or the `Sync` typeclass, calling
`delay` or `suspend` will automatically catch non-fatal errors and
sequence them into the effect type:


```tut
IO.raiseError(new RuntimeException("doh!")).attempt.unsafeRunSync

IO.delay(throw new RuntimeException("doh!")).attempt.unsafeRunSync
```

In functional code, using `throw` seems a little odious, so we
generally prefer code like:

```tut:silent
IO(1).flatMap { x => 
  if (x < 2) IO.raiseError(new RuntimeException("Too low")) 
  else IO.pure(x)
}
```

to:

```tut:silent
IO(1).map { x =>
  if (x < 2) throw new RuntimeException("Too low")
  else IO.pure(x)
}
```

even though both will evaluate to an `IO[Int]` that contains a
`RuntimeException`. In general, you should prefer `raiseError` to
`throw`, however there is a catch: `raiseError` will always blindly
trap any error you pass to it. If you are attempting to raise a
_fatal_ error and bypass the normal error handling mechanisms, you
must `throw` it:

```tut
me.raiseError(new OutOfMemoryError("Oom noom!"))
```

With that, let's turn to how we can handle errors that we may
encounter in a running application.

## Methods on MonadError
We will consider the following methods defined by the `MonadError` and
`ApplicativeError` typeclasses:

### `attempt`

Use `attempt` to convert an `F[A]` into an `F[Either[Throwable, A]]`.
This is useful if you don't want to cause an entire chain of `map` or
`flatMap` calls to fail due to one failure - for example, if you want
to deal with failures later on, or prefer to do error accumulation.
Here's an example:

```tut
val fs: List[IO[Int]] = Nil

fs.traverse(_.attempt.map(_.toEitherNel)).map(_.parSequence)

```

### `adaptError`
### `handleError`
### `handleErrorWith`
### `onError`
### `recover`
### `recoverWith`
### `redeem`
### `redeemWith`
### `rethrow`


## Choose Your Own (Table)

| recover | partial | return | method |
|---------|---------|--------|--------|


