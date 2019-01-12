---
title: "Introducting authmagic.io - a passwordless authentication service"
date: 2019-01-12T11:19:09+01:00
categories:
- announcement
---

Lately I have decided to take some time to work on my side-projects
and make them useful for the broader public.

Inevitably, I bumped into a problem. Actually, not really a problem,
but I had to implement a simple authentication scheme that at the same
time wouldn't be an overkill.

I decided to use passwordless login and registration links because it
seemed to fit my app just right and I wanted to have as little
overhead as possible.

As I was writing the authentication logic it suddenly struck me that
what I was doing could be made a piece on its own. A lightweight
solution for people who don't want to -- or rather don't *need* to --
invest into full-fledged user and identity management solutions.

So I extracted the relevant code and that's how
[Authmagic.io](https://authmagic.io) was born. Now, what is it
exactly?

Like I have already mentioned, it's a **passwordless authentication**
scheme. With [Authmagic.io](https://authmagic.io) you can send a
message -- together with a payload -- directly to your user and when
he returns to your app with a token issued by
[Authmagic](https://authmagic.io) we confirm whether or not the token
is valid and hand you the verified payload associated with the token.

That payload can be many things. For instance, when implementing
[Slack](https://slack.com)-like magic login links you could attach
user's id or email, and when he comes back you know that he has access
to his email account and -- by extension -- is who he claims to
be and can set a session cookie.

All of that is happening without a single password being typed.

The same principle is used for user confirmation and password reset,
only they differ in the action taken upon authentication.

Either way, I hope you like the result. Feel free to contact me if you
have some ideas or feedback -- I'll gladly accept any!
