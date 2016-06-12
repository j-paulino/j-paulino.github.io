---
layout: post
title:  "Under the hood: Angular 1 one-time binding "
date:   2016-06-11 00:00:00 -0700
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

There the angular team created an algorithm to handle ontime binding. From the
[Angular Docs](https://docs.angularjs.org/guide/expression#value-stabilization-algorithm){:target="_blank"}:

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
sets `oneTime` to true and continues parsing the expression without `::`. Next we see how angular setups watcher temporary watchers
for our ontime expressions:

{% highlight javascript linenos %}
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

In line 6 we see `oneTimeLiteralWatchDelegate` and `oneTimeWatchDelegate`. These functions essentially create our watchers.
The following example shows the different between these two functions.

{% highlight html %}
<!-- HTML -->
<!-- Ex 1 oneTimeWatchDelegate for scoped variable-->
{% raw %}{{::item1}}{% endraw %}
<!-- Ex 2 oneTimeLiteralWatchDelegate for literal expressions  -->
{% raw %}{{::[item1, item2]}}{% endraw %}
<!-- Ex 3 oneTimeWatchDelegate for scoped literal -->
{{::$scope.items}}
{% endhighlight %}

{% highlight javascript %}
// In controller
$scope.item1 = 'test1';
$timeout(function(){
    $scope.item2 = 'test-2';
}, 3000);
$scope.items = [$scope.item1, $scope.item2];
{% endhighlight %}

Here `oneTimeWatchDelegate` deletages a watcher that waits until `item1` is defined. Once it's defined, the watcher is destroyed.
Because the `item1` is assigned a value right away, the string `'test-2'` gets rendered in our template. `oneTimeLiteralWatchDelegate`
It handles literal expression in our templates (Array and Object literals). It sets up a watcher and waits to all items are defined.
So in the exmaple above, ['test-1', 'test-2'] gets rendered after 3 seconds because `item2` is `undefined` until then.

{% highlight html %}
<!-- What gets rendered  -->
<!-- Ex 1 oneTimeWatchDelegate for scoped variable-->
test-1
<!-- Ex 2 oneTimeLiteralWatchDelegate for literal expressions  -->
[test-1, test-1]
<!-- Ex 3 oneTimeWatchDelegate for scoped literal -->
[test-1, null]
{% endhighlight %}

If you onetime bind
an expression with that is a Object or Array literal, then all the element need to be defined for the binding to be considered stable: all elements are
defined.

To hammer things home lets look at the these functions.
{% highlight javascript linenos %}
function oneTimeWatchDelegate(scope, listener, objectEquality, parsedExpression) {
    var unwatch, lastValue;
    return unwatch = scope.$watch(function oneTimeWatch(scope) {
        return parsedExpression(scope);
    }, function oneTimeListener(value, old, scope) {
        lastValue = value;
        if (isFunction(listener)) {
            listener.apply(this, arguments);
        }
        if (isDefined(value)) {
            scope.$$postDigest(function () {
                if (isDefined(lastValue)) {
                    unwatch();
                }
            });
        }
    }, objectEquality);
}
{% endhighlight %}

This function is a bit intimidating, but where  magic happens (between lines 11 and 15) it's pretty easy to follow. It checks the value of
of our expression after the end of the digest loop. If the value is defined, then unwatch is excecuted and our watch is destroyed.
If the value is `undefined` then we don't even reach line 11. Line 10 insures that the watch stays. This process continues until our expression's
value is defined.

{% highlight javascript linenos %}

function oneTimeLiteralWatchDelegate(scope, listener, objectEquality, parsedExpression) {
    var unwatch, lastValue;
    return unwatch = scope.$watch(function oneTimeWatch(scope) {
        return parsedExpression(scope);
    }, function oneTimeListener(value, old, scope) {
        lastValue = value;
        if (isFunction(listener)) {
            listener.call(this, value, old, scope);
        }
        if (isAllDefined(value)) {
            scope.$$postDigest(function () {
                if (isAllDefined(lastValue)) unwatch();
            });
        }
    }, objectEquality);

    ...
}
{% endhighlight %}

`oneTimeLiteralWatchDelegate` is very similar to `oneTimeWatchDelegate`. The main difference is the use of `isAllDefined` vs
`isDefined`.  `isAllDefined` iterates througn of the values of the expression and checks that they are defined.
Remember oneTimeLiteralWatchDelegate handles expression that invlove literals (Array and Object). So oneTimeLiteralWatchDelegate
waits until all the values in our expression our defined before killing our watch.