---
layout: post
title: "Algebraic Data Types"
subtitle: "Huh? What are they good for?"
description: "Algebraic data types, usually abbreviated ADTs, are a useful construct to know how to use."
header-img: "img/mon-monmouth.jpg"
authors:
    -
        name: "John Klingler"
        githubProfile : "jfklingler"
        twitterHandle : "jfklingler"
        avatarUrl : "https://en.gravatar.com/userimage/53272847/ea13fb59ab7158db2518972472255f90.jpg"
tags: [scala, algebraic, adt]
---
Algebraic data types, usually abbreviated ADTs, are a useful construct to know how to use. At a high level, they're somewhat similar to enums in Java, but Scala lends much more power to the construct. ADTs define a fixed set of all possible values of a given type. The classic example of an ADT in Scala is the [`Option`](http://www.scala-lang.org/api/current/index.html#scala.Option) type. An `Option` represents something that may or may not have a value. No value is `None` and some value is `Some()`. This is the functional version of returning an object or null from a method in Java, and is very useful for Java interoperability to eliminate `NullPointerException`'s.

So to see how we might apply this in our own code, we're going to take the ever-popular FizzBuzz and show how it could be implemented using ADTs. Just as a refresher, let's look at how FizzBuzz might be implemented in Java.

```java
public static void printIt(Integer i) {
  if (i % 3 == 0 && i % 5 == 0) {
    System.out.println("FizzBuzz");
  } else if (i % 3 == 0) {
    System.out.println("Fizz");
  } else if (i % 5 == 0) {
    System.out.println("Buzz");
  } else {
    System.out.println(String.valueOf(i));
  }
}

public static void main(String[] args) {
  for (int i = 1; i <= 100; i++) {
    printIt(i);
  }
}
```

Nothing out of the ordinary here. If we just translated this into Scala, we'd have something probably about like this.

```scala
def printIt(i: Int): Unit =
  if (i % 3 == 0 && i % 5 == 0)
    println("FizzBuzz")
  else if (i % 3 == 0)
    println("Fizz")
  else if (i % 5 == 0)
    println("Buzz")
  else
    println(String.valueOf(i))

for (i <- 1 to 100) {
  printIt(i)
}
```

But, we've been using Scala for a while now so we know there are more functional ways to do this. There are certainly different way to accomplish this, but it might look something like this.

```scala
val isMod: (Int, Int) => Boolean = (m, i) => i % m == 0
val mod3 = isMod(3, _)
val mod5 = isMod(5, _)

val fizzBuzzIt: Int => String = {
  case i if mod3(i) && mod5(i) => "FizzBuzz"
  case i if mod3(i)            => "Fizz"
  case i if mod5(i)            => "Buzz"
  case i                       => i.toString
}

Stream(1 to 100: _*).map(fizzBuzzIt).foreach(println)
```

So that works just fine - it does exactly what we needed. But what if we want to share the FizzBuzzification functionality as a library? We could just publish our `fizzBuzzIt` function. That would work, but what if users wanted to do something different depending on whether it's a Fizz or a Buzz. They'd have to resort to string matching which is brittle and definitely not type safe. One typo in their string matching would result in incorrect behavior or failures at runtime. Sure, they should have written tests to catch that, but what if they made the same typo, or defined a constant with the typo? The tests would never catch it.

Here is where can leverage ADTs to get the job done. First off, we need to define the type of our collection of values.

```scala
sealed abstract class FizzBuzzADT(i: Int) {
  override def toString: String = i.toString
}
```

There are two things to note here. First, we've defined a `sealed` class. If you're not familiar that, it basically tells the compiler that the only types that are allowed to inherit from this class are in the same file. What this means is that a user is not allowed to add more values to our type. This is important because adding more values somewhere else might lead to, for example, a MatchError being thrown in a pattern match that doesn't handle this new value. Second, we've declared shared functionality for all values - standard OO stuff.

Now we need to define all of the possible values of our ADT. Remember, these all have to be in the same file as the trait.

