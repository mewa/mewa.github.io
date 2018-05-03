---
categories:
- Clojure
- Kubernetes
- Continuous Deployment
- Continuous Integration
title: "Launch a CI service in 100 lines of Clojure"
date: 2018-05-02T02:18:29+02:00
comments: true
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

As mentioned earlier, we have one goal --- to run code and report. Since CI jobs can be long running we're
going to split it into 2 separate endpoints: one for posting a job, another for retrieving info about it.

We're going to use [ring](https://github.com/ring-clojure/ring) for our API server.

```clojure
(require '[ring.adapter.jetty :as jetty])
```

Next, we'll create `/run` and `/get` endpoints, which will run their respective handlers. We'll expose them at port 4000.

```clojure
(defn route
  "Given a map of HANDLERS, returns a Ring handler which matches
  requst URIs on map keys and executes handlers associated with
  those keys"
  [handlers]
  (fn [request] (if-let [handler (handlers (:uri request))]
                  (into {:headers {"Content-Type""application/json"}} (handler request))
                  {:status 404 :body "Not found"})))

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

If you've taken a look at Kubernetes job specs you'd see it's actually a 1:1 mapping.

In order to verify it's working, we'll need to supply a context for the Kubernetes API. Let's create a helper function that will take
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

(defn job-pods
  "Get pods for job ID and return a channel with information about
  each pod"
  [id]
  (map (fn [v] {:name (get-in v [:metadata :name])
                :status (get-in v [:status :phase])})
       (:items (k8scorev/list-core-v1-namespaced-pod
                "default"
                {:label-selector (str "job-name=" id)}))))

(defn pod-logs
  "Get logs for each item in channel PODS and return a channel
  with logs for each pod"
  [podinfo]
  (let [pod (:name podinfo)
        status (:status podinfo)]
    {:pod pod
     :status status
     :log (k8scorev/read-core-v1-namespaced-pod-log pod "default")}))
```

Now we can easily retrieve our logs. Note that we have to force evaluation of returned sequence,
with `doall`, to get side effects (run API calls).
```clojure
(defn get-job
  "Get information for job with given ID"
  [id]
  (let [pods (job-pods id)]
    (doall (map pod-logs pods))))
```

Last thing that's left is hooking these functions in as request handlers in our API. It's pretty straight-forward.

```clojure
(require '[cheshire.core :refer :all])

(defn run-handler
  "Ring handler which parses REQUEST and starts a job"
  [request]
  (let [handler (fn [cmd] (run-k8s new-job cmd))
        resp (-> request
                 :body
                 slurp
                 handler)]
    {:status 200
     :body (generate-string resp {:pretty true})}))

(defn explode-query
  "Explode query string Q into a map"
  [q]
  (reduce #(apply assoc %1 %2) {}
       (map #(s/split % #"=") (s/split q #"&"))))

(defn get-handler
  "Ring handler which parses REQUEST and returns job info"
  [request]
  (if-let [id ((explode-query (:query-string request)) "id")]
    (let [jobinfo (run-k8s get-job id)]
      {:status 200
       :body (generate-string jobinfo {:pretty true})})
    {:status 400
     :body "id query param is required"}))
```

With our handlers being ready, it's time to package our API and run it on our servers.

Let's just do a final check before we finish to make sure I wasn't lying to you:

```sh
mewa@sea$ wc -l < src/k8s/core.clj
95
```

95 lines of code, including docstrings! If we wrote this in Java our imports would have more!

### Wrapping up

Now that our CI is packaged all that's left to do is to buy an `.io` domain, hire a marketing team and start collecting money!

I hope you had as much fun reading through this as I had preparing it. As always, code used in this article is
present [on my Github](https://github.com/mewa/clojure-k8s).

And when you ship --- remember, the `io` part is crucial!
