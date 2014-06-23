---
layout: post
title: "Error Handling in Scala"
tags: 
- scala
- errors
- either
- try
---

Managing errors and failures is hard, often harder than managing success - I guess no one likes to think about all the different ways their pretty code may fail and burn in flames. 

When I used C I learnt that returning an integer other than 0 was a good way to indicate failure, when I moved to Java I learnt that throwing an exception was the best practice; now I mostly work with Scala, and while the language supports exceptions their use is strongly discouraged. Instead, it is more common to use "wrapping objects", specific containers designed to handle errors. Avoiding exceptions is definitely a plus, however it is not always easy to decide which container to use. Should one use `Option` or `Either`? `Try` or `Future`? What about `\/` and `Validation`? Are all failures even created equal? Should they all be managed in the same way? 

The purpose of this post is to describe the strategy I have been using for quite some time with good results. I also want to take the time to go over all the steps that led me to this design and explain why I discarded other options. 

## The background for our noble quest

Every hero needs an intriguing backstory, here is ours: we are working on a web application implementing a REST API, one of our endpoints is used to create new user accounts.

<div class="scrollable-code">
{% highlight scala linenos=table %}
class UserController(userService: UserService) {

  def createUserAccount(email: String, password: String): HttpResult = {
    val domainResult = userService.createUserAccount(email, password) 
    toHttpResult(domainResult)
  }
}
{% endhighlight %}
</div>

That's quite a leazy controller: it just calls some service implementing our domain logic to create a new user account, receives *some* result, turns it into an instance of `HttpResult` (a type provided by our imaginary web framework) and returns it to the API user. The only missing part is the type of `domainResult`, the result of a call to `UserService.createUserAccount`.

<div class="scrollable-code">
{% highlight scala linenos=table %}
trait UserService {
  def createUserAccount(email: String, password: String): ??? // see how there is no type yet?
}
{% endhighlight %}
</div>

What is the best type to return from our domain object? Let's begin our journey toward the ideal return type...

## In a brave new world...

<div class="scrollable-code">
{% highlight scala linenos=table %}
trait UserService {
  def createUserAccount(email: String, password: String): UserAccount
}
{% endhighlight %}
</div>

Boom! Done. Next. 

OK, this one was just a jest. We all know this `UserService` only exists in a perfect imaginary world where nothing ever goes wrong, everyone is happy, the Kardashians are not on TV and children ride unicorns to school. Moving on.

## Do. Or do not. There is no try.

Well, actually there is: 

<div class="scrollable-code">
{% highlight scala linenos=table %}
trait UserService {
  def createUserAccount(email: String, password: String): Try[UserAccount]
}

class UserController(userService: UserService) {

  def createUserAccount(email: String, password: String): HttpResult = 
    userService.createUserAccount(email, password) match {
      case Success(newUserAccount) => // 200 - OK
      case Failure(exception) => // 500 - ERROR
    }
}
{% endhighlight %}
</div>

By returning a `Try[UserAccount]` our `UserService` states that it will *try* to return a `UserAccount` but something may go wrong and every calling object should be ready for it. The `UserController` is a diligent caller and it properly handles both cases. Not too bad, let's make it slightly better.

## Embrace the trend

Asynchronous, non-blocking, reactive. These are the new buzz words every cool kid has on his lips nowadays. Our `UserService` should surf this new wave of coolness and return an instance of `Future`!

<div class="scrollable-code">
{% highlight scala linenos=table %}
trait UserService {
  def createUserAccount(email: String, password: String): Future[UserAccount]
}

class UserController(userService: UserService) {

  def createUserAccount(email: String, password: String): Future[HttpResult] = 
    userService.createUserAccount(email, password) map {
      case newUserAccount => // 200 - OK
    } recover {
      case exception => // 500 - ERROR
    }
}
{% endhighlight %}
</div>

For our current purpose of managing failure, a `Future` works pretty much like a `Try`: it either succeeds and wraps the desired result (here a `UserAccount`) or fails and contains an exception. The difference is that the `Future` will complete at some point in the *future* as opposed to the `Try` we previously used. Notice that I cheated a little bit and my controller is now returning a `Future[HttpResult]`, I assume my imaginary web framework knows how to handle those and save myself a (disgraceful) call to `Await.result`.

Even if our `UserService` is not truly asynchronous it is probably a good idea to stick to `Future`, at least it gives us the opportunity to make it asynchronous later on without changing the calling controller.

## When in doubt blame the user

