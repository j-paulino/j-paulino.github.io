---
layout: post
title:  "Hello console!"
date:   2016-05-30 00:00:00 -0700
author: joel paulino
comments: true
categories: js
---
Although the console object should be in every Front End Developer's toolbelt, it's actually a non-standard
feature. But in most browsers the API is the same, so let's checkout some of my favorite methods.

#### Console.log
I am sure this snippet looks very familiar.
{% highlight javascript %}
// Ex 1
console.log("Hello console!");
{% endhighlight %}
<!--more-->

How about this?
{% highlight javascript %}
// Ex 2
console.log("Hello", "console", "!");
{% endhighlight %}

The console.log method takes any number of arguments and essentially prints to the console the string representations of each
of argument appended together in ordered list separated by a space.

{% highlight javascript %}
// Output of Ex 1
> Hello console!
// Output of Ex 2
> Hello console ! // Notice the space between console and !
{% endhighlight %}

You can pass it any of the JS primitives, other native objects and even your own custom objects.

{% highlight javascript %}
// Ex 3
> console.log(new Date(), 2, String("hello"), [1,3], new Person("Joel"))
{% endhighlight %}

<figure class="highlight">
<pre class="output">> Sun May 29 2016 22:55:10 GMT-0700 (PDT) 2 "hello" [1, 3] Person {name: "Joel"}</pre>
</figure>

The log method also has another API.

{% highlight javascript %}
// Ex 4
> console.log("Hello, %s! You're %d years old.", "Joel", 28)
{% endhighlight %}

<figure class="highlight">
<pre class="output">> Hello, Joel! You're 28 years old.</pre>
</figure>

Here we pass <code>console.log</code> a message and substrings which will replace substitution strings in the main string.

#### Console.dir
The `dir` is the cousin of the `log` method. To see the usefulness of dir, type the following into your console:

{% highlight javascript %}
// Ex 5
> console.log(window.document)
{% endhighlight %}

<figure class="highlight">
<pre class="output">
> #document
    &lt;DOCTYPE html&gt;
    &lt;html&gt;
        &lt;head&gt;...&lt;/head&gt;
        &lt;body&gt;....&lt;/body&gt;
    &lt;/html&gt;
</pre>
</figure>

Not much different than inspecting the page's source. The document is special and the console gives it special treatment.
It outputs the DOM Tree in html format. Now try the following:

{% highlight javascript %}
// Ex 6
> console.dir(window.document)
{% endhighlight %}

<figure class="highlight">
<pre class="output">
> #document
    URL: "http://localhost:4000/articles/2016/05/01/hello-console/"
    activeElement: body.site.mdl-color--grey-200
    alinkColor:""
    all: HTMLAllCollection[206]
    anchors: HTMLCollection[0]
    applets: HTMLCollection[0]
    baseURI:c"http://localhost:4000/articles/2016/05/01/hello-console/"
    bgColor: ""
    ...
</pre>
</figure>

Now you can see the document object's properties and methods.

#### Console.info, warn and error

These methods work exactly like `log` except that output UI and formatting is slightly different.

{% highlight javascript %}
// Ex 7
> console.info('Test')
> console.warn('Test')
> console.error('Test')
{% endhighlight %}

![alt text][ex-7-out]

As you can see, color and iconography are used to classify the outputs. If your using the console API to provide information to user,
it's nice to have some control over how your messages and errors get displayed.


### Console.table

The `table` method is a fun one. It lets you display tabular data in a...table. The example below was lifted from the console
output of the [Performance-Analyser](https://chrome.google.com/webstore/detail/performance-analyser/djgfmlohefpomchfabngccpbaflcahjf?hl=en target="_blank")
chrome extension (highly recommend you check it out).

{% highlight javascript %}
// Ex 8,
console.table(table.data, table.columns);
{% endhighlight %}

![alt text][ex-8-out]

I covered some goodies, but for a more comprehensive tutorial visit [MDN](https://developer.mozilla.org/en-US/docs/Web/API/console target="_blank").
I will probably also have some follow ups to this post.


[ex-7-out]: /img/post/2016/05/30/ex-7-out.png "Example 7 output"
[ex-8-out]: /img/post/2016/05/30/ex-8-out.png "Example 8 output"

