---
layout: post
title: "A word of implicit conversions and value classes"
tags:
- scala
- implicit
- value classes
---

Implicit conversion is a very powerful feature provided by the Scala language. Its most common use case is to add capabilities to a class that is outside of the developer's control (e.g. a class provided by a dependency or a third party library). Let's look at a very basic example:

{% highlight scala %}
// This is the class provided by the third party library, no way to change this code.
class Person(val firstName: String, val lastName: String)

// A wrapper for Person providing the much needed fullName field
class RichPerson(person: Person){
  val fullName: String = person.firstName + " " + person.lastName
}

// The implicit conversion to go from a Person instance to a RichPerson
implicit def personToRichPerson(person: Person): RichPerson = new RichPerson(person)

// Assuming that the implicit conversion is in scope, you can now write:
val p = new Person("Bob", "Saget")
println(p.fullName)
{% endhighlight %}

Most of the work happens at compile time: the compiler realizes that `fullName` does not exist in `Person`, it then searches the scope for an implicit conversion from `Person` to *[something that has a fullName field defined]*. If no matching implicit conversion is available in scope you get a compilation error.

It turns out Scala 2.10 introduces a new syntax that makes it even easier to define implicit conversions. Behold **implicit classes**:

{% highlight scala %}
// Still the class provided by the third party library
class Person(val firstName: String, val lastName: String)

// A wrapper for Person providing the much needed fullName method
implicit class RichPerson(person: Person){
  val fullName: String = person.firstName + " " + person.lastName
}

val p = new Person("Bob", "Saget")
println(p.fullName)
{% endhighlight %}

Note that `personToRichPerson` is gone. By using the `implicit` keyword when defining our `RichPerson` wrapper we save ourselves the need to write the implicit conversion, the compiler will generate it for us.

Let's talk about the cost of implicit conversions. The compiler has to look up the appropriate conversion, this takes time, but this look-up is done at compile time so not much of an issue. However, at runtime, the program still needs to instantiate an instance of `RichPerson` *each time* you want to access the `fullName` field (or any other field defined in your rich wrapper):

{% highlight scala %}
// What really happens when you invoke fullName on Person
val p = new Person("Bob", "Saget")
println(new RichPerson(p).fullName)
{% endhighlight %}

For most applications, this is not a problem, and frankly it is probably not worth looking for a workaround right? What if this workaround was incredibly simple and required no efforts? Enter **value classes**:

{% highlight scala %}
// Still the class provided by the third party library
class Person(val firstName: String, val lastName: String)

// A wrapper for Person providing the much needed fullName method
implicit class RichPerson(val person: Person) extends AnyVal {
  def fullName: String = person.firstName + " " + person.lastName
}

val p = new Person("Bob", "Saget")
println(p.fullName)
{% endhighlight %}

OK, so a few things changed here. The most important one is that now `RichPerson` extends `AnyVal` which turns it into a *value class* - all the other changes are a consequence of this new status since value classes come with a very strict set of rules to follow, we will go through them in a minute. First, what does it mean that `RichPerson` is both an implicit class and a value class? Basically, there is no need to create a `RichPerson` instance when calling `fullName` on a `Person`! Here is how it works: the compiler silently creates a companion object for our value class `RichPerson` with a method `fullName(person:Person)`[(1)](#truth_about_method_name) and each time we call `fullName` on a `Person` instance the compiler re-routes this call to use the static method defined on the companion object and pass the `Person` instance as single argument.

Let's look at the other changes we made:

- `fullName` is now a `def` instead of a `val`, this is because value classes can only have `def` members (it makes sense since these members are converted into static methods on a companion object, `val` members would not work);
- we added the `val` keyword in front of `person` in the primary constructor, that's because the primary constructor of a value class can only take one parameter and it has to be a `val`.

There are many other rules to follow when writing a value class, for example:

- the class can only have one constructor, the primary constructor;
- the single `val` parameter of the primary constructor cannot have a value class for type.

I invite you to [read more about value classes and universal traits](http://docs.scala-lang.org/overviews/core/value-classes.html)

---

<a name="truth_about_method_name"></a>(1) Actually the name is slightly different, but it does not matter.