It seems a bit unfair to always return a code 500 when our `UserService` fails to create a new account. After all, it may be failing due to bad parameters given by the API user, we should blame him! We could throw a `UserCreationException` containing an explanatory message (`"Invalid email."`, `"Email already exists."`, etc) but it's not very convenient to handle in our `UserController` if we wish to know what went wrong exactly. A better way is to return a different type of exception for each error:

<div class="scrollable-code">
{% highlight scala linenos=table %}
trait UserService {
  def createUserAccount(email: String, password: String): Future[UserAccount]
}

object UserService {
  case object EmailAlreadyExistsException extends Exception
  case object InvalidPasswordException extends Exception
}

class UserController(userService: UserService) {

  def createUserAccount(email: String, password: String): Future[HttpResult] = 
    userService.createUserAccount(email, password) map {
      case newUserAcccount => // 200 - OK
    } recover {
      case EmailAlreadyExistsException => // return 409 - CONFLICT
      case InvalidPasswordException => // return 400 - BAD REQUEST
      case exception => // 500 - INTERNAL ERROR
    }
    
}
{% endhighlight %}
</div>

Now *you* deal with your buggy code, API user!

But wait, what if we add a new error? Let's say we decide to use a regular expression to validate that the email given to our `UserService` is an actual email address. We create a new exception `InvalidEmailAddress` in the `UserService` object but nothing forces us to pattern match against it in the controller! Damn, we are back to returning a 500 and taking the blame for the user's mistake! That can't be!

## The end of our journey

OK, time to take a step back and try something different. First, we can divide potential errors in two categories:

* **domain/business errors**: these are **expected** failures, they are part of the normal flow of our application and are almost exclusively due to invalid input violating business rules (invalid values, duplicates, permission denied, etc). We want to be able to give a useful report to the function caller so that it can understand what went wrong and why its request cannot be fulfilled (and potentially escalate to its own caller and etc).

* **internal/technical errors**: these should not happen, but they will. The database will crash, our call to a web service will time out, someone will plug the wrong wire somewhere and kill the network, etc. In this case, we do not want to give too much information about what went wrong - a caller using a `UserService` should not have to handle a `DatabaseDownException` just because the service is implemented by a `MySqlUserService`. We want to avoid leaking the underlying implementation details.

Having established these two different kinds of errors, we can design a return type to usefully report them. Instead of just using a `Future`, we will use an instance of `Either` *wrapped in a `Future`*.

<div class="scrollable-code">
{% highlight scala linenos=table %}
trait UserService {
  def createUserAccount(email: String, password: String): Future[Either[UserCreationError, UserAccount]]
}

object UserService {
  sealed trait UserCreationError
  case object EmailAlreadyExistsException extends UserCreationError
  case object InvalidPasswordException extends UserCreationError
}

class UserController(userService: UserService) {

  def createUserAccount(email: String, password: String): Future[HttpResult] = 
    userService.createUserAccount(email, password) map {
      case Right(newUserAccount) => // return 200
      case Left(EmailAlreadyExistsException) => // return 409 - CONFLICT
      case Left(InvalidPasswordException) => // return 400 - BAD REQUEST
    } recover {
      case exception => // 500 - INTERNAL ERROR
    }
}
{% endhighlight %}
</div>

Here is what happens:

* domain errors are reported by a **successfull `Future` containing an instance of `Left` containing an object extending a base sealed trait**. Each possible business error is described by a case object (or a case class if we want to return some details about the failure). Our error objects extend a sealed trait so callers can confidently pattern match: if we add a new error the compiler will yell at them and complain that their pattern matching is not exhaustive.

* technical errors are reported by an **exception in a failed `Future`**. Ideally, the application should take action to fix the problem (e.g. try to reconnect to the DB, clear the HTTP connection pool, page the poor lad who is on call that night, etc), but it is not the responsibility of the caller, we just let it know that our attempt miserably failed.

***

The only major drawback I've experienced with this design is its extrem verbosity. You now have two levels of containers to deal with - the `Future` and the `Either` - which means more mapping and pattern matching to write. Plus, creating a new case object/class for each possible error can get extremely annoying, but I guess you can see it as a good motivation to break your service down into several objects addressing different concerns.

## Last words: Scalaz for extra power

If you'd rather stick to the official Scala API, `Future[Either[ErrorBaseTrait, Result]]` is a good option. If you can include **Scalaz** in your dependencies I would suggest replacing `Either` with `\/` (good luck googling that one!). `Either` uses left and right projections to handle its content, while `\/` is right-bias (the "happy path"), which is usually what you want and allows you to use it in for-comprehensions. `Validation` is another candidate if you want to *accumulate* several failures and return them.
