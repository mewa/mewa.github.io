---
categories:
- Clojure
- Kubernetes
- Continuous Deployment
- Continuous Integration
title: "Start a CI service in 100 lines of Clojure"
date: 2018-05-02T02:18:29+02:00
comments: true
draft: true
---

There are many factors that contribute to the quality of the code we produce. Undoubtfully, adopting Continuous Integration
is one of the biggest leaps one can make when closing the gap between *The Holy Grail Of Software Engineering*.

Over time there have emerged quite a few CI services that make it easy to integrate changes to our code.
Unless you've been living in vacuum for the past few years, you must've heard names like Jenkins, TravisCI or CircleCI.

But what if I told you, you could roll your own YetAnotherCI in just around 100 lines of Clojure? If this sounds interesting, make
sure to follow along and ship YACI with us.
<!--more-->

### Preliminaries

While various CI services may have some differences between them, ultimately they have one goal:

1. They have to run some code,
2. and they have to report what the outcome is.

The code will usually involve triggering tests or trigerring a build, but in principle it doesn't matter what kind of code
it runs, as long as the end user is satisfied with the result it produces (much like in classic programs).

When designing a CI service we also have to take into account that there will be many users, submitting jobs concurrently --
and their submissions should be treated more or less equally. In order to ensure that we'll have to employ some sort of
a queueing mechanism.

Then, we have another factor -- we have to assign these jobs to our worker machines. We can't place all jobs on a single machine,
or else it will choke under the load. We have to enforce some limits and make sure worker nodes only get more work when they are
ready to receive it. That's where we arrive to the next point -- we need a scheduler that will poll our queue and assign jobs
accordingly.

Now that we know what we're up to, let's get down to work.

### Orchestration

Luckily for us, all the things I have mentioned above have already been done, and instead of writing our own queues and schedulers we can
just roll a container orchestrator like [Kubernetes](https://kubernetes.io/) which will take care of all that.

If you want to get an overview of what Kubernetes is, [Kelsey Hightower gave a great talk on Kubernetes]
(https://www.youtube.com/watch?v=HlAXp0-M6SY) at PuppetConf 2016.

However, for the purpose of this article it will suffice that you know that you can just give your tasks to Kubernetes and it will run them
using available resources.

Conveniently enough, Kubernetes also has an API we can easily interact with.

That's a huge leap forwards, because what we now have to implement is basically a wrapper around this API, that will hide the details from
our end users.

But first let's get a Kubernetes cluster we could work on. Incidentally, I already had one in my pocket (check yours too) -- but if you don't
have one, you can get one up and running from most cloud providers in a matter of minutes.

Alright, now that we have our cluster, let's proxy it, so the API is accessible locally.

```sh
mewa@sea$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

Let's confirm it's working:

```sh
mewa@sea$ curl localhost:8001/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "35.205.116.105"
    }
  ]
}
```

Great, let's get down to writing our API wrapper.

### Writing our API

First of all, as listed earlier we have one goal --- to run code and report. Since CI jobs can be long running, we're
going to split it into 2 separate endpoints: one for posting a job, another for retrieving info about it.

We're going to use [ring](https://github.com/ring-clojure/ring) for our API server.

```clojure
(require '[ring.adapter.jetty :as jetty])
```

Next, we create `/run` and `/get` endpoints, which will run their respective handlers. We'll expose them at port 4000.

```clojure
(defn -main
  [& args]
  (jetty/run-jetty (route {"/run" run-handler
                           "/get" get-handler}) {:port 4000}))
```

Before we can actually write these handlers we'll need code for interfacing with Kubernetes API. Let's start with posting new jobs.

```clojure
(require '[kubernetes.api.batch-v- :as k8sbatch])

(defn new-job
  "Create job which executes CMD"
  [cmd]
  (let [job-name (str "k8s-job-" (java.util.UUID/randomUUID))]
    (k8sbatch/create-batch-v1-namespaced-job
     "default"
     {:metadata {:name job-name}
      :spec {:template {:spec {:containers [{:image "alpine"
                                             :name "k8s-job"
                                             :command ["sh" "-c" cmd]}]
                               :restartPolicy "Never"}}}})
    {:name job-name}))
```

Before we can verify it's working, we'll need to supply a context for the Kubernetes API. Let's create a helper function that will take
a function and run it in the context of our proxied Kubernetes API.

```clojure
(require '[kubernetes.core :as core])

(def kube-config {:base-url "http://localhost:8001"})

(defn run-k8s [f & args]
  (core/with-api-context kube-config
    (try (apply f args)
         (catch Exception e {:error (or (ex-data e) e)}))))
```

Let's run it in a REPL

```clojure
repl=> (run-k8s new-job "echo success")
{:name "k8s-job-5079d2cf-acc7-4787-96d6-12d83be720a6"}
```
and verify the job was created

```sh
mewa@sea$ kubectl get jobs
NAME                                           DESIRED   SUCCESSFUL   AGE
k8s-job-5079d2cf-acc7-4787-96d6-12d83be720a6   1         1            1m
mewa@sea$ kubectl get pods \
    --selector job-name=k8s-job-5079d2cf-acc7-4787-96d6-12d83be720a6 \
    --show-all
NAME                                                 READY     STATUS      RESTARTS   AGE
k8s-job-5079d2cf-acc7-4787-96d6-12d83be720a6-2fs6t   0/1       Completed   0          1m
mewa@sea$ kubectl logs k8s-job-5079d2cf-acc7-4787-96d6-12d83be720a6-2fs6t
success
```

Great, it seems to work. Now let's retrieve those logs programatically.

The steps required are identical to what we've been doing in with `kubectl`.

1. Retrieve pods for job `id`,
2. Retrieve logs for pods returned

```clojure
(require '[kubernetes.api.core-v- :as k8scorev])

```
