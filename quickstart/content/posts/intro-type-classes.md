+++ 
draft = false
date = 2021-09-20T22:29:52-04:00
title = "Introduction to Type Classes in Scala"
slug = "" 
tags = []
categories = []
thumbnail = "images/tn.png"
description = ""
+++

# Introduction

A quick Google search of type class may yield the following (arguably vague / abstract) phrases or definitions:

- Retroactive extension or retroactive / ad-hoc polymorphism

- Decoupling type definition from associated type methods

In this post, I hope to distill these ideas a bit and provide a soft introduction to type classes and how they're defined and used in Scala.

# Definition

In short, type classes are a way associating desired functionality to some type; these types may be native Scala types (e.g. String, Int, etc), or they can be specific to your problem domain (e.g. Person, Shop, etc). Because these types may not have been originally designed to perform this desired functionality, we say this is a form of ad-hoc or retroactive extension.

##### Defining a Type Class

In Scala, we define a type class by defining a trait with at least 1 type parameter; you can think of this trait as a bag of desired functionalities that we can retroactively extend onto the type(s) in the type class definition.

Suppose we're interested in functionality that allows one to express whether a value of some type is false-y or truth-y. This maybe useful in languages like JavaScript, where a `Number` value of `0` is false-y, and anything else is truth-y. Let's define the type class to encode this:

```
trait Truthiness[A] {
  def isTruthy(value: A): Boolean
  def isFalsey(value: A): Boolean
}
```

##### Defining Type Class Instances

Now that we've define the _Truthiness_ type class, we need to provide some implementations of the type class for types for which we care about. Note, this opt-in approach makes sense because for some types, it may make sense to have this _Truthiness_ functionality, but for others, it's hard to conceptualize what a false-y or truth-y value is. Implementations of a type class are referred to as **type class instances**, and they are defined by creating actual instances of the type class and tagging them with the _implicit keyword_. Let's define one for Scala integers:

```
implicit val truthinessInInts: Truthiness[Int] =
  new Truthiness[Int] {
    def isTruthy(value: Int): Boolean = value != 0
    def isFalsey(value: Int): Boolean = value == 0
  }
```

##### Using your Type Class

Awesome! Now we have a type class with an implementation for `Int`'s. Let's create and expose an API that will allow users to determine whether a value of any type is considered false-y or truth-y. Let's create a singleton object to contain our API:

```
  object TruthinessAPI {
    def isTruthy[A](value: A)(implicit t: Truthiness[A]): Boolean = t.isTruthy(value)
    def isFalsey[A](value: A)(implicit t: Truthiness[A]): Boolean = t.isFalsey(value)
  }
```

and I can use this API as follows: `TruthinessAPI.isTruthy(0) // false`.

Underneath the hood, the Scala compiler was able to _summon_ an instance (i.e. truthinessInInts) to perform the functionalities required by the _Truthiness_ type class for Ints. Trying to do something like `TruthinessAPI.isTruthy("0")` will yield an error that informs you that the compiler could not find an implementation of _Truthiness_ for Strings.

# Parting Words and Resources

This was meant to be a basic, superficial introduction to type classes in Scala, and I hope you're able to leave with a better intuition and understanding of the phrases mentioned at the beginning of this post. Feel free to copy all the code in your own Scala workspace and experiment with it.

To learn more about type classes in Scala, these are two resources I used to grasp the concept:

1. [Daniel Westheide: Neophytes Guide to Scala - Type Classes](https://danielwestheide.com/blog/the-neophytes-guide-to-scala-part-12-type-classes/)
2. [Underscore.io: Scala with Cats](https://underscore.io/books/scala-with-cats/)
