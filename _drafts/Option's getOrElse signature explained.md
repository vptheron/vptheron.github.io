---
layout: post
title: "Option's getOrElse signature explained"
tags: 
- scala
- variance
---

The standard Scala library provides developer with this nifty little piece of code that is `Option[A]`. I will assume that everyone reading this is familiar with this abstract class that has two implementation: either `Some[A]` to wrap a value of type `A` or `None` which is basically an empty container.

`Option` is most commonly used either as a function's return type to capture the idea of "absence of result", or as a function parameter's type to indicate an optional argument. In both contexts, it is not uncommon to want to use a default value either if no result was found or no parameter was provided, and that's where `getOrElse` enters the stage:

<div class="scrollable-code">
{% highlight scala linenos=table %}
def newConnection(hostOption: Option[String], portOption: Option[Int]): Connection = {
  val host = hostOption.getOrElse("localhost")
  val port = portOption.getOrElse(80)
  ...
}
{% endhighlight %}
</div>

However useful `Option.getOrElse` appears, its signature can lead to very tricky errors:

<div class="scrollable-code">
{% highlight scala linenos=table %}
// What you would expect getOrElse to be defined as:
def getOrElse(default ⇒ A): A

// What the definition actually is:
def getOrElse[B >: A](default: ⇒ B): B 
{% endhighlight %}
</div>

Here is the sort of things that we can write with `getOrElse`:

<div class="scrollable-code">
{% highlight scala linenos=table %}
scala> val a = Some(42)
a: Some[Int] = Some(42)

scala> a.getOrElse(3L)
res0: AnyVal = 42

scala> val b = Some(Set(1,2,3))
b: Some[scala.collection.immutable.Set[Int]] = Some(Set(1, 2, 3))

scala> b.getOrElse(1 :: 2 :: Nil)
res1: scala.collection.immutable.Iterable[Int] with Int => AnyVal = Set(1, 2, 3)

scala> b.getOrElse(1L :: 2L :: Nil)
res2: scala.collection.immutable.Iterable[AnyVal] with Int => AnyVal = Set(1, 2, 3)
{% endhighlight %}
</div>

Two things to notice:
  * `getOrElse` does not complain when it is called on a `Some[Int]` with a default value of type `Long`, or on a `Some[Set[Int]]` with default value of type `List[Long]`.
  * the return type of `res0`, `res1` and `res2` is set to the first common parent of the type wrapped by `Some[A]` and the given default value.

In practice, it is very unlikely that this would lead to serious errors. For example, if we try to use `res2` as a `Set[Int]` the compiler will yet at us, we will realize our mistake and call `getOrElse` with a `Set[Int]` instead of a `List[Long]`. No big deal. But why is `getOrElse` so permissive? What is exactly the point of allowing a different type for the default value instead of simply using `A`?
