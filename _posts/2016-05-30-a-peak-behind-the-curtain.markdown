---
layout: post
title:  "Under the hood: Angular 1 one-time binding "
date:   2016-05-30 00:00:00 -0700
author: joel paulino
comments: true
categories: js
tags: angular javacript
---
One of the best ways to become a better developer is to look under the hood of your favorite frameworks.
Today we are on going to look at how one-time binding works in Angular templates. I assume that you already know how to create a
two way binding in Angular:
{% highlight html %}
{% raw %}{{person.first_name }}{% endraw %}
<!-- OR -->
<span ng-bing="person.first_name"></span>
{% endhighlight %}
<!--more-->

Two-way bindings create watchers that track changes on your models. This can be costly,
especially if you have models that won't change once stable (not `undefined`).
So, in Angular 1.3 angular introduced the bind one-time binding api:

{% highlight html %}
{% raw %}{{::person.first_name }}{% endraw %}
<!-- OR -->
<span ng-bing="::person.first_name"></span>
{% endhighlight %}

There the angular team created an algorithm to handle ontime binding. From the Angular Docs:

> 1. Given an expression that starts with ::, when a digest loop is entered and expression is dirty-checked, store the value as V
> 2. If V is not undefined, mark the result of the expression as stable and schedule a task to deregister the watch for this expression when we exit the digest loop
   Process the digest loop as normal
> 3. When digest loop is done and all the values have settled, process the queue of watch deregistration tasks. For each watch to be deregistered, check if it still evaluates to a value that is not undefined. If that's the case, deregister the watch. Otherwise, keep dirty-checking the watch in the future digest loops by following the same algorithm starting from step 1

So this lays out the algorithm behind one-time bindings, lets look at the juice parts of the implementation.

{% highlight javascript %}
// Angular 1.5.6 Source code
if (exp.charAt(0) === ':' && exp.charAt(1) === ':') {
  oneTime = true;
  exp = exp.substring(2);
}
{% endhighlight %}

This snippet comes from the `$parse` service. This service takes angular expressions (string) and converts them into functions
which get called against the current context (scope). We see above that when binding an expression which begins with  `::`, $parse
sets `oneTime` to true and continues parsing the expression with out `::`. Next we see how angular setups watcher temporary watchers
for our ontime expressions:

{% highlight javascript %}
// Angular 1.5.6 Source code
if (parsedExpression.constant) {
  parsedExpression.$$watchDelegate = constantWatchDelegate;
} else if (oneTime) {
  parsedExpression.$$watchDelegate = parsedExpression.literal ?
      oneTimeLiteralWatchDelegate : oneTimeWatchDelegate;
} else if (parsedExpression.inputs) {
  parsedExpression.$$watchDelegate = inputsWatchDelegate;
}
{% endhighlight %}

The `oneTimeLiteralWatchDelegate` and `oneTimeWatchDelegate` essentially create our watchers. The following example shows
the different between these two functions.


{% highlight html %}
<!-- HTML -->
<!-- oneTimeWatchDelegate-->
{% raw %}{{::item1}}{% endraw %}
<!-- oneTimeLiteralWatchDelegate -->
{% raw %}{{::[item1, item2]}}{% endraw %}
{% endhighlight %}

{% highlight javascript %}
// In controller
$scope.item1 = 'test1';
$timeout(function(){
    $scope.item2 = 'test-2';
}, 3000);
{% endhighlight %}

Here `oneTimeWatchDelegate` deletages a watcher that waits until `item1` is defined. Once it's defined, the watcher is destroyed.
Because the `item1` is assigned a value right away, the string `'test-2'` gets rendered in our template. `oneTimeLiteralWatchDelegate`
I handles literal expression in our templates (Array and Object literals). It sets up a watcher and waits to all items are defined.
So in the exmaple above, ['test-1', 'test-2'] gets rendered after 3 seconds because `item2` is `undefined` until then.