---
categories:
- Software Engineering
- Relational databases
comments: true
date: "2018-12-20T17:00:00Z"
title: "Abandon relations all ye who enter here: a treatise on silver bullets"
---

Recently I read an article about [*The Guardian* migrating from Mongo to Postgres](https://www.theguardian.com/info/2018/nov/30/bye-bye-mongo-hello-postgres). What struck me far more than the article itself, was the heated discussion going on under the reddit post.
<!--more-->

As if the topic wasn't heated enough it was titled *Bye bye Mongo, Hello Postgres*, which immediately attracted attention of all the relational folks out there naysaying NoSQL solutions. It reminded me some points of a conversation I had several days ago with a couple Oracle zealots.
Don't get me wrong, Mongo has its flaws -- but on the other hand so does any other solution *including* RDBMSs. However, there are people that strongly believe that not only are they the appropriate solution, but also the *single* solution to all problems humanity has ever had (in terms of software engineering). Including writing all your business logic as stored procedures[^1]. Including serving web pages[^2]. Including Bitcoin mining[^3]. Well, everything.

Have you ever heard the term *jack of all trades*? Let's face the truth, **relational databases are not a silver bullet**.

I'm going to support my claim (or a bold statement, if you will -- pun intended) in a moment but first let's stop and think what is the current state of affairs. Ever since the relational boom a couple decades ago, people have started putting everything in relational databases -- and without questioning it. If you want to store an image and associate it with some data (say it's a user's avatar), what does make more *sense*: to put it together with user data into the database *or* rather save it as a file share and just save a pointer to it? Because both are definitely possible.

Luckily, the NoSQL initiatives have emerged and helped to eschew some of the weird practices -- not that it's impossible to do such things in NoSQL databases, because in some it is. The NoSQL initiatives have been more of an eye-opener in that they showed that storage does not neccessarily equal a "SQL"[^4] database. That there are other ways to do it.

There are even ways which eliminate the database from the equation. If you come to think of it -- do you even *need* a database when all you're doing is publish some content, like a blog? You could just use plain old files for storage. It's no wonder static sites and generators like Jekyll and Hugo are becoming increasingly popular. Serving a static website is both more time- and cost-effective. With services like S3 it really is infinitely cheaper and also more reliable.

I'd argue that many of the apps (especially simple CRUD apps) don't need the goodies relational databases come with, yet choose to pay for them. SQL databases are good because you can do anything with them -- or so they say. You can run arbitrary queries and be happy with it. But is it true? 

As it turns out, beyond certain scale you are not allowed to do some queries, some types of updates, etc. or else the rest of your system dies due to throughput issues.

Some of you might say: *but there are read replicas!* Yes, there are read replicas, but then due to asynchronicity of the process your data is not consistent. You lose the very consistence RDBMS zealots put in front of their list of arguments. And if you're doing synchronous replication -- it doesn't solve your performance issues.

Most smart people have already agreed that eventual consistence is a good thing and just learn how to deal with it. Obviously it would be better to have consistent data at all times but sometimes it's just inevitable.

So what should we do? Use NoSQL? Use SQL?

In many aspects it has become more of a political matter rather than technical. NoSQL vs SQL has become another instance of the endless *vim vs emacs* debate. I'm an Emacs user myself but if you're a Vimmer and it *works for you* then why should I bother. NoSQL vs SQL is exactly that -- except this time you don't have to pick camps.

Anyone who has worked in this industry long enough knows that there's no optimal solution for *anything* but the simplest problems (which may be complex on their own, but that's another story). There are more ways to solve problems, to *do* things.

You can have an algorithm that does something optimally in terms of time complexity, or you can have an algorithm that does something optimally in terms of space complexity, but usually not both. Which one you choose to optimize for should be your informed choice. This brings us to the next topic -- engineering.

I'm a huge fan of engineering. In this industry probably the vast majority of us are engineers -- and yet we very often tend to forget what engineering is all about. So what is engineering one might ask?

It's the process of applying *knowledge* to a problem. It's about making informed choices. It's about *solving* your specific problem instance given particular constraints.

I'm also a huge fan of Haskell and yet, if I had to deliver a product quickly Haskell would never be my first choice. Because Haskell is about solving puzzles and not problems. It also has many other properties that would hinder my time-to-market. Therefore I'd decide not to use it -- I'd much rather use a Lisp, due to its flexibility.

Does it make more sense what I wrote earlier that you don't have to pick sides in the SQL vs NoSQL battle?

Because why not use both? You could have part of your system require extreme performance that no SQL database would ever deliver, and then you could normalize the data for other purposes and feed it into a RDBMS and perform some analytics. If your *use cases* dictate such properties then it's all for the best. Choosing just one piece of the equation will leave you with one variable missing.

The key point is that you should stop and think before choosing blindly a solution -- be it SQL or NoSQL or else -- and apply KISS principle to anything you do. I think [this article by Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/05/15/NODB.html) sums it up pretty nicely.

Remember -- there are no silver bullets -- apply engineering to your input.

[^1]: An actual example of a "great" idea presented by one of the Oracle people
[^2]: Also an actual example of a "great" idea given -- at this point I was rendered speechless
[^3]: A made up and exaggerated idea to support my claim. But it's not *that* far-fetched if you asked the *O* people
[^4]: Used for brevity: as opposed to NoSQL databases