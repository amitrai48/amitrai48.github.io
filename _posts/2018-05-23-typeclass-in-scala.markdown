---
layout: post
title:  "Typeclass in Scala"
date:   2018-05-23
categories: Scala Typeclass
comments: true
---

Before we look into typeclass pattern in Scala, it would help if we understand basics of polymorphism, because typeclasses in Scala helps in [adhoc polymorphism](#adhoc-polymorphism).<!--excerpt-->


## Polymorphism

The same **operation** working on different **types** of values. Broadly speaking there are three types of polymorphism:

* Parametric Polymorphism
* Subtype Polymorphism
* Adhoc polymorphism

### Parametric Polymorphism

Defined over a range of types, acting in the same way for each type, i.e *the difference in type is ignored*. Example, the `length` function acts in the same way for `List[Int]` or `List[String]` or `List[User]`. Another example is `head` function which takes a `List[A]` and returns `A`. And it does not matter what the `A` is: It could be `Int`, `String` etc.

```scala
scala> def head[A](xs: List[A]): A = xs(0)
head: [A](xs: List[A])A

scala> head(List(1,2,3))
res0: Int = 1

scala> head(List(User(1, "Ginny"), User(2, "Harry")))
res1: User = User(1,Ginny)
```

### Subtype Polymorphism

Subclasses provide different implementations of some superclass method. The decision on which implementation is being invoked is made at **runtime**. Let us consider a simple example of a function plus which adds two values of type `A`.


```scala
def plus[A](a: A, b: A): A = ???
```

The implementation for the above function depends on type `A`. Based on type, the definition of add changes. One way to solve them is through subtyping.

```scala
scala> trait Plus[A] {
	def plus(b: A): A
}
defined trait Plus

scala> def plus[A <: Plus[A]](a: A, b: A): A = a.plus(b)
plus: [A <: Plus[A]](a: A, b: A)A

```

`plus[A <:Plus[A]]` tells that A is a subtype of Plus. We can now provide different definition of `plus` for `A`. But it is not very flexible since trait Plus needs to be mixed in at the time of defining datatype. So can't work with `Int`, `String` or any other datatype which was not defined by you.

### Adhoc polymorphism

From the paper [How to make ad-hoc polymorphism less ad hoc, by Philip Wadler and Stephen Blott](https://www.cse.iitk.ac.in/users/karkare/courses/2010/cs653/Papers/ad-hoc-polymorphism.pdf) adhoc polymorphism is defined as 

*Ad-hoc polymorphism occurs when a function is
defined over several different types, acting in a different
way for each type.*

While *Parametric Polymorphism* too was defined for several types but it was agnostic of type, adhoc polymorphism depends on type and the definition changes based on type. *Method Overloading* is one way of achieving adhoc polymorphism. Imagine the same plus function overloaded for different types:

```scala
def plus(a: Int, b: Int): Int = a + b

def plus(a: String, b: String): String = a.concat(b)
```

Uh Oh!!! We would now need to write the plus function for all the types we care about. Hence method overloading is not widely used. Another way is:

```scala
scala> trait Plus[A] {
     	def plus(a: A, b: A): A
       }
defined trait Plus
scala> def plus[A](a: A, b: A)(p: Plus[A]): A = p.plus(a,b)
plus: [A](a: A, b: A)(p: Plus[A])A
```
The plus function now take 2 variable of type `A` and also a `p` of type `Plus[A]` which knows how to plus them. By doing this we can give any instance of Plus for any type. Below is the example for some:

```scala
scala> object IntPlus extends Plus[Int] {
         def plus(a: Int, b: Int): Int = a + b
       }
defined object IntPlus

scala> object StringPlus extends Plus[String] {
         def plus(a: String, b: String): String = a.concat(b)
       }
defined object StringPlus

scala> plus(2,3)(IntPlus)
res0: Int = 5

scala> plus("Harry", "Potter")(StringPlus)
res1: String = HarryPotter
```

The above works but it's not very nice for below reasons:
* Too verbose do we really need to pass the second argument `IntPlus`, `StringPlus` etc all the time.
* Standard GOF pattern nothing unique.

**Implicits Incoming**

What if we made the second argument an implicit so that we don't have to pass it all the time. Below is the refined code: 

```scala
scala> def plus[A](a: A, b: A)(implicit p: Plus[A]): A = p.plus(a,b)
plus: [A](a: A, b: A)(implicit p: Plus[A])A

scala> implicit object IntPlus extends Plus[Int] {
         def plus(a: Int, b: Int): Int = a + b
       }
defined object IntPlus

scala> implicit object StringPlus extends Plus[String] {
         def plus(a: String, b: String): String = a.concat(b)
       }
defined object StringPlus

scala> plus(2,3)
res0: Int = 5
```

We have now eliminated the verbosity of the previous implementation. We can further take advantage of another syntactic sugar called `Context Bound`.

```scala
scala> def plus[A: Plus](a: A, b: A): A = implicitly[Plus[A]].plus(a,b)
plus: [A](a: A, b: A)(implicit evidence$1: Plus[A])A
```
No more implicit in the function argument.


## Typeclass

Typeclasses are used to enable [adhoc polymorphism](#adhoc-polymorphism). Type class (see how type class is sperated by space) is a language feature in haskell, but in scala typeclass(hmm.. no space) is a design pattern. Adhoc polymorphism helps in creating common behavior of classes without using the subtyping polymorphism and hence more flexible.

### Anatomy of typeclass

1. Define some functionality `T` as `trait`.
2. Provide default implementation for the types you are interested in.
3. Expose the type class behavior to user

Let's see how the above three parts were fulfilled in our `Plus` example.

#### 1 - Define functionality

```scala
trait Plus[A] {
     	def plus(a: A, b: A): A
       }
```

#### 2 -Typeclass implementations

```scala
object PlusInstances {
	implicit object IntPlus extends Plus[Int] {
         def plus(a: Int, b: Int): Int = a + b
       }

	implicit object StringPlus extends Plus[String] {
         def plus(a: String, b: String): String = a.concat(b)
       }
}
```

#### 3 - Typeclass interface (exposing the behavior to user)


This is done in two ways

**1. Interface Objects**
```scala
object Plus{
	def plus[A: Plus](a: A, b: A): A = implicitly[Plus[A]].plus(a,b)
}
```
**2. Interface Syntax (using implicit classes)**
```scala
object PlusSyntax{
	implicit class PlusOps[A](a: A) {
    	def plus(b: A)(implicit p: Plus[A]): A = {
        	p.plus(a, b)
        }
    }
}
```

The above lets you call `plus` directly.

```scala
scala> import PlusInstances._
import PlusInstances._

scala> import PlusSyntax._
import PlusSyntax._

scala> "a".plus("b")
res1: String = ab
scala> 2.plus(5)
res2: Int = 7

scala> "a".plus("b")
res3: String = ab
```

Typeclasses are extensively used in Scala functional programming libraries such as [ScalaZ](https://github.com/scalaz/scalaz) and [Cats](https://typelevel.org/cats/) and is a very widely used pattern. I hope the above notes helped in demystify typeclasses in Scala.