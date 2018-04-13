---
categories:
- git
- CI/CD
title: "Using git notes to improve CD workflow"
date: 2018-04-12T17:34:29+02:00
comments: true
draft: true
---

Git is certainly one of the most widely used tools in software development industry -- it's essential.
However, many people forget about the many options it provides apart from serving simply as a VCS.
I think `git notes` is one of such commands, which can immensely improve your workflow -- let's see for ourselves why you'd like to start using it.
<!--more-->

I'm not going to try different words, since in this case [git docs](https://git-scm.com/docs/git-notes) covers it nicely, in short:

> git-notes
>
> Adds, removes, or reads notes attached to objects, without touching the objects themselves.

The key part we're about to exploit is *without touching objects*.

But first, let's begin with the following scenario.

### The scenario

We're running a company `example.com` which has two services:

* a frontend service, located in a GitHub repository `example.com/frontend`
* and a backend service, located in a GitHub repository `example.com/backend`.

Since `frontend` and `backend` depend on each other, we'll have another repository, `example.com/deploy` which covers
end-to-end tests as well as manages configuration of each service.

Each repository has two branches:

   * master
   * and production

On `master` we will find the latest code, it may contain bugs.

We're doing serious business obviously, so it's vital for our customers to only access code that has been tested thoroughly.
Hence we're setting up a CI service to ensure the quality of our product. Commits to `master` automatically trigger our CI service,
which then tests and builds our code -- and if it succeeds it commits information about build artifacts, so that they can be retrieved easily
in the following steps, and merges the changes to `production`.

Naturally, changes flow further to `example.com/deploy` repository where they are integrated and prepared for delivery/deployment. Thanks to the fact
that we have saved the build artifacts, we don't have to rebuild them again (that would've been a waste, right?) and all that's left is configuration
and testing.

Now, this is great, because we have the confidence to push our code further in in our deployment pipeline. Or do we?

As it turns out, not so much. This approach revolves aroud the assumption that the developers will push their code to the `master` branch.
But what happens if they don't? Right now it's entirely possible -- whether by mistake or deliberately -- to skip tests and possibly taint our production
environment by simply pushing directly to `production` branch.

Since we're committed to delivering the highest quality product, even if it sounds improbable, we have to take measures against it.
Luckily, we can just protect our `production` branch and make GitHub enforce for us that only tested changes can be merged in.

However, we're hitting another problem. Previously, after a successfult build we were committing information about the build artifacts to production branch.
Well, we cannot do that now, since doing so would mean we're introducing *untested* changes that cannot be merged into `production` --- even
if we know for a fact it's just metadata.

### git notes revisited

This is where `git notes` command comes in handy. Just as I have already mentioned it lets us attach objects (data) without modifying our commit hash.

There is one caveat:

git notes are separate refs that can be overwritten or even deleted. This means that critical data shouldn't be stored in them.

Information about build artifacts, however, is not critical, because it can be easily reproduced through building our production branch again.
