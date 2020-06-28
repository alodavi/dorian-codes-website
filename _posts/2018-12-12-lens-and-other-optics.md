---
layout: post
title: "Functional references: Lens and other Optics in Scala"
date: 2018-12-12
---

Functional programming languages, like Scala, make it possible to ensure immutability and referential transparency. But how this two concepts cope with updating values inside an object?

<img class="responsive-img" src="/assets/img/2018/camera_on_piano.jpeg">

Let’s suppose we have to design a system to store addresses:

```scala
case class Street(name: String, number: Int)
case class Address(country: String, city: String, street: Street)
```

and we want to access the number from the Address . If we were to implement this functionality in an imperative object oriented way we could, for example, make number a mutable parameter:

```scala
case class Street(name: String, var number: Int)
case class Address(country: String, city: String, street: Street) {
  def changeStreetNumber(int: Int): Unit = {
    this.street.number = int
  }
}
```

Scala though offers a function called copy to modify the parameters value inside a case class. This function doesn’t mutate the referred value, instead it creates a new object :

```scala
case class Street(name: String, number: Int)
case class Address(country: String, city: String, street: Street) {
  def changeStreetNumber(int: Int): Address = this.copy(
    street = street.copy(
      number = int
    )
  )
}
```

With a functional programming mindset we could then decide to separate the creation of the object from its functionality and move the changeStreetNumber function outside of the class scope:

```scala
case class Street(name: String, number: Int)
case class Address(country: String, city: String, street: Street)

def changeStreetNumber(address: Address, int: Int): Address = address.copy(
  street = address.street.copy(
    number = int
  )
)
```

But let’s suppose that we have to change a more deeply nested object:

```scala
case class Street(name: String, number: Int)
case class Address(country: String, city: String, street: Street)
case class User(id: Long, address: Address)
case class Account(id: Long, user: User, isActive: Boolean)

def changeStreetNumber(account: Account,
                       int: Int): Account =
  account.copy(
    user = account.user.copy(
      address = account.user.address.copy(
        street = account.user.address.street.copy(
          number = int
        )
      )
    )
  )
```

The greater the level of nesting of the objects, the less readable the syntax becomes.

## Introducing Lenses

Let’s take a step backwards and have a look to what we’re trying to achieve here.

A computer program that accesses data is said a reference. Using mutable variables we make implicit use of references. Indeed the reference cells can hold any value and are of reference type a ref, where a is to be replaced with the type of value pointed to. If the reference is mutable, it can be pointed to different objects. An example of mutable reference in imperative programming languages are pointers.

In functional programming languages, in order to enforce immutability, other data structures are used in place of pointers — even if the compiler under the hood still uses them.

As the School of Haskell points out:

```plaintext
A lens is a first-class reference to a subpart of some data type.
```
We can define a Lens as follows:

```scala
case class Lens[A, B](
    get: A => B,
    set: (A, B) => A
)
```

In a less formal way, we can then describe the Lens as a group of functions, set and get, that allows us to manipulate data inside a class.

We have now a data structure that allows us to easily update the street number:

```scala
val streetNumberLens = Lens[Street, Int](
  get = _.number,
  set = (a, b) => a.copy(number = b)
)

val bigStreet = Street("Unter den Linden", 3)

streetNumberLens.get(bigStreet)
//res0: Int = 3

streetNumberLens.set(bigStreet, 9)
//res1: Street = Street(Unter den Linden,9)
```

So far so good, but besides giving us a better syntax and a more functional way to get and set a value in a case class, Lenses don’t seem to provide much.

Where is then the advantage of using a Lens in place of a nested copy function, if every time we have to create a new Lens? Here is the thing: we don’t have to.

Debasish Ghosh, in in the book Functional And Reactive Domain Modeling, defined a compose function that allows us to put together Lenses and reuse code:

```scala
def compose[Outer, Inner, Value](
    outer: Lens[Outer, Inner],
    inner: Lens[Inner, Value]
) = Lens[Outer, Value](
  get = outer.get andThen inner.get,
  set = (obj, value) => outer.set(obj, inner.set(outer.get(obj), value))
)
```

This is a powerful feature that allows us to create new Lenses from existing ones in a modular way.

```scala
val addressStreetLens = Lens[Address, Street](
  get = _.street,
  set = (a, b) => a.copy(street = b)
)

val addressStreetNumberLens: Lens[Address, Int] = compose(addressStreetLens, streetNumberLens)
```

So basically we can imagine that a Lens is like an instance of a function — or to be more accurate it’s an instance of a profunctor, a generalization of function.

In the Profunctor Optics Modular Data Accessors paper is indeed introduced in this way:
```scala
Any data accessor for a component of a data structure is ‘function-like’, in the sense that reading ‘consumes’ the component from the data structure and writing ‘produces’ an updated component to put back into the data structure. The type structure of such function-like things — henceforth transformers — is technically known as a profunctor.
```
In a notation that is not 100% accurate we could say that `Lens[A,B] ~ A => B` composed with this other `Lens[B,C] ~ B => C` gives `Lens[A,C] ~ A => C`.

