# Understanding `F[_]` in Scala

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

# What is a higher-order type ?

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

Answer : **A higher-kinded type**

It's a type which abstract over types which abstract over types !!!

Very inception like, but hopefully you followed until here so it's all clear now.


# Why do I need `F[_]` ?

We abstracted over all the first order types with one hole, we can now
define common functions between all of them for example

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


# [BONUS] Kinds ?

Now that it all makes sense, there is a different way to talk about types, namely *kinds*

- A proper type is of kind : * 

- A higher order type with one hole is of kind : `* -> *` (takes one type and produce an proper type) 
  `List[_]` takes a `String` and produce a List[String]

- A higher order type with two holes is of kind : `* -> * -> *` (takes two types and produce an proper type) 
  `List[_]` takes a `String` and produce a List[String]

- A higher kinded-type s of kind : `(* -> *) -> *` (take a higher order type with one hole and produce a proper type )

- A higher kinded-type s of kind : `(* -> * -> *) -> *` (take a higher order type with two holes and produce a proper type )

This gives a visual way to talk about the type of types.







