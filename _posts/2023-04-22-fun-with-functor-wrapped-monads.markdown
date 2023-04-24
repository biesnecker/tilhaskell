---
layout: post
title: Fun with functor-wrapped monads
image: fun-with-functor-wrapped-monads.jpg
date: 2023-04-22 18:18:50 -0400
categories: monads, functors
---

## The problem

I recently encountered a scenario where I needed to compose a function that accepted a plain value and returned a monad `a -> m b` with another function that produced a value of the same monad wrapped within another monad `(Monad m, Monad n) => m (n a)`. To my surprise, I couldn't find a combinator for this purpose in the standard library.

Initially, I was looking for a function signature like this:

{% highlight haskell %}
(Monad m, Monad n) => (a -> n b) -> m (n a) -> m (n b)
{% endhighlight %}

In my specific case, I had a database-querying function that returned an `Either e a` wrapped inside a database monad, a function `a -> Either e b`, and I wanted to leverage the short-circuiting behavior of `Either`. So, more concretely, I was looking for a type signature like:

{% highlight haskell %}
(Monad m) => (a -> Either e b) -> m (Either e a) -> m (Either e b)
{% endhighlight %}

However, staying more generic allows us to create a more reusable solution.

## The solution

The first step is to recognize that you need to work inside the outer monad, but you don't have to do anything with it, implying that all we need on the outside is a `Functor`, not a `Monad`.

{% highlight haskell %}
(Monad m, Functor f) => (a -> m b) -> f (m a) -> f (m b)
{% endhighlight %}

Given that we can unwrap the functor with `fmap`, we are left with a function `a -> m b` and a monadic value `m a`. We need to end up with an `m b`, which `fmap` will then helpfully wrap back up in `f`. This is precisely what a monadic bind does!

{% highlight haskell %}
(>>=) :: forall a b. m a -> (a -> m b) -> m b
{% endhighlight %}

It took me some time to realize this, but all that remains is to partially apply `>>=` so that we have a function `m a -> m b`, and then partially apply `fmap` to obtain our final function:

{% highlight haskell %}
helper :: (Monad m, Functor f) => (a -> m b) -> f (m a) -> f (m b)
helper f = fmap (>>= f)
{% endhighlight %}

There's also the entirely point-free version, but I don't think it's as easy to read.

{% highlight haskell %}
helper :: (Monad m, Functor f) => (a -> m b) -> f (m a) -> f (m b)
helper = fmap . (=<<)
{% endhighlight %}

## Why use functor instead of monad?

I'll admit that the code I shipped had the original signature with two monads because that's all I needed at the time and I had other things to do, but using `Functor` for the outer structure is a better choice. This is because of power reduction, which means limiting the "power" of abstractions to the minimum required for a given task. Monads are more "powerful" than Functors--they can do more--which also means they're rarer. All monads are functors, but not all functors are monads.

An example is `Data.Map`, which is a functor but doesn't have a monad instance. Imagine you had a validation function `a -> Maybe b`, and a map `Map k (Maybe a)` and you want to apply the validation function to all of the entries that exist. The original two-monad signature version couldn't be used `helper` here, but the power-reduced version is actually the full solution.

{% highlight haskell %}
validateMap :: Ord k => (a -> Maybe b) -> Map k (Maybe a) -> Map k (Maybe b)
validateMap = helper
{% endhighlight %}
