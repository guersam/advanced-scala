## Anatomy of a Type Class

There are three important components to the type class pattern: the *type class* itself, *instances* for particular types, and the *interface* methods that we expose to users.

### The Type Class

The *type class* itself is a generic type that represents the functionality we want to implement. For example, we can represent generic "serialize to JSON" behaviour as a generic trait.

```tut:book
// Define a very simple JSON AST
sealed trait Json
final case class JsObject(get: Map[String,Json]) extends Json
final case class JsString(get: String) extends Json
final case class JsNumber(get: Double) extends Json

// The "serialize to JSON" behavior is encoded in this trait
trait JsonWriter[A] {
  def write(value: A): Json
}
```

### Type Class Instances

The *instances* of a type class provide implementations for the types we care about, including standard Scala types and types from our domain model.

We define instances by creating concrete implementations of the type class and tagging them with the `implicit` keyword:

```tut:book
final case class Person(name: String, email: String)

object DefaultJsonWriters {
  implicit val stringJsonWriter = new JsonWriter[String] {
    def write(value: String): Json =
      JsString(value)
  }
  implicit val personJsonWriter = new JsonWriter[Person] {
    def write(value: Person): Json =
      JsObject(Map("name" -> JsString(value.name), "email" -> JsString(value.email)))
  }
  // etc...
}
```

### Interfaces

An *interface* is any functionality we expose to users. Interfaces to type classes are generic methods that accept instances of the type class as implicit parameters.

There are two common ways of specifying an interface: *Interface Objects* and *Interface Syntax*.

**Interface Objects**

The simplest way of creating an interface is to place the interface methods in a singleton object.

```tut:book
object Json {
  def toJson[A](value: A)(implicit writer: JsonWriter[A]): Json =
    writer.write(value)
}
```

To use this object, we import any type class instances we care about and call the relevant method.

```tut:book
import DefaultJsonWriters._

val json: Json = Json.toJson(Person("Dave", "dave@example.com"))
```

**Interface Syntax**

As an alternative, we can use *type enrichment* to extend existing types with interface methods[^pimping]. Cats refers to this as *"syntax"* for the type class:

[^pimping]: You may occasionally see enrichment referred to as "pimping". This is an older term that we don't use anymore.

```tut:book
object JsonSyntax {
  implicit class JsonWriterOps[A](value: A) {
    def toJson(implicit writer: JsonWriter[A]): Json = {
      writer.write(value)
    }
  }
}
```

We use interface syntax by importing it along-side the instances for the types we need:

```tut:book
import DefaultJsonWriters._
import JsonSyntax._

val json: Json = Person("Dave", "dave@example.com").toJson
```

### Exercise: *Printable* Library

Scala provides a `toString` method to let us convert any value to a `String`. However, this method comes with a few disadvantages. It is implemented for *every* type in the language, many implementations are of limited use, and we can't opt-in to specific implementations for specific types.

Let's define a `Printable` type class to work around these problems:

 1. Define a trait `Printable[A]` containing a single method `format`.
    `format` should accept a value of type `A` and returns a `String`.

 2. Create an object `PrintDefaults` containing default
    implementations of `Printable[A]` for `String` and `Int`.

 3. Define an object `Print` with two generic interface methods:

    - `format` accepts a value of type `A` and a `Printable` of the corresponding type. It uses the relevant `Printable` to return a
    `String` version of `A`;

    - `print` accepts the same parameters as `format` and returns `Unit`. It prints the `A` value to the console using `println`.

<div class="solution">
These steps define the three main components of our type class. First we define `Printable`---the *type class* itself:

```tut:book
trait Printable[A] {
  def format(value: A): String
}
```

Then we define some default *instances* of `Printable` and package then in `PrintDefaults`:

```tut:book
object PrintDefaults {
  implicit val stringPrintable = new Printable[String] {
    def format(input: String) = input
  }

  implicit val intPrintable = new Printable[Int] {
    def format(input: Int) = input.toString
  }
}
```

Finally we define an *interface* object, `Print`:

