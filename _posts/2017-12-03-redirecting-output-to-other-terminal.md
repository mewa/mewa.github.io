---
layout: post
title: Redirecting output to other terminal
categories: [Shell]
comments: true
---

There are many reasons one might want to see output from shell commands in another terminal emulator but it definitely has its uses. The other day it just so happended that I needed such a functionality. Without going into details I'm going to show you how to achieve such behaviour -- and more -- easily, by leveraging the fact that under the hood std(in\|out\|err) are just *nix file descriptors. 
<!--more-->

### Hey, it's a one-liner!

All that's needed is the snippet below
{% highlight sh %}
exec 1>/proc/<PID>/fd/1
{% endhighlight %}

Where `<PID>` is id of the process you want to redirect your output to (and you can obtain it for instance by running `echo $$` inside the target terminal).

But don't take my word for it -- let's see for ourselves how it works. 

### Exec me

I think the biggest revelation comes with acknowledging the fact that `exec` is used for something more than to (according to `man`):

~~~
replace the shell with command without creating a new process
~~~

because this functionality isn't even the first thing mentioned in `man`. The first thing `man` has to say about `exec` is that

~~~
The exec utility shall open, close, and/or copy file descriptors as specified by any redirections as part of the command.
~~~

Wait, that's great! It's actually *much* more powerful that the trivial case I used as an excuse to write this article. Really, the opportunities are countless and they enable you to operate on a whole new level. 

You could for instance concatenate outputs from multiple commands and collect a composite log. You could plug in a pipe to a **running** process to start logging stuff **without ever restarting it**. You could enable (and debug) output redirected to `/dev/null` and disable it again once you're done. And all you're ever going to need is the `exec` command.

The easiest solutions often come unnoticed. I hope this short article will be of use for at least some of you!
