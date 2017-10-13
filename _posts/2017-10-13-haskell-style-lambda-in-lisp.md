---
layout: post
title: Haskell-style lambdas in Lisp
categories: [Common Lisp, Haskell]
comments: true
---

Recently I was toying around with Lisp a bit and thought I'd share some insights. 

As you may or may not know programming in Lisps is somewhat different from the average programming languages you're used to. In what way, you might ask -- and no, I *don't* mean being swarmed up with parenthesis (besides syntax should be the least concern when picking *The Right Tool*). Here's why.

<!--more-->
It's about its extensibility. In most languages you're, more or less, forced to use the syntax and constructs which are offered by the particular language & compiler combo. Lisp, on the other hand, will be evolving alongside your project[^1]. To prove a point I'm going to show you just a simple gain that could get multiplied by thousands of lines of code.

In case you've never seen any Lisp code, this is what it looks like:

{% highlight common-lisp %}
(defun do-math (list)
  (let ((just-do-it #'(lambda (a) (+ a 3))))
    (mapcar just-do-it list)))
{% endhighlight %}

### &#955;

One of the nicer things different languages have to offer are the lambda expressions. Lisp, being a nice language, has them as well -- in the form of `(lambda (args) (body))`.
It doesn't take much to conclude that it's actually quite verbose -- compare it for example to the Haskell lambdas
{% highlight Haskell %}
\arg1 arg2 -> body
{% endhighlight %}

However, the great thing about Lisp is that with just a little bit of twiddling we can change that syntax into something else. Case in point -- we're going to add a Haskell-ish syntax for lambdas. 

To accomplish this we're going to use Lisp's macro system which is different from what you've probably been thinking about macros (for instance the impaired C macros). In Lisp macros are program-generating functions that are run at compile-time. Don't just believe me, though -- see for yourself in practice.

### The Code

For the record I'm using the SBCL implementation of Common Lisp, which is one of the most used Lisp dialects.

Just to clarify things, we'll be replacing `(lambda (args) (body))` with something along the lines of `(\args -> body)`.

First thing to note is that `->` symbol is the boundary between lambda's arguments and its body. Since we'll need to be able to refer to it in our macro, we're going to define it.

{% highlight common-lisp %}
(defconstant -> nil)
{% endhighlight %}

Let's get down to work on our little macro -- which we'll by the way call `\\` because `\` alone starts (no way!) an escape sequence. 

First, we're going to set the `body` variable to contain the whole body passed into the macro, because we don't know yet what part of it is arguments and what is the actual body. Then we're assigning `args` a list containing a value, which we'll later on discard, so we can append the actual arguments to it. While iterating through the macro arguments we're gradually appending consecutive elements to `args` and at the same time removing them from the `body`. When we meet `->` we simply discard it -- hence it could be just about any symbol, it's not evaluated anyway -- and proceed to code generation. 

Inside the back-ticked blocks we can use `,` and `,@` to expand lists of expressions in their place. The difference is trivial -- `,` expands a variable into a list under that name, whereas `,@` splices the contents of the list pointed by that variable in its place. We could actually forget about `,@` altogether but adding such the clause using it allows us to define constant lambdas (since we cannot -- in the general case -- evaluate to a list holding just a value but rather to the singleton element of that list).

{% highlight common-lisp %}
(defmacro \\ (&body lambda-body)
  (let* ((body lambda-body)
	 (args '(t))
	 (body (dolist (elem lambda-body)
		 (progn
		   (setf body (cdr body))
		   (when (eq '-> elem) (return body))
		   (setf args (append args (list elem))))))
	 (args (cdr args)))
    (if (eql 1 (length body))
	`(lambda ,args ,@body) ;; constant functions
	`(lambda ,args ,body))))
{% endhighlight %}

Now we can compress the lambda in our first example -- not a huge difference, but again -- it does add up.

{% highlight common-lisp %}
(defun do-math (list)
  (let ((just-do-it (\\ a -> + a 3)))
    (mapcar just-do-it list)))
{% endhighlight %}

If we replaced `\\` -- for instance with `$` -- that's another character we're saving in all the call sites. 

Anyway, I hope this small example was enough to demonstrate the whole point behind Lisp's macros -- to create *abstractions* and then write *code that uses them* (possibly even higher-level abstractions). 

###### The small print
[^1]: This statement is actually taken from Paul Graham's [*On Lisp*](http://www.paulgraham.com/onlisp.html) -- which is a great read, by the way
