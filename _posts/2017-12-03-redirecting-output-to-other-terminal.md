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

And before you ask -- yes, there's a caveat. Unfortunately (or fortunately for process security), once the process has started there's no way to change its file descriptors, unless you resort to some [gdb wizardry](https://www.redpill-linpro.com/sysadvent/2015/12/04/changing-a-process-file-descriptor-with-gdb.html) -- and even then it's *not always* possible. 

Wrapping up -- while it's not really a game-changer, it's still a feature that can sometimes be put to good use.
