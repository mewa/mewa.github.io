---
title: "Where to host your project as a solo-founder?"
date: 2020-08-23T22:43:55+01:00
categories:
- AWS
- GCP
- Heroku
- Azure
- Netlify
- DigitalOcean
---

Recently I asked solo-founders what are their choices when it comes to
hosting their projects. Here are the most common choices along with
their rationales.

<!--more-->

When it comes to hosting our projects we have several options:
* platforms
* cloud infrastructure
* serverless

## Platforms

#### Heroku

Heroku has been a very popular option for hosting projects for several
years now. It's very easy to set up and has a wide range of plugins
such as PostgreSQL, Redis or NewRelic.  It also has a huge community
raised on building products on Heroku.

However, it can get fairly expensive as you scale your app. Their
resource tiers are also very arbitrary and sometimes force you into
paying for way more than what you actually need. From my personal
experience, reliability on Heroku can also be a pain.

#### Netlify / Vercel

Netlify and Vercel seem to have a pretty similar offering. For some
reason, I see Netlify more often, but Vercel as the creators of Next.js
framework are not to be dismissed. If you want an in-depth [comparison
of Vercel and
Netlify](https://www.lambrospetrou.com/articles/battle-of-jamstack-platforms-netlify-vercel-aws/),
Lambros Petrou wrote a nice post summing up their capabilities.

Both options are a very very popular option for hosting static pages
-- and for a good reason. Easy to deploy, both feature built-in CI/CD
along with commit previews. And of course, they employ a fast CDN so
that your site load times are very small.

If you're into serverless (which really makes sense as a
solo-founder), they also have the option of running cloud functions.

#### Firebase

Firebase is another popular option and offers a suite of tools for
building web apps rapidly. This includes everything from storage,
through identity management to cloud functions. Others have mentioned
it can get pretty expensive, especially when it comes to their hosting
offering.

## Virtual machines

Perhaps what was fairly surprising to me was that VMs (virtual
machines) are still a very popular option among Indie Hackers.

They require a little bit of upfront setup and ongoing maintenance,
but it's definitely on the cheaper end, which is definitely alluring
when bootstrapping. Personally, unless that's something you're dealing
with daily I wouldn't recommend going this route -- and
even if it's something you're familiar with I find it that time is
better spent on building your offering.

#### DigitalOcean

Together with Linode and Vultr, DigitalOcean is a very popular option among
those that choose to run on their own VMs.

When designing fault-tolerant applications it's good to bear in mind
that their SLA (service-level agreement) only guarantees 99.99%
uptime. While it's the same level major cloud providers offer,
DigitalOcean only applies the lost credits 1:1, which may have some
implications if you have SLAs for your own customers.

Pros:
* cheap _shared_ VMs
* different regions worldwide

Cons:
* dedicated VMs aren't that much cheaper than other major cloud
  providers
* SLA credits policy leaves much to be desired

#### Linode

Even though Linode, DigitalOcean and Vultr keep competing with
one another and their offerings tend to even out pretty quickly after
one of them introduces reductions at the point of writing this article
Linode has a much better price point for dedicated machines and offer
more CPU cores than their competitors.

Similarly to DigitalOcean, Linode has a fairly mediocre SLA, where
they refund lost credits at a 1:1 ration.

Pros:
* cheap _shared_ and _dedicated_ VMs
* different regions worldwide

Cons:
* SLA credits policy leaves much to be desired

#### Vultr

What makes Vultr stand out out of the three cloud providers is their
lowest shared tier -- starting at $2.5 without IPv4 address or $3.5
with IPv4 included.

Unlike DigitalOcean and Linode, they offer a 100% uptime guarantee and
apply credits far more generously.

Pros:
* cheap _shared_ VMs
* different regions worldwide
* compelling SLA

#### Hetzner

If your project is targeted at EU audience, Hetzner is one of the most
cost-effective options out there. With prices starting as low as €3
for a shared VM and €24 for a dedicated one, their offering really is
a steal.

Pros:
* _very_ cheap VMs
* _very_ cheap bandwidth

Cons:
* EU-only

## AWS / GCP / Azure

With their rich service offering, these three major cloud providers
can fit in any of these categories -- from basic VMs to platforms
such as the AWS Elastic Beanstalk, to serverless functions.

This diversity is both a boon and a curse -- while you won't ever need
to leave a cloud provider once you choose it, there's a good chance
you'll find a significant portion of your time navigating around this
complexity, especially in the beginning.

Most notably, AWS (as the first public cloud) has an astounding number
of services, tailored to specific use-cases, most of which you'll
never use.

That being said, all of them offer discounts for committed usage of
their VMs -- something that is lacking in the offering of the other
cloud providers. If you can decide how much resources you're going to
need in advance this can bring you considerable savings.

## Serverless

Interestingly enough, nobody on the thread mentioned serverless
services like AWS Lambda. Personally, I think they're a great fit for
a bootstrapping founder as the less time you're spending managing your
services, the more time you have for growing your product.

I attribute it to lower awareness due to its relatively young age. One
major drawback I see here is that the tooling can be quite lacking
around serverless and there's a considerable time investment in terms
of the boilerplate required.

That's why platforms like Firebase, Netlify and Vercel that provide
that additional tooling are far more popular, even though they're
essentially leveraging the same building blocks. My prediction is that
as tooling in the serverless ecosystem matures, the benefits of those
platforms will diminish.

Thanks for tuning in and I hope you are now better equipped to choose
the provider for your next project! As always, feel free to leave a
comment on
[Twitter](https://twitter.com/intent/user?screen_name=marcin_chm), my
DMs are open as well.
