# Understanding `F[_]` in Scala

This very abstract syntax comes up all the time in Scala, 
I will try to give you an intuition of what it means and how to use it.

# Overview

The goal of this post is to understand what this syntax and why you would need it.
In order to do so we  will gradually climb the ladder of abstractions and
answers the following questions : 

- What is a value?
- What is a proper type?
- What is a first-order type ?
- What abstracts over a first-order type ?
- Why do I need `F[_]` ?

# What is a value ?

Values represent raw data. They have the lowest level of abstraction and are the simplest concept that we need to deal with. 

```scala
val name = "daniel"
val one = 1
val oneAndTwoAsTuple = (1,2)
val oneAndTwoInAList = List(1,2)
```

Take a look at the right hand side of these examples - it's just data and it's trivial to understand.

If a child asks you what your funky BigPanda tshirt costs and you answer $12 then they'll understand what you mean. They'll certainly understand the value in your answer (2). But if they ask you what a dollar is then suddenly things get more complicated. Explaining money and currencies is a bit more tricky. This takes us to types.

# What is a proper type ?

```scala
scala> val name = "daniel"
name: String = daniel

scala> val one = 1
one: Int = 1

scala> val oneAndTwoAsTuple = (1,2)
oneAndTwoAsTuple: (Int, Int) = (1,2)

scala> val oneAndTwoInAList = List(1,2)
oneAndTwoInAList: List[Int] = List(1, 2)
```

Look at the information the REPL spits out. It keeps telling you about types: `String`, `List[Int]` etc. These are all proper types.

Proper types are a higher level concept than values. Let's talk about how they are related: types can be *instantiated* to produce a value and values are a specific *instance* of a type.

`String` can produce all the string literals you can think up ("a", "ab", "algorithmic service operations" etc). If we go back to our pricing example, $ can be instantiated to $2, $3, [$49,000,000](https://bigpanda.io/resources/bigpanda-expands-series-b-funding-49-million/)...) or any other amount.

Moving from values to proper types took us up a level of abstraction. What do we get if we go one higher?

# What is a first-order type ?

In the previous example we said that `List[Int]` is a proper type, but what is`List`?

```scala
scala> val l: List = List(1,2,3)
<console>:12: error: type List takes type parameters
       val l: List = List(1,2,3)
              ^
```

This doesn't compile. The compiler won't let us say that a value is a `List`. It wants us to say that it is a list of something, a `List[_]`.
 
There's slot there. If we want the compiler to give us a type we need it put something in the slot. It's like a parameter to a function that returns a type. There's a name for this special kind of function: a **type constructor**.

You've probably met other **type constructors**: `Option[_]`, `Array[_]`, `Map[_,_]` and friends. Notice that `Map` is a little different; it needs a type for the key and for the value. It has 2 slots for 2 parameters.

First-order types are just types (`List`, `Map`, `Array`) that have type constructors (`List[_]`, `Map[_, _]`) that take proper types and produce proper types (`List[Int]`, `Map[String, Int]`). 

Going from proper types to first-order types tooks us up a layer of abstraction. In most programming languages you can't abstract any further. However, Scala let's you got a step further.  Let's take that last step and see where it takes us.

# What abstracts over a first-order type ?

Every step we've taken so far has added an abstraction over the previous abstraction: 

```scala
List("a") -> List[String] -> List           -> ???
// value     proper type     first order type  ???
```

In Scala you can abstract over a first-order type with this syntax:


```scala

trait WithMap[F[_]] {

}

```
Let's forget about `WithMap` for a second to focus on the `F[_]` syntax. `F[_]` represents a first-order type with one slot. For example `List[_]` or `Option[_]`.


Now the million dollar question: What is `WithMap`?

Answer : **A second-order type**

It's a type which abstracts over types which abstract over types!!!

Feel like inception, right? Hopefully you followed until here and everything is starting to fall into place.

# Higher kinded types

Let's introduce one more piece of terminology, and then try and clarify how everything fits together.

A type with a type constructor (ie. a type with `[_]`) is called a *higher kinded type*. A type constructor is just a function that takes a type and returns a type.

Let's do a quick analogy between types and functions :

- A type constructor `List[_]` is just a function of type 

```scala

T => List[T]

```
For example:

```scala
String => List[String]

```

Given a proper type it will return another proper type you can think about it as 
a function that works at the type level, a type level function.

But wait we returned only a proper type, what if we return another first order type : 


```scala

List[_] => WithMap[List[_]]

```

Or generalized to any **one-hole** type

```scala

F[_] => WithMap[F[_]]

```
Give a type level function we return another type level function.

Higher order functions are functions that returns functions, in the same

# The `*` notation 

The type of a type is called kind and uses `*` as notation to communicate what order they are.

- `String` is of kind `*` and is Order 0

- `List[_]` is of kind `* -> *` (takes one type and produce a proper type, Order 1) 
   takes a `String` and produce a List[String]

- `Map[_,_]` is of kind: `* -> * -> *` (takes two types and produce a proper type, Order 1) 
  takes a `String,Int` and produce a Map[String,Int]

- `WithMap[F[_]]`  of kind : `(* -> *) -> *` (take a Order 1 type `(* -> *)` and produce a proper type, Order 2)

This gives a visual way to talk about the type of types.


# Why do I need `F[_]` ?

We abstracted over all the first order types with one hole, we can now
define common functions between all of them for example :

```scala

trait WithMap[F[_]] {

def map[A,B](fa: F[A])(f: A => B): F[B]

}

```

You can mentally replace `F` by `List` or `Option` or any other first-order types.
This allows us to define a `map` function over all first-order types. 

Yes that's it, it allows us to define functions across a lot of different types in a concise
way, this is very powerful but is not in the scope of this post. Just remember that now you have a 
way to talk about a range of types based on how many holes they have and not on what they
represent (`Option`, `List`) 


# Takeaways

- `1`, `"a"`, `List(1,2,3)` are values

- `Int`, `String`, List[Int] are proper types 

- `List[_]`, `Option[_]`  are type constructors, takes a type and construct a new type,
can be generalized with this syntax `F[_]`

- `G[F[_]]` is a type constructor that takes another type constructor like, 
`Functor[F[_]]`, can be tough of higher order function at type level
