---
title: "A super-quick way to speed up your containers on AWS"
date: 2020-11-12T22:30:00+01:00
categories:
- aws
- ec2
- iam
- performance
---

We all hate it when it happens. You know what I'm talking about -- your app works perfectly fine locally, you deploy it... and then _bam!_

In my case, I hit a terrible, _terrible_ performance bottleneck.

It all started when I was setting up a whole new service from scratch. A pretty standard container deployment, running on EC2 instances. Nothing too fancy, and it seemed everything went smooth. First version deployed.

Then it happened.

The app was unbearably slow -- with requests taking more than 40s to complete.

My first thought was it had something to do with the fact that dev environments were using access keys as opposed to IAM roles.

That initial hunch turned out to be correct. But not in the way I envisaged.

Either way, I set off to collect some data to confirm my hypothesis.

### How IAM roles work

As suspected, the app was making an unreasonable amount of calls to the AWS Instance Metadata Service (IMDS) -- the machinery that makes IAM roles work.

Up until recently all that was needed to retrieve the temporary credentials was firing a request at `http://169.254.169.254/latest/meta-data/iam/security-credentials/<credentials>`. Unfortunately, this approach posed several [security threats](https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/), which is why a v2 of the IMDS was introduced (IMDSv2).

You might be wondering: so what has changed?

The main change was that instead of retrieving all the meta-data directly you have to obtain a timed token used for accessing the metadata (including IAM role credentials).

These tokens are issued via an endpoint secured on the IP protocol level. The server responds with packets that have their TTL set to a small value -- by default it's 1. What does this mean in practice?

It means that even though the token requests to IMDSv2 were successful, the responses never reached the container. Instead, they reached the EC2 hosts and (due to their TTL expiration) terminated there.

But wait: wouldn't it mean the app was unable to respond at all?

As it turned out, not in this case -- the app returned a 200.

### Let's break it down

To visualize what's going on I used this `tshark` snippet:

```
tshark -i ens5 -Y http -T fields \
    -e frame.time_delta_displayed -e ip.ttl \
    -e http.request.method -e http.request.full_uri \
    -e http.response.code -e http.response_for.uri
```

Using this snippet I captured packets on my EC2 host. Then I fired a request to my app.

For brevity, I used the endpoint that was the quickest to respond (it had just a single AWS API call).

```
Capturing on 'ens5'
0.000000000	63	PUT	http://169.254.169.254/latest/api/token
0.000268694	1			200	http://169.254.169.254/latest/api/token
0.640535677	64,1			200	http://169.254.169.254/latest/api/token
0.360916710	63	PUT	http://169.254.169.254/latest/api/token
0.000260324	1			200	http://169.254.169.254/latest/api/token
0.630918120	64,1			200	http://169.254.169.254/latest/api/token
0.370845977	63	GET	http://169.254.169.254/latest/meta-data/iam/security-credentials/
0.000305874	255			200	http://169.254.169.254/latest/meta-data/iam/security-credentials/
0.000652059	63	GET	http://169.254.169.254/latest/meta-data/iam/security-credentials/<credentials>
0.000243943	255			200	http://169.254.169.254/latest/meta-data/iam/security-credentials/<credentials>
```

Let's get started.

As you can see the AWS SDK first tried retrieving the IMDSv2 token. Due to TTL of 1 the response never made it to the container. The app then tried to get the IMDSv2 token once more -- and failed again. Finally, we can see it retrieved the IAM credentials successfully (200 status code and TTL of 255). How?

Here's the deal: the IMDSv1 is still enabled by default -- it just isn't SDK's default choice. Only when the SDK detected it was unable to fetch the IMDSv2 token, did it fall back to using the older service.

In the end, even though the app returned successfully it incurred a delay of over 2s.

For a single AWS API call.

### Mitigation

Luckily, the remedy is pretty straightforward. You can change the TTL value returned by the IMDSv2 token endpoint easily using the following command:

```
aws ec2 modify-instance-metadata-options --instance-id <instance-id> \
    --http-put-response-hop-limit 2
```

Finally, with this small change the response times are back to normal.

### Final thoughts

Even though this change eliminates the latency issues we may have, let's not forget that the primary reason IMDSv2 was introducted was to _improve security_. As such, I deeply recommend disabling the IMDSv1 endpoint altogether.

However, for this to work you must ensure nothing is relying on the presence of IMDSv1. This entails things like upgrading to the latest AWS SDKs. But it's not just about the SDKs. One commonly found example is the AWS CLI v1 -- that still ships with many distros.

But that's another story.