## Lens Laws

A Lens is expected to satisfy general laws:

* Identity — If you get and then set back with the same value, the object remains identical:

```scala
  def getSet[S, A](lens: Lens[S, A], s: S): Boolean =
  lens.set(s, lens.get(s)) == s
```

* Retention — If you set with a value and then perform a get, you get the same value back:

```scala
  def setGet[S, A](lens: Lens[S, A], s: S, a: A): Boolean =
  lens.get(lens.set(s, a)) == a
```
* Double set — If you set twice in succession and then perform a get, you get back the last set value:

```scala
def putPut[S, A](lens: Lens[S, A], s: S, a: A, b: A): Boolean =
  lens.get(lens.set(lens.set(s, a), b)) == b
```
## Beyond Lenses: Optics

Lenses are not the only functional references we can think of. Generalizations of Lenses are called Optics.

As described in the Monocle documentation — where Monocle is a Scala library for Optics:

    Optics are a group of purely functional abstractions to manipulate (get, set, modify, …) immutable objects.

What if we want to manipulate data inside a trait , in general referred as a sum type or coproduct? Prisms come in handy. They’re like Lenses but for sum types.

```scala
//this is a simplification of Prism
case class Prism[S, A](_getOption: S => Option[A])(_reverseGet: A => S) {
  def getOption(s: S): Option[A] = _getOption(s)
  def reverseGet(a: A): S = _reverseGet(a)
}

val petPrism = Prism[Pet, String]{
  case Dog(n) => Some(n)
  case _ => None
}(name => Dog(name))

petPrism.getOption(Dog("Santa's Little Helper"))
res0: Option[String] = Some(Santa's Little Helper)

petPrism.reverseGet("Santa's Little Helper")
res1: Pet = Dog(Santa's Little Helper)
```

There is a generalization of Prism in case the object of type A may not exist, it’s called Optional.

```scala
//this is a simplification of Optional
case class Optional[S, A](_getOption: S => Option[A])(_set: A => S => S){
  def getOption(s: S): Option[A] = _getOption(s)
  def set(a: A): S => S = _set(a)
}

sealed trait Box
case class Present(quantity: Int) extends Box
case object NoPresent extends Box

val maybePresents = Optional[Box, Int] {
  case present: Present => Some(present.quantity)
  case _                => None
} { numberOfPresents => box =>
  box match {
    case present: Present => present.copy(quantity = numberOfPresents)
    case _                => box
  }
}

maybePresents.getOption(Present(3))
res0: Option[Int] = Some(3)

maybePresents.set(9)
res1: Box => Box = <function>
```

Unlike Prism when using set on Optional we lose information: we don’t have enough information to go back to S without additional argument.

We could go a step further and extend the logic behind Optional to traversable datatypes, such as List or Tree : in this case we would need an optic called Traversal. More on how a Traversal works can be found here.

## When to use Optics?

We’ve already seen that one possible use case would be in case of deeply nested objects. Another interesting use case is when you have to work with different representations of essentially the same data. circe-optics is based on this principle. More in general they would prove very useful in Parsers implementations.

The composition with the State Monad when updating to a new state would be another application of Optics.

Optics are a powerful instrument in the “Functional Programming Toolbox”, but they’re not always necessary. One thing is sure: the ability to compose optics gives us flexibility and expressiveness to traverse and update complex objects.

## Note

You can find the code mentioned above in [this gist](https://gist.github.com/alodavi/89d09435872ee629cc5d6a56e4bb81bb).

## References

* [Monocle documentation](http://julien-truffaut.github.io/Monocle/)
* [Profunctor Optics Modular Data Accessors](https://arxiv.org/ftp/arxiv/papers/1703/1703.10857.pdf) paper by Matthew Pickering, Jeremy Gibbons and Nicolas Wu
* [Optics beyond Lenses with Monocle blog post](https://blog.scalac.io/optics-beyond-lenses-with-monocle.html) by Michał Sitko
* [A Little Lens Starter Tutorial](https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/a-little-lens-starter-tutorial) by Joseph Abrahamson — School of Haskell
* [Scala Lens: An Introduction](https://www.slideshare.net/knoldus/scala-lens-an-introduction) presentation by Seth Tissue
* [Functional and Reactive Domain Modeling](https://www.manning.com/books/functional-and-reactive-domain-modeling) by Debasish Ghosh
* [Scala Exercises page](https://www.scala-exercises.org/monocle/lens)

From this [article](https://medium.com/zyseme-technology/functional-references-lens-and-other-optics-in-scala-e5f7e2fdafe) originally posted through [ZyseMe](https://medium.com/zyseme-technology).
