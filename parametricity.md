---

# Functional Programming's most powerful super power : Parametricity in Scala

---

In this post I will explain parametricity with code examples, this concept is essential in the road to convince yourself that there is something more profound to functional programming that meets the eye.

---

# Type parameters

Type parameters are like normal parameters but at the type level 

```scala

def f[A](a: A, b: String): String = "hi"

```

`A` is a type parameter, the caller will decide what it is when he calls the function


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

# Let's play the guess my implementation game

Parametricity was defined in [Theorems for free!](http://ecee.colorado.edu/ecen5533/fall11/reading/free.pdf) by Philip Walder, here is a click-bait quote from the introduction

> "Write down the definition of a polymorphic function on
a piece of paper. Tell me its type, but be careful not
to let me see the function’s definition. I will tell you a
theorem that the function satisfies."

Basically he claims that given any polymorphic function he can guess things that are true about this function or in a sense guess its implementation !

Seems magical, but I guarantee you by the end of this post you will also know how to do it.

--- 

# How many implementation can this function have ?

Disclaimer: The functions seems like toy examples but there is a point to it so hang on !


```scala

def f[A](a: A): A = ???

```

Keep in mind that you don't know anything about `A`, it can be any type


Answer : 1


```scala

def f[A](a: A): A = a


```

We restrict ourselves to Scala FP safe subset, no `toString`, `hashCode` built in functions, no exceptions.


## Let's add another parameter `b` of the same parametric type 

```scala
scala> def f[A](a: A, b: A): A = ???
f: [A](a: A, b: A)A
```

Answer : 2

```scala
scala> def f[A](a: A, b: A): A = a
f: [A](a: A, b: A)A

scala> def f[A](a: A, b: A): A = b
f: [A](a: A, b: A)A
```

## Let's make `b` a fixed typed `String` instead of a parametric type 

```scala
scala> def f[A](a: A, b: String): A = ???
f: [A](a: A, b: String)A
```

Answer : 1

```scala
scala> def f[A](a: A, b: String): String = b
f: [A](a: A, b: String)String
```
Note that because we are restricting ourselves to Scala safe subset we are not allowed to do something like this : 

```scala
scala> def f[A](a: A, b: String): String = a + b
f: [A](a: A, b: String)String

scala> f(1, "a")
res3: String = 1a
```

What does the `+` operator do ? 
 
- `a` is converted to `String` using the illegal `toString` default method of every java object

- `+` finds `(String, String)` parameters, it can now concatenate them 

Not good ! We don't like surprises so we are not allowing this.


## We make the type richer we use `List[A]`

```scala
scala> def f[A](as: List[A]): List[A] = ???
f: [A](as: List[A])List[A]
```

Answer : A little bit more than 1, but still small



```scala
scala> def f[A](as: List[A]): List[A] = as
f: [A](as: List[A])List[A]

scala> def f[A](as: List[A]): List[A] = if (as.isEmpty) Nil else as.tails.toList.head
f: [A](as: List[A])List[A]

scala> def f[A](as: List[A]): List[A] = if (as.isEmpty) Nil else as.head :: Nil
f: [A](as: List[A])List[A]
```

We are still fairly limited in our possible implementations and that is because
we don't know anything about specific `A`s. We cannot sort them as we don't have an ordering, we are just stuck to return a subset of the original list.


### We add a new type parameter `B`

```scala
scala> def f[A,B](as: List[A]): List[B] = ???
f: [A, B](as: List[A])List[B]
```

Answer : 1


```scala
scala> def f[A,B](as: List[A]): List[B] = Nil
f: [A, B](as: List[A])List[B]
```


---

### We Changed the return type to a fixed `Int` type

```scala
scala> def f[A](a: A): Int = ???
f: [A](a: A)Int
```

Answer : 2^32 + 1 (Number of `Int`s)

We can completely ignore the input type here and start implementing away


```scala
scala> def f[A](a: A): Int = 1
f: [A](a: A)Int

scala> def f[A](a: A): Int = 2
f: [A](a: A)Int

scala> def f[A](a: A): Int = 3
f: [A](a: A)Int

scala> def f[A](a: A): Int = 4
f: [A](a: A)Int
```

---

### A pattern emerges


|  # of parameters | # of type parameters | # of implementations  |
|:----------------:|:--------------------:|:-----:|
|1                 |1                     |1      |
|2                 |1                     |2      |
|2                 |2                     |1      |
|1                 |0(ignored it returns`Int`)|2^32 + 1      |



By introducing a type parameter, we hide information from the implementation of the function .

The implementation space is reduced because we cannot use any property on this type (think about `List[A]` we cannot sort `A`s because we know nothing about it)

Reduced possibilities of implementation means reduced possibilities of errors in code !

What's important to see here, is making function more parametric is not about making them more general for greater code reuse it's about reducing the number of possible implementations.

---

### Where is parametricity used ?

Parametricity is heavyly used in typeclasses here are a few examples



```scala

trait Show[T] {
  def show(f: T): String
}

trait Eq[A] {
  def eq(a: A, b: A): String
}

```

`Show[T]` provides a textual representation of type `T`, you can think of it has the reasonable version of `toString`
`Eq[A]`  used to determine equality between 2 instances of the same type, it's basically better version of the built-in non-safe `==`


What's important here is the that they use 1 type parameter


Now some more advanced ones

```scala

trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}


trait Applicative[F[_]] extends Functor[F] {
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]

}

```

I will not delve in what these type classes are, what matters is they use MORE type parameters, they also use type constructors `F[_]` (if you want an explanation on what this is go to my previous post [Understanding `F[_]` in Scala](https://medium.com/bigpanda-engineering/understanding-f-in-scala-4bec5996761f))


```scala

trait Traverse[F[_]] extends Functor[F] { 

  def traverse[G[_]: Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]

}

```

`traverse` is interesting because it asks from one of its type parameter `G[_]` to also be an `Applicative` re-using a previous type class.


The point here is not to scare you off, but to give you a sense that these abstract signatures make things easier at the implementation stage and not harder, if you go look up their implementation in a library like cats or scalaz you will see very little code there BECAUSE of the high parametricity


---

# Takeawawys

- Type parameters are like normal parameters but at the type level `f[A](a: A)`
- Parametric functions do not use fixed types like `String`, `Int`, ...
- Parametricity is NOT about code re-use, it's a nice side effect though !
- By introducing type parameters, we hide information from the implementation of the function
- By hiding information we lower the implementation space and thus reduce the amount of possible mistake
- Typeclasses make heavy use of parametricity, you can get a sense of it by looking at how small their implementations are

---

# References

[mpilquist
 FSiS video](https://www.youtube.com/watch?v=D0Fnzr15BAU)
[Tony Morris - Parametricity](https://www.youtube.com/watch?v=qBvFsA3dglk)