```tut:book
object Print {
  def format[A](input: A)(implicit printer: Printable[A]): String = {
    printer.format(input)
  }

  def print[A](input: A)(implicit printer: Printable[A]): Unit = {
    println(format(input))
  }
}
```
</div>

**Using the Library**

The code above forms a general purpose printing library that we can use in multiple applications. Let's define an "application" now that uses the library:

 1. Define a data type `Cat`:

    ```scala
    final case class Cat(name: String, age: Int, color: String)
    ```

 2. Create an implementation of `Printable` for `Cat` that returns content in the
    following format:

    ```
    NAME is a AGE year-old COLOR cat.
    ```

 3. Create an object `Main` to act as a front-end for your application. `Main` should create a `Cat` and print it to the console:

    ``` scala
    object Main extends App {
      val cat = Cat(/* ... */)

      // etc...
    }
    ```

<div class="solution">
This is a standard use of the type class pattern. First we define a set of custom data types for our application.

```scala
final case class Cat(name: String, age: Int, color: String)
```

Then we define type class instances for the types we care about. These either go into companion objects, or separate objects that act as namespaces.

```scala
object Cat {
  import PrintDefaults._

  implicit val catPrintable = new Printable[Cat] {
    def format(cat: Cat) = {
      val name  = Print.format(cat.name)
      val age   = Print.format(cat.age)
      val color = Print.format(cat.color)

      s"$name is a $age year-old $color cat."
    }
  }
}
```

```tut:silent
// Declare Cat and it's companion object together in a way the REPL understands,
// so that our later code works
object cat {
  final case class Cat(name: String, age: Int, color: String)
  object Cat {
    import PrintDefaults._

    implicit val catPrintable = new Printable[Cat] {
      def format(cat: Cat) = {
        val name  = Print.format(cat.name)
        val age   = Print.format(cat.age)
        val color = Print.format(cat.color)

        s"$name is a $age year-old $color cat."
      }
    }
  }
}
import cat._
```

Finally, we use the type class by bringing the relevant instances into scope and using interface object/syntax. If we defined the instances in companion objects Scala brings them into scope for us automatically. Otherwise we use an `import` to access them:

```tut:book
object Main extends App {
  val cat = Cat("Garfield", 35, "ginger and black")

  Print.print(cat)
}
```
</div>

**Print Syntax**

Let's make our printing library easier to use by defining some print syntax:

 1. Create an object called `PrintSyntax`.

 2. Inside `PrintSyntax`, define an `implicit class PrintOps[A]` to
    wrap up a value of type `A`. `PrintOps` should define the
    following methods:

     - `format` accepts an implicit `Printable[A]` and return
       a `String` representation of the wrapped value;

     - `print` accepts an implicit `Printable[A]` and return `Unit`.
       It should print a `String` representation of the wrapped value
       to the console.

 3. Update the `Main` object from the previous exercise to
    use `PrintSyntax`.

<div class="solution">
First we define an `implicit class` to "enrich" our target classes with extra methods. This is generally referred to as "type enrichment" in Scala. Similar features exist in other languages, for example "extension methods" in C# and "categories" in Objective C:

```tut:book
object PrintSyntax {
  implicit class PrintOps[A](value: A) {
    def format(implicit printable: Printable[A]): String = {
      printable.format(value)
    }

    def print(implicit printable: Printable[A]): Unit = {
      println(printable.format(value))
    }
  }
}
```

With `PrintOps` in scope, we can call the imaginary `print` and `format` methods on any value for which Scala can locate an implicit instance of `Printable`:

```tut:book
object Main extends App {
  import PrintSyntax._

  Cat("Garfield", 35, "ginger and black").print
}
```
</div>

### Take Home Points

In this section we revisited the concept of a **type class**, which allows us to add new functionality to existing types.

The Scala implementation of a type class has **three parts**:

 - the *type class* itself, a generic trait;
 - *instances* for each type we care about; and
 - one or more generic *interface* methods.

Interface methods can be defined in *interface objects* or *interface syntax*. Implicit classes are the most common way of implementing syntax.

In the next section we will take a first look at Cats. We will examine the standard code layout Cats uses to organize its type classes, and see how to choose type classes, instances, and syntax for use in our code.