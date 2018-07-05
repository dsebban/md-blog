---

# Understanding Parametricity in Scala

---

In this post I will explain a functional programming concept called parametricity with code examples.

This concept is essential in the road to convince yourself that there is something more profound to functional programming that meets the eye.

---

# Type and value parameters

```scala

def f[A](a: A, b: String): String = "hi"

```
`a` and `b` are __value__ parameters, the one you use most of the time.

`A` is a __type__ parameter, the caller will decide what it is when he calls the function


```scala
scala> f("hello", "panda")
res0: String = hi

scala> f(2, "panda")
res1: String = hi

scala> f(List(1,2,3), "panda")
res2: String = hi
```

As you can see I can pass whatever I want in the first parameter, which seems to give more generality, but something more is going on here

---

# What is Parametricity?

Parametricity was defined in [Theorems for free!](http://ecee.colorado.edu/ecen5533/fall11/reading/free.pdf) by Philip Walder, here is a click-baity quote from the introduction

> "Write down the definition of a polymorphic function on
a piece of paper. Tell me its type, but be careful not
to let me see the function’s definition. I will tell you a
theorem that the function satisfies."

Basically he claims that given any function with type parameters he can guess things that are true about this function or in a sense guess its implementation !

Seems magical, but I guarantee you by the end of this post you will also know how to do it.

--- 

# Let's make this concrete with a game

Rules of the game: we operate in the FP world, so no exceptions, `println` or any nonsense like this.
Also you are not allowed to use the built-in `toString`, `hashCode`, or others methods that come from the `Object` base class in Java/Scala.

Your task: 
Find out __How many implementation can this function have ?__

We will go through different code examples, playing with their type parameters and types and see how it affects the number of implementations.

## Let's start!

```scala

def f[A](a: A): A = ???

```

`A` is a type parameter, it can be any type.
We can implement this way:

```scala

def f[A](a: A): A = a

```

And that's it !
Answer: 1
Here is the only possible answer, that's interesting, try to ask yourself why is it the only possibility ?
We will answer this later.

--- 

## Let's add another value parameter `b`

Nothing much changed here, we only added `b`, let's see how it affects the implementation space


```scala

def f[A](a: A, b: A): A = ???

```
We can implement this way:

```scala

def f[A](a: A, b: A): A = a
def f[A](a: A, b: A): A = b

```

Answer: 2
Notice you can't do anything between `a` and `b`, even if they are from the same type, we cannot do this for example ```a + b```.
Again ask yourself why? soon you will see the whole picture and will become clear

--- 

## Let's make `b` a fixed typed `String` instead of a parametric type 

```scala

def f[A](a: A, b: String): A = ???

```
We can implement this way:

```scala

def f[A](a: A, b: String): String = b

```

Answer: 1
By the way if you were tempted to use  `+`, well don't! 
The rules of the game also prohibit it and here is why:

```scala

def f[A](a: A, b: String): String = a + b
```

```scala
scala> f(1, "a")
res3: String = 1a
```

What does the `+` operator do ? 
 
- `a` is converted to `String` using the illegal `toString` default method of every java object

- `+` finds `(String, String)` parameters, it can now concatenate them 

Not good ! We don't like surprises so we are not allowing this.

--- 

### We make the type richer with `List[A]`

```scala

def f[A](as: List[A]): List[A] = ???

```

We can implement this way:

```scala

def f[A](as: List[A]): List[A] = as
def f[A](as: List[A]): List[A] = if (as.isEmpty) Nil else as.tails.toList.head
def f[A](as: List[A]): List[A] = if (as.isEmpty) Nil else as.head :: Nil

```

Answer: A little bit more than 1, but still not so many

We are still fairly limited in our possible implementations and that is because
we don't know anything about specific `A`s. We cannot sort them as we don't have an ordering, we are just stuck to return a subset of the original list.

--- 

### We add a new type parameter `B`

```scala

def f[A,B](as: List[A]): List[B] = ???

```

We can implement this way:

```scala

def f[A,B](as: List[A]): List[B] = Nil

```

Answer: 1
Interesting, we are back to one when adding a new type parameter.

---

### We change the return type to `Int` type

```scala

def f[A](a: A): Int = ???

```

We can completely ignore the input type here and start implementing away


```scala

def f[A](a: A): Int = 1
def f[A](a: A): Int = 2
def f[A](a: A): Int = 3
def f[A](a: A): Int = 4

```

Answer: 2^32 + 1 (Number of `Int`s)

We jumped to another dimension here, so many possible implementations ...

---

# Let's recap


|  # value parameters | # type parameters | # implementations  |
|:----------------:|:--------------------:|:-----:|
|1                 |1                     |1      |
|2                 |1                     |2      |
|2                 |2                     |1      |
|1                 |0(ignored it returns`Int`)|2^32 + 1      |



- By introducing type parameters we __hide information__ from the implementation of the function.
- The __implementation space is reduced__ because we cannot use any properties on this type (think about `List[A]` we cannot sort `A`s because we know nothing about individual `A`s)
- Making a function __more parametric__  will reducing the number of possible implementations.
- It will also make your code __more general__ as we can plug any type we want instead of our type parameters.
- Reduced possibilities of implementation means __reduced possibilities of errors in code__ !

---


# Takeaways

- Type parameters are like normal value parameters but at the type level `f[A](a: A)`
- Parametric functions do not use fixed types like `String`, `Int`, ...
- By introducing __type parameters__, we __hide information from the implementation__ of the function
- By hiding information we __lower the implementation space__ and thus __reduce the amount of possible mistakes__
- This notion is called __Parametricity__
- Parametricity is NOT about __code re-use__, it's a nice side effect though !

---

# References

- [mpilquist
 FSiS video](https://www.youtube.com/watch?v=D0Fnzr15BAU)
- [Tony Morris - Parametricity](https://www.youtube.com/watch?v=qBvFsA3dglk)
