---
layout: post
title:  "Under the hood: Angular 1 one-time binding "
date:   2016-06-13 00:00:00 -0700
author: joel paulino
comments: true
categories: js
tags: Angular Javacript
---
One of the best ways to become a better developer is to look under the hood of your favorite frameworks.
Today we are going to look at how one-time binding works with Angular expressions. I assume that you already know how to create a
two way binding in Angular:
{% highlight html %}
{% raw %}{{person.first_name }}{% endraw %}
<!-- OR -->
<span ng-bing="person.first_name"></span>
...
{% endhighlight %}
<!--more-->

Two-way bindings create watchers that track changes on your models. These watchers potentially live for the life span of
the session in which they were created. This can be costly, especially if you have models that won't change once stable.
So, in Angular 1.3 Angular introduced the bind one-time binding api:

{% highlight html %}
{% raw %}{{::person.first_name }}{% endraw %}
<!-- OR -->
<span ng-bing="::person.first_name"></span>
...
{% endhighlight %}

If you use one-time binding, you create watchers that get automatically
destroyed once your expression does not evaluate to  `undefined` (becomes stable).
One-time binding is outlined in the following (form the
[Angular Docs](https://docs.Angularjs.org/guide/expression#value-stabilization-algorithm){:target="_blank"}):

> 1. Given an expression that starts with `::`, when a digest loop is entered and expression is dirty-checked, store the value as V
> 2. If V is not undefined, mark the result of the expression as stable and schedule a task to deregister the watch for this expression when we exit the digest loop
   Process the digest loop as normal
> 3. When digest loop is done and all the values have settled, process the queue of watch deregistration tasks. For each watch to be deregistered, check if it still evaluates to a value that is not undefined. If that's the case, deregister the watch. Otherwise, keep dirty-checking the watch in the future digest loops by following the same algorithm starting from step 1

Know that we know the algorithm behind one-time bindings, lets look at the juicy parts of the implementation.

{% highlight javascript %}
// Angular 1.5.6 Source code
if (exp.charAt(0) === ':' && exp.charAt(1) === ':') {
  oneTime = true;
  exp = exp.substring(2);
}
{% endhighlight %}

This snippet comes from the `$parse` service. This service takes Angular expressions (Strings) and converts them into functions
which get called against the current context (scope). These functions make sure that what is rendered in our templates matches the values that exist in our controllers (and link functions).
We see above that when parsing an expression which begins with  `::`, $parse
sets `oneTime` to true and continues parsing the expression without `::`. Next we see how Angular sets up temporary watchers
for our one-time expressions:

{% highlight javascript linenos %}
// Angular 1.5.6 - $parse fn
if (parsedExpression.constant) {
  parsedExpression.$$watchDelegate = constantWatchDelegate;
} else if (oneTime) {
  parsedExpression.$$watchDelegate = parsedExpression.literal ?
      oneTimeLiteralWatchDelegate : oneTimeWatchDelegate;
} else if (parsedExpression.inputs) {
  parsedExpression.$$watchDelegate = inputsWatchDelegate;
}
{% endhighlight %}

In line 5 we see `parsedExpression.literal`. What is `parsedExpression` anyway?

## Angular Expressions

Angular let's you write Javascript expressions in your HTML. But not all JS is legal in Angular Templates.
From the [Angular Docs](https://docs.Angularjs.org/guide/expression#Angular-expressions-vs-javascript-expressions){:target="_blank"}:

> - **Context:** JavaScript expressions are evaluated against the global window. In Angular, expressions are evaluated against a scope object.
> - **Forgiving:** In JavaScript, trying to evaluate undefined properties generates ReferenceError or TypeError.
 In Angular, expression evaluation is forgiving to undefined and null.
> - **Filters:** You can use filters within expressions to format data before displaying it.
> - **No Control Flow Statements:** You cannot use the following in an Angular expression: conditionals, loops, or exceptions.
> - **No Function Declarations:** You cannot declare functions in an Angular expression, even inside ng-init directive.
> - **No RegExp Creation With Literal Notation:** You cannot create regular expressions in an Angular expression.
> - **No Object Creation With New Operator:** You cannot use new operator in an Angular expression.
> - **No Bitwise, Comma, And Void Operators:** You cannot use Bitwise, , or void operators in an Angular expression.

Angular subsets Javascript inside HTML through an Abstract Syntax Tree (AST).
The AST class can compile and interpret things like logical expression `foo === bar`, assignments `foo  = 'bar'`,
literals `[foo, bar]`, etc.
`parsedExpression` is an instance of the AST class. The literal property tells us if our expression
is an Array `[...]`, Object literal `{..}` or empty (`''`, `'  '`, etc) expression. The value of the `parsedExpression.literal` tells us
If our watch will be an instance of `oneTimeWatchDelegate` or `oneTimeLiteralWatchDelegate`.

## oneTimeWatchDelegate

Let's take a look at `oneTimeWatchDelegate`:

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
value is defined. Let's watch `oneTimeWatchDelegate` in action.

{% highlight html %}
<!-- HTML -->
<!-- Ex 1 oneTimeWatchDelegate -->
{% raw %}<span ng-bind="::item1"></span>{% endraw %}
{% endhighlight %}


{% highlight javascript %}
// Controller
(function(){
  'use strict';
  Angular.module('one-time', [])
    .controller('MainCtrl', MainCtrl);

  function MainCtrl($scope, $rootScope,$timeout){
    $timeout(function(){
          $scope.item1 = 'Test-1';
    }, 1000);
  }
})();
{% endhighlight %}

Rerun the Pen below to see this example in action.

<p data-height="265" data-theme-id="1" data-slug-hash="MeyjEP" data-default-tab="result" data-user="j-paulino" data-embed-version="2" class="codepen">See the Pen <a href="http://codepen.io/j-paulino/pen/MeyjEP/">One time binding Ex1</a> by Joel Paulino (<a href="http://codepen.io/j-paulino">@j-paulino</a>) on <a href="http://codepen.io">CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

Here `oneTimeWatchDelegate` deletages a watcher that waits until `item1` is defined.
`item1` is not initialized until a second after our app bootstraps. After that second, our expression is no longer `undefined`
and our watcher is destroyed.

## oneTimeLiteralWatchDelegate

`oneTimeLiteralWatchDelegate` handles literal expression (Array and Object literals) in our templates.
If you one-time bind an expression that is an Object or Array literal, then all the elements
must to be defined for the binding to be considered stable. What exactly do I mean by literal expression?
Let's look at an example.

{% highlight html %}
<!-- Ex 2 -->
<span ng-bind="::[item1]"></span>
{% endhighlight %}

Using the same controller as before, we modified our template by enclosing `item1` within square brackets. If this looks like Array syntax, it's because that's
is exactly what it is. And again our one-time binding seems to be working (rerun the Pen below).

<p data-height="265" data-theme-id="0" data-slug-hash="LZNxVd" data-default-tab="result" data-user="j-paulino" data-embed-version="2" class="codepen">See the Pen <a href="http://codepen.io/j-paulino/pen/LZNxVd/">One time binding Ex2</a> by Joel Paulino (<a href="http://codepen.io/j-paulino">@j-paulino</a>) on <a href="http://codepen.io">CodePen</a>.</p>
<script async src="//assets.codepen.io/assets/embed/ei.js"></script>

Well the bad news is that things aren't working as expected. The correct output is showing up in our HTML, but for the wrong reason.
If you open your console, you will see an error related to the Pen above. Essentially, we've created an unstable digest loop. Angular has a safety
mechanism built into the `$digest` function that kills a runaway digest loops after 10 iterations (you can change this I believe).

The failure stems from the following comparison, `['Test-1'] === ['Test-1']` (run this in your console). While dirty checking our expression,
this comparison is evaluated and always evaluates to `false`. This makes Angular think that our expression's value is always changing,
hence the digest loop error.

Full disclosure, we are dealing with the undocumeneted internals of Angular here. The [Angular Docs](https://docs.Angularjs.org/guide/expression#how-to-benefit-from-one-time-binding){:target="_blank"}
are a little light on parsing literal expressions. The Angular unit and integration tests on the other hand, proved to be a great resource.
Check out the tests for the [$parse service](https://github.com/Angular/Angular.js/blob/master/test/ng/parseSpec.js#L3256){:target="_blank"}.
With test like:

{% highlight javascript %}
var fn = $parse('::[foo,bar]');
...
$rootScope.foo = 'foo';
$rootScope.$digest();
expect($rootScope.$$watchers.length).toBe(1);
expect(log.empty()).toEqual([['foo', undefined]]);
{% endhighlight %}

You can see that our tests are on the right track. But back to our analysis.

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
`isDefined`.  `isAllDefined` iterates through all of the values of the expression and checks if they are defined. Once all the values are defined (not just truthy, `null` and `''` also work)
 the watch is destroyed. We ran into issues when trying to verify this, but not all is lost.

Swapping `objectEquality` in line 15 for `true` in Angular's source code gets rid of our error. This is because when objectEquality is true, instead running the comparison
`['Test-1'] === ['Test-1']` Angular runs `angular.equals(['Test-1'], ['Test-1'])` when dirty checking our array. For arrays `angular.equals` checks that `array1.length === array2.length` and that corresponding elements
in the arrays are equivalent. For objects it checks that properties match and are equal. When manually creating watchers, we can easily set the value of
`objectEquality`. But from what I can tell, there is no way to set it while Angular is creating it internal watchers when evaluating expression in HTML.


It was fun diving into the inner workings of Angular. I must admit, I need to do some further expirimentation and research on this topic. I will update this
entry accordingly based on my finding. But my next "Under the hood" will probably deal with Angular's AST implementation