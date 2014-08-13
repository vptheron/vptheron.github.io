---
layout: post
title: "Immutability is slow"
tags: 
- immutability
- functional programming
---

"Immutability is slow!" - sounds familiar? It sure does to me! This is the response I usually get after mentioning immutability as a core principle of functional programming.  Even though most people have no problem acknowledging that immutable data structures are easier to reason about and work with, such acknowledgment is always paired with performance concerns, and such concerns are used as an undisputable argument to dismiss immutability forever, end of discussion.

> Sure, immutability sounds nice, but I am working on real serious stuff here.  I have to do things with my data: add an item to my list, remove a key from my map, update the counter of the FizzBuzz, you know - grown up stuff! If all my structures are immutable I'll have to create a copy *each time* I want to change something! It's gonna be so slow and use so much memory! Nah, immutability sounds nice but it's really not efficient enough.

Now, just to get it out of the way: I am not going to pretend that immutability is *always* faster than mutable states, but claiming that it is *always* **slower** is just plain wrong.  Developers not used to immutable data may not realize how smart implementations of immutable data structures end up using less space and computation time, but I also believe that most of them vastly underestimate the actual cost of mutability.

In this post, I want to present some simple Java code to demonstrate how expensive it can be to write secure, bug free code relying on mutable states.

## Our use case

We are going to work with a very simple example: a class to model an itinerary composed of geo-points.

<div class="scrollable-code">
{% highlight java linenos=table %}
public class Itinerary {
    private List<Point> points;

    public Itinerary(List<Point> points){
        if(points.isEmpty(){
            throw new IllegalArgumentException("Need at least one point!");
        }
        this.points = points;
    }

    public List<Points> getPoints(){
        return points;
    }

    public void setPoints(List<Point> points){
        if(points.isEmpty(){
            throw new IllegalArgumentException("Need at least one point!");
        }
        this.points = points;
    }

}
{% endhighlight %}
</div>

This looks like a pretty reasonnable class right? We want our itinerary to always contain at least one point and throw an exception to protect the integrity of our instances. Great.

There is a bug in every single method/constructor of this class.

## Arguments are not to be trusted

Our constructor and our getter blindly trust the given parameters and simply create a private *reference* pointing to the *same* instance.