```scala
case class Fizz(i: Int) extends FizzBuzzADT(i) {
  override val toString = "Fizz"
}

case class Buzz(i: Int) extends FizzBuzzADT(i) {
  override val toString = "Buzz"
}

case class FizzBuzz(i: Int) extends FizzBuzzADT(i) {
  override val toString = "FizzBuzz"
}

case class JustInt(i: Int) extends FizzBuzzADT(i)
```

We've defined `Fizz`, `Buzz`, `FizzBuzz`, and `JustInt`. It might not be clear yet why we've defined constructor arguments for `Fizz`, `Buzz`
, and `FizzBuzz`, but it gives us some opportunites that we'll talk about in a minute.

So all we need now is a way to create our values given any integer. The `FizzBuzzADT` companion object is the perfect place for this.

```scala
object FizzBuzzADT {
  def apply(i: Int): FizzBuzzADT = i match {
    case _ if i % 3 == 0 && i % 5 == 0 => FizzBuzz(i)
    case _ if i % 3 == 0               => Fizz(i)
    case _ if i % 5 == 0               => Buzz(i)
    case _                             => JustInt(i)
  }
}
```

Here you see that the logic for determining whether the integer is a "Fizz", "Buzz", "FizzBuzz", or just an integer, hasn't changed or gone away. But instead of just printing the value, we're mapping the integers to one of the values of our ADT. So to run our FizzBuzz algorithm and print the values, we can do this.

```scala
Stream(1 to 100: _*).map(FizzBuzzADT(_)).foreach(println)
```

So you're probably thinking that this doesn't look all that different from our "idiomatic Scala" implementation from above. Scala is supposed to be more concise. What did all that extra code buy us? Well, remember the constructor arguments to the case classes? Let's say, for example, that we have a collection of `FizzBuzzADT`'s (but no access to the collection of integers from which it was created) and we only want to print something out if the original integer was even. We can do this using pattern matching.

```scala
val even: Int => Boolean = i => i % 2 == 0

// Pretend the Stream[Int] came from somewhere else
Stream(1 to 100: _*).map(FizzBuzzADT(_)).foreach {
  case a@Fizz(i) if even(i)     => println(a)
  case a@Buzz(i) if even(i)     => println(a)
  case a@FizzBuzz(i) if even(i) => println(a)
  case a@JustInt(i) if even(i)  => println(a)
  case _                        => // Be quiet
}
```

Now, certainly, we could have accomplished this example more effectively, by exposing the original integer as a value in the abstract class, but the purpose here was to demonstrate the power and flexibility of pattern matching with ADTs.

For another example, let's suppose for a second that I don't like Fizz. In fact, I hate it; I never want to see it. Let's get rid of the Fizz.

```scala
Stream(1 to 100: _*).map(FizzBuzzADT(_)).foreach {
  case Fizz(_) => // Down with the bloody bad Fizz
  case x       => println(x)
}
```
In this case, we don't really need pattern matching and we can accomplish the same thing more concisely.

```scala
Stream(1 to 100: _*).map(FizzBuzzADT(_)).filter(!_.isInstanceOf[Fizz]).foreach(println)
```

Sure, you could just as easily have ridden the world of Fizz with the implementation above.

```scala
Stream(1 to 100: _*).map(fizzBuzzIt).filter(_ != "Fizz").foreach(println)
```

But are you sure there's not a typo? What if it's decided that Fizz should be spelled Fisz? These all end up being runtime errors - we don't like runtime errors. We have to write tests for these scenarios and even those tests are prone to the same problems. The ADT turns these runtime errors into compile time errors. And since we can't even run tests if the code doesn't compile, we've eliminated a whole class of tests. And these tests weren't truly valuable anyway - they validated library integration, not business functionality. So, in that sense, ADT's also let us spend more brain cycles on business functionality.

The FizzBuzz ADT isn't truly the best example for demonstrating the power of ADTs, but it did serve to show how you can take an algorithm from a rather brittle form susceptible to runtime errors to a type-safe form with compile-time errors. Hopefully I've demystified algebraic data types and shown how they're valuable constructs to understand and how you can employ them in your code to build more robust applications.

All code shown can be found [here](https://github.com/jfklingler/algebraic-datatypes) or just `git clone git@github.com:jfklingler/algebraic-datatypes.git`.
