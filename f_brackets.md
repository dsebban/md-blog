# Understanding `F[G[_]]` in Scala

In order to really understand this syntax, we will define some basic concepts and then
relate them together, it should all make sense at the end.

# What is a value ?

The simplest concept is a value, values represents raw data, the lowest level 
abstraction that exist

```scala
val name = "daniel"
val one = 1
val oneAndTwoAsTuple = (1,2)
val oneAndTwoInAList = List(1,2)
```

As you see on the right side it's pure date, very simple for your brain. 
For example you can talk to a child
and tell him that a toy costs 2$, he would understand that, try now to explain what 
$ is, you have to go explain currencies and all kind of other complicated concepts.


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

Look at what the REPL spits as information , he spits types, `String`, `List[Int]`, ...
There are all proper types, it's a higher level concept, types can be *instantiated* and 
produce a value. 
`String` can produce all the string literals you can think up ("a", "ab", ...), if 
we go back to our currency analogy $ can be instantiated to (2$, 3$, ...) or whatever else
you can think up.
We just jumped one level on the abstraction scale, your brain can still follow, let's move on to the next level.

# What is a first-order type ?

In the previous example we said that `List[Int]` is a proper type, but what about
`List` ?

```scala
scala> val l: List = List(1,2,3)
<console>:12: error: type List takes type parameters
       val l: List = List(1,2,3)
              ^
```

This obviously does not compile, `List` in itself is meaningless, it needs something, it misses something: `List[_]` 
There is a hole , in order to exist you need to fill the hole with a type, it's like a parameter to a function. thus it's  also called a **type constructor**
Other familiar **type constructor** : `Option[_]`, `Array[_]`, `Map[_,_]`, ...
Notice that `Map` is the same , it needs a type for the key and the value, it has 2 holes, 2 parameters.

You would think , that's it and in a lot of languages that's all you have, but in scala and other functional languages you
have on more concept , let's do the last jump

# What abstracts over a first-order type ?

As you notice each abstraction , abstract over the previous one

```scala
List("a") -> List[String] ->    List      ->          ???
// value     // proper       //first order   // abstraction over List
```

In Scala you can abstract over a first-order type with this syntax


```scala

trait WithMap[F[_]] {

}

```
Forget about `WithMap` for a second, what is interesting is the `F[_]` syntax, this is the way to
say a first-order type with one hole for example `List[_]`, `Option[_]`.


Now the million dollar question 

- What is `WithMap` ?

Answer : **A second-order type**

It's a type which abstract over types which abstract over types !!!

Very inception like, but hopefully you followed until here so it's all clear now.


# Higher kinded types

Let's make things simple, each time you see a type constructor `[_]` 
this is called a *higher kinded type*

A type constructor is just a function that takes a  type and returns a type.

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
