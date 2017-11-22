# Refactoring to functional patterns in Scala

If you have a background in Java like me you probably read
[Refactoring To Patterns](https://industriallogic.com/xp/refactoring/). It's a very cool book about refactorings that shows you how to refactor Object Orientated code step by step and eventually reach full blown Gang of Four design patterns. It had a huge impact on me at the time. It left me with the feeling that code is alive and wants to be rearranged this way, and that patterns emerge naturally.

Fast forward 10 years, and I work in a very cool startup ([Bigpanda](https://bigpanda.io/careers/)) where we use Scala and Functional programming for our back end services.
Working with FP is very different, and in my opinion far easier. The are no more complicated class hierarchies and no more complicated patterns, only functions, functions, functions. If GoF design patterns are no longer our destination, then the refactoring steps we must take are very different. 

In this post I'll introduce you to some approaches to refactoring functional code. I will build from simple refactorings to more advanced ones using the State and Writer monads - functional design patterns.

## Make sure you have a full suite of tests with good coverage

Refactoring without tests is like jumping without a safety net. You can use sbt as a very useful continuous test runner:

```scala

~testQuick io.company.service.PricingServiceSpec

```

Each time you save your file it will recompile it and rerun only the previously failing tests 


## Eliminate primitive types with value classes

*Before*

```scala
def buy(lastPrice: String, name: String, exchange: String): (Double, Long) = ???
```

*After*

```scala
case class Symbol(name: String) extends AnyVal
case class Exchange(name: String) extends AnyVal
case class Price(value: Long) extends AnyVal
case class Timestamp(ts: Long) extends AnyVal

def buy2(lastPrice: Price, symbol: Symbol, exchange: Exchange): (Price, Timestamp) = ???
```

We now get type safety for primitives and the function signatures are more
expressive.
Also value classes will not allocate runtime objects , it will just use the wrapped primitives 

We have a package called `types` and we will put all our value classes in `values.scala` file
We will also add `Ordering` implicits there.

```scala
implicit val timestampOrdering: Ordering[Timestamp] = Ordering.by(_.ts)
```

## Rewrite on the side and then switch the functions

Typically I do not start by deleting old code. First I write the new function on the side, make sure it compiles
and then switch the old ones and make sure the tests pass. This is very handy trick to let you backtrack 
quickly if you have an error somewhere.

## Align the types between functions

If your functions compose together in a natural way, it means that you have found the right level of abstraction.

Keep them small and focused on one thing, add type annotations for the return types to increase readability.

If you find that you need to work hard with type transformations to be able to compose your functions together then try this: 

- Inline, inline, inline, and retry.

After a while you get that hang of it and your functions will be focused and compose together. You can also do some upfront design.

Personally I found [A Type Driven Approach to Functional Design](https://www.infoq.com/presentations/Type-Functional-Design#.WgQgsvnDY9Q) helpful. It's in Haskell but it is still very relevant and will give you a sense of how to design functions that compose together.

## Use State monad for functions that need previously computed values 

Let's define some types to work with:

```scala
sealed abstract class CreditRating(val rating: Int)

case class Good(points: Int) extends CreditRating(points)
case class Fair(points: Int) extends CreditRating(points)
case class Poor(points: Int) extends CreditRating(points)


case class PriceEvent(symbol: Symbol, price: Price, ts: Timestamp)

```

In any meaningful service you will need previously computed data. You'll also want to persist it in case you crash or restart your app. This lead to sateful functions.

In order to rate a stock we need the previous prices and rating, this usually leads to long ugly parameter lists. Because our data structures are immutable so we have to return new updated versions of them.

*Before*

```scala
def rateStock(historicalPrices:  Map[Symbol, List[(Timestamp, Price)]],
             lastRatings: Map[Symbol, CreditRating],
             symbol: Symbol, 
             newPrice: Price): (Map[Symbol, CreditRating], List[(Timestamp, Price)]) = ???

```

Quite ugly!

This is a very common pattern in FP. You can use `State` monad to communicate to the reader
that they are about to deal with stateful functions.

We use cat's [State](https://typelevel.org/cats/datatypes/state.html).

We encapsulate the moving parts in our own defined type : 

```scala
case class StockState(historicalPrices: Map[Symbol, List[(Timestamp, Price)]],
                      lastRatings: Map[Symbol, CreditRating])
```

We use `State` to cleanup the parameter list and the return type : 

```scala
import cats.implicits._
import cats.data.State

def rateStatefulStock(symbol: Symbol, newPrice: Price): State[StockState, CreditRating] = ???
```
We can improve the type readability with a type alias:

*After*

```scala

type StateAction[A] = State[StockState, A]
def rateStatefulStock(symbol: Symbol, newPrice: Price): StateAction[CreditRating] = ???

```

The function is far cleaner and it can compute and update the ratings from the previous state.

This gives us the ability to chain state functions as follows and be guaranteed that each function
receive the correct latest updated state, very cool !!!

```scala

for {

  a <- rateStatefulStock(Symbol("AAPL"), Price(145.5))	

  // something magical happens here, 
  // it passes on the correct StockState to the next function

  s <- rateStatefulStock(Symbol("SAMSNG"), Price(2123.3))	

} yield (a, s)

```

## Use `Writer` monad to track state transitions when using `State`

If you work with Event sourcing you will want to recreate your state from all
the transitions you carried out. In order to keep track of state transitions without
complicating your function you can use `Writer` monad to log all the transitions in a List.

First let's define some more types:  

```scala
  sealed trait Transition
  case class UpgradedRating(newRating: CreditRating) extends Transition
  case class DowngradedRating(newRating: CreditRating) extends Transition
```

We want to use State and Writer together, so let's use WriteT to combine them:

```scala
  import cats.data.WriterT 

  type StateActionWithTransitions[A] = WriterT[StateAction, List[Transition], A]
```

Use this function to log transitions and add it to the final transition list: 


```scala
  def archive(evts: List[Transition]): StateActionWithTransitions[Unit] =
    WriterT.tell(evts)
```

Boilerplate to wire up State and Writer together

```scala
  def lift[A](s: StateAction[A]): StateActionWithTransitions[A] =
    WriterT.lift(s)
```

Pure functions have simple return types that are not wrapped in `StateActionWithTransitions`.
This tells the the reader that this function does not change the state.


```scala
  def calculateRating(stock: Symbol, old: CreditRating, newPrice: Price): CreditRating = 
    if (stock.name == "AAPL") Good(1000) else if(newPrice.value == 0) Poor(0) else Fair(300) 

  def calculateTransition(oldRating: CreditRating, newRating: CreditRating): Transition = 
    if(newRating.rating > oldRating.rating) UpgradedRating(newRating) else DowngradedRating(oldRating)
```

Stateful functions have the return type  `StateActionWithTransitions`. This tells the reader
to pay special care because this function uses or updates the state: 

```scala
  import com.softwaremill.quicklens._ 

  def setNewRating(symbol: Symbol, newRating: CreditRating): StateActionWithTransitions[Unit] = 
   lift(State.modify(_.modify(_.lastRatings).using(_ + (symbol -> newRating))))

  def getRating(s: Symbol): StateActionWithTransitions[CreditRating] = 
    lift(State.inspect[StockState, CreditRating](_.lastRatings.get(s).getOrElse(Poor(0))))
```

Here is the final version of our function:

- Whenever the reader sees `<-` he knows to pay special attention as it is a stateful function 
- Whenever the reader sees `=` he knows it's a pure function and nothing related to state happens there

```scala
  def rateStatefulStock(symbol: Symbol, newPrice: Price): StateActionWithTransitions[CreditRating] =   
    for {
  	  oldRating <- getRating(symbol)
  	  newRating = calculateRating(symbol, oldRating, newPrice)
  	  _ <- setNewRating(symbol, newRating)	
  	  transition = calculateTransition(oldRating, newRating)
  	  _ <- archive(transition :: Nil)
    } yield newRating
```

## Takeaways
 
- Before refactoring make sure you have good tests with decent coverage 
- Strongly type as much as you can, use meaningful names and abstractions
- Design your functions so their types align and compose together
- Use cats's `State` data type to write functions that need state
- Use type aliases to cleanup boilerplate types
- Use cat's `Writer` data type to log state transitions
