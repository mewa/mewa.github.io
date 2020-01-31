---
title: "Things to consider when choosing nodes for your Kubernetes cluster"
date: 2020-01-31T01:43:55+01:00
categories:
- Kubernetes
- AWS
- GCP
---

Regardless of whether you're architecting new services
or preparing to migrate your workloads -- choosing the
proper underlying nodes size can make or break your experience with
this amazing tool. Here are some key points you should pay attention to.
<!--more-->

##### The basics

Every Kubernetes cluster has only as much computing capacity as the
nodes it consists of, minus the overhead. It might seem that the
only metric you should pay attention to is the combined amount of CPU
and memory they provide. While this is true in general, there are
some important points to keep in mind.

As Kubernetes nodes are often used in context of autoscaling in might
be tempting intitially to use the smallest granularity unit. As
mentioned, however, there is a cost to running a Kubernetes node. For
this reason alone you should refrain from using the weakest nodes --
it makes no sense to spin up a node that only handles the Kubernetes
internals with very little room for user-space apps.
Picking a bigger also node makes the operating overhead smaller
comparatively to our cluster size. Although you don't want the biggest
nodes either as it makes controlling costs very difficult -- just
imagine having a granularity of 128 CPUs.

Then there's also the availability aspect. Having more nodes means the
workloads may be more evenly distributed. Especially replicated
services benefit from having smaller nodes as in case of a failure
only a smaller portion of the service gets affected. I'm not going to
delve deep into availability concerns in this post as this is
something that's usually defined as part of the business strategy.

But let us return to the cost aspect -- in practice, there is always
going to be a trade-off between unused capacity and capacity wasted on
operating the cluster. The overhead will increase or decrease
depending on how many additional apps you are running on each node --
usually centered around monitoring. For the sake of this article let's
assume an 8 CPU node with 32GB memory strikes a good balance for
powering a general purpose cluster.  Will it be able to run any
workload combination that uses less than 32GB memory in total[^1]?
Most of them, yes -- but not any. As it turns out there are some more
caveats.

###### Pod capacity

Every node has a certain capacity in terms of how many pods it can
run. Let's say you wanted to run a larger number of very light pods,
each requiring 256MB of memory. In theory that should give us a little
over 120 pods per node. But then Real World kicks in. An example?
Almost all AWS (Amazon Web Services) nodes that have 8 CPUs available can only assign 60 IP
addresses. Since each pod requires to be IP-addressable that
effectively limits our capacity by 50%. One might conclude saying it's
just yet another limitation to be mindful of. And such a statement
would've been correct under the assumption that the IP address space
scaled linearly -- which to be honest feels more natural. But
a 16 CPU node can actually assign 240 IP addresses. And so can a node
with 32 and 48 CPUs.

Apart from the hard limits dictated by your cloud provider there are
also guidelines by the Kubernetes community. They include some real
numbers backed by load tests as to what to numbers to pick in order to
ensure stable cluster operation. In this case, running more than 100
pods per node is discouraged, due to the bookkeeping overhead it
generates. But like I said, it's just a guideline. If you test that
this configuration works for you -- it works for you.

> At v1.17, Kubernetes supports clusters with up to 5000 nodes. More specifically, we support configurations that meet all of the following criteria:
>
> * No more than 5000 nodes
> * No more than 150000 total pods
> * No more than 300000 total containers
> * No more than 100 pods per node
> 
> -- <cite>[Building large clusters -- kubernetes.io](https://kubernetes.io/docs/setup/best-practices/cluster-large/)</cite>

The key takeaway here is to be mindful of both your workloads and the
underlying provider that supplies the nodes -- while keeping in mind
the limitations inherent to computing at scale.

###### Cloud provider limitations

Another instance of such a limitation is the ability to attach volumes
or network devices.

In particular, running pods that require access to block storage (such
as EBS on AWS) will restrict our capacity to around 30 pods (in
theory)[^2]. However, this limit is also used by the network devices
so we end up trading between disk and network capacity.

However, the very same setup on GCP (Google Cloud Platform) wouldn't
have caused issues with the number of volume attachments as it can
accomodate up to 128 volumes in most cases[^3].

##### Wrapping up

Whatever your setup may be make sure to consult the documentation of
your Cloud Provider do the math before deploying. Having multiple
capacity dimensions can lead to scheduling failures that are often
unobvious for the untrained eye. Also, make sure to do some load tests
if you happen not to fall within the limits suggested by the Kubernetes
community.

[^1]: CPU omitted since it can throttle -- memory can't
[^2]: see [Instance Type Limits on AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html#instance-type-volume-limits)
[^3]: see [Machine Types on GCP](https://cloud.google.com/compute/docs/machine-types)
