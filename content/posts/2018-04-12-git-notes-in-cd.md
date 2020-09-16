---
categories:
- git
- Continuous Deployment
- Continuous Integration
title: "Using git notes to improve workflow"
date: 2018-04-12T17:34:29+02:00
comments: true
---

## TL;DR

There used to be a post about using git notes in a hypothetical CD
scenario, but since I thought it wasn't providing any value anymore I
decided to rake it and instead I've limited it to describing `git
notes` features and linking to the [official documentation](https://git-scm.com/docs/git-notes).

<!--more-->

> git-notes
>
> Adds, removes, or reads notes attached to objects, without touching the objects themselves.

In practice this means we can give our commit a lengthier description,
if we want to bring something to the attention of our readers. Explain
the rationale behind it. Add some other important, well, notes. And
while we could just amend the commit changing the commit message,
using `git notes` we're able to provide that additional context
without altering the commit itself -- which is important if your
scenario doesn't allow for rewriting git history.
