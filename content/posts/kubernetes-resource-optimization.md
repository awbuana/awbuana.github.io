---
title: "The Resource Request You Think Is Saving Money Is Actually Breaking Your App"
date: 2026-03-19
description: "Most developers set Kubernetes resource requests wrong. Here's what I learned about CPU throttling, bin packing, and why your optimized GKE cluster might be the problem."
tags:
  - kubernetes
  - devops
  - performance
  - gke
  - cost-optimization
draft: false
---

I thought I was being clever.

When we migrated our services to Google Kubernetes Engine with [auto scale profile optimized](https://cloud.google.com/blog/products/containers-kubernetes/gke-features-to-optimize-resource-allocation), I looked at our resource specs and saw an opportunity. Our pods were requesting 100m CPU but had limits set to 1000m. Ten times headroom! Surely we could tighten that up and save some money.

So I did what seemed logical: I kept the limits high (just in case of traffic spikes) but dropped the requests even lower. 50m here, 25m there. The cluster was happy. Our costs went down. I patted myself on the back for being such a savvy engineer.

Then the Kafka consumers stopped consuming.

Not all of them. Just... some of them. Randomly. One consumer would process thousands of messages per second, another would hang for minutes at a time. No errors in the logs. No memory pressure. The pods were "healthy" according to Kubernetes. But they weren't *working*.

## The Difference Between Request and Limit (That I Didn't Understand)

Here's what I thought I knew about Kubernetes resource management:

- **Request**: What the container *needs* to run
- **Limit**: What the container is *allowed* to use

Seems straightforward, right? Set requests low to pack more pods onto fewer nodes. Set limits high so pods can burst during traffic spikes. This is what "optimized" looks like.

What I didn't understand was how GKE auto scale profile optimized (and any node auto-provisioning system) actually uses these values.

When you set a CPU request of 50m (that's 0.05 CPU cores, by the way), GKE looks at that number and says "I need to provision a node that can guarantee this pod 50m of CPU." It doesn't look at your limit. At all. Your 1000m limit might as well not exist when GKE is deciding how big your nodes should be.

So you end up with nodes that are sized for your *requests*, not your *actual usage*. And here's where it gets painful.

## CPU Throttling: The Silent Killer

When your pod tries to use more CPU than its request, Kubernetes doesn't stop it immediately. That would be too obvious. Instead, it uses something called CFS (Completely Fair Scheduler) quotas to throttle the container's CPU usage.

The math works like this: if you request 50m CPU, your container gets allocated 50ms of CPU time every 100ms period. Use it or lose it. If your container needs more than that 50ms in a given period, it gets throttled. It has to wait for the next period.

To the application, this feels like... random latency. Sometimes fast, sometimes slow. Sometimes your Kafka consumer processes a batch in 50ms, sometimes it takes 500ms. Sometimes your API responds in 10ms, sometimes in 200ms.

We spent weeks chasing this. We thought it was a network issue. Then a Kafka configuration problem. Then maybe GC pressure from our Go services (we're a Go shop). We added metrics, traces, alerts. Everything looked fine except... the application just wasn't performing consistently.

Then I stumbled on a GitHub issue about CPU throttling. Someone mentioned running `kubectl top pod` and comparing the CPU usage to the request. I ran it on our problematic service:

```
NAME                     CPU(cores)   MEMORY(bytes)
consumer-7d9f4b8c5-x2v9   48m          512Mi
```

48m actual usage. 50m request. We were living right at the edge, and any slight burst would trigger throttling. The "high" limit of 1000m was a fantasy. The node didn't have that CPU to give.

## Bin Packing: The Optimization Trap

Here's the thing that really stung: this was all working "as designed."

GKE with auto scale profile optimized is doing exactly what we asked it to do. We told it we only needed 50m CPU, so it packed 40 of our pods onto a single 2-core node. That's efficient! That's cost-effective! That's exactly what we said we wanted!

The problem is that "bin packing efficiency" and "application performance" are often in direct conflict. The more densely you pack pods onto nodes, the more likely you are to hit resource constraints.

I think of it like airplane seats. You *can* fit more people on the plane by removing legroom. The plane is technically more efficient. But nobody's comfortable, and the experience degrades for everyone.

But there's an even bigger risk we didn't consider: the "all eggs in one basket" problem. When you pack 40 pods onto a single node, you're centralizing your entire deployment. If that node goes down—hardware failure, maintenance, whatever—you're not losing one pod. You're losing 40. It's the classic operational risk of concentrating too much in one place.

The real kicker? We weren't actually saving money. Yes, our GKE bill was lower. But we were paying for it in:

- Engineering time spent debugging performance issues
- Slower message processing (which meant larger Kafka backlogs)
- Customer complaints about intermittent latency
- The cognitive load of "why is this slow *sometimes*?"
- Higher operational risk from over-centralized workloads

## What CPU Throttling Actually Looks Like

Let me give you some specific symptoms we saw, because I wish someone had handed me this list six months ago:

**Kafka consumers that randomly stop keeping up with producers.** Your lag grows, then shrinks, then grows again. No pattern you can correlate to traffic. The consumer group is "healthy" but just... slow.

**API endpoints with bimodal latency distributions.** 90% of requests take 20ms, 10% take 200ms+. Same endpoint, same payload, same time of day. The slow ones aren't errors—they're just throttled.

**Applications that hang during startup.** Your pod passes the readiness probe eventually, but it takes 3 minutes instead of 30 seconds because the initialization code keeps getting throttled.

**Metrics that look fine.** CPU usage is well under the limit. Memory is stable. No restarts. No OOMKills. But the application is clearly struggling.

The pattern to look for: **inconsistent performance without obvious resource exhaustion.**

## How to Actually Set Resource Requests

So what should you do instead?

First, stop treating requests as "what I hope the container uses" and start treating them as "what the container actually needs during normal operation." This requires measuring, not guessing.

Here's the workflow I use now:

**Step 1: Measure actual usage**

Run your service under realistic load and watch the CPU usage patterns. Don't just look at averages—look at P95 and P99. You want to understand what "busy" looks like, not just what "typical" looks like.

```bash
# Watch CPU usage in real-time
kubectl top pod -l app=my-service --containers

# Or get historical data from Prometheus/metrics-server
```

**Step 2: Add headroom for bursts**

Once you know your P95 usage, add 20-30% headroom. If your service typically uses 200m but bursts to 300m during traffic spikes, request 350m. Not 50m.

The goal isn't to maximize node packing. The goal is to ensure your application has the resources it needs to perform consistently.

**Step 3: Set limits closer to requests (usually)**

Here's a hot take: your limits should probably be close to your requests. Like, within 50% of each other.

I know, I know. "But what about traffic spikes?" If you get traffic spikes, you should be scaling horizontally (more pods), not vertically (more CPU per pod). Horizontal scaling is what Kubernetes is good at. Relying on vertical bursting is a recipe for inconsistent performance.

The exception: batch jobs or background workers that genuinely have variable resource needs. But for stateless web services? Keep them close.

**Step 3b: Right-size your containers (critical for bin packing)**

Here's something that took me too long to learn: effective bin packing depends entirely on right-sizing. If your containers are too large—say, requesting 75% of a node's capacity—you won't be able to fit additional containers onto the node regardless of how clever your packing strategy is. You'll essentially force waste.

The resources you allocate to a container and the resources a machine offers are critical factors. Before you optimize your bin packing, make sure your container sizes make sense for your node sizes. Otherwise you're optimizing the wrong thing.

**Step 4: Monitor throttling explicitly**

Add this metric to your dashboards:

```promql
rate(container_cpu_cfs_throttled_periods_total[5m]) /
rate(container_cpu_cfs_periods_total[5m])
```

This gives you the percentage of CPU periods that were throttled. If it's consistently above 0, you're under-provisioned.

## The GKE Auto Scale Profile Optimized Gotcha

If you're using GKE with [auto scale profile optimized](https://cloud.google.com/blog/products/containers-kubernetes/gke-features-to-optimize-resource-allocation), there's one more thing to understand: the node provisioning is based entirely on your requests, not your actual usage or your limits.

This means that setting low requests and high limits doesn't give you "room to burst." It gives you "nodes that are too small for your actual needs, plus a false sense of security."

When we fixed our resource specs—bumping requests from 50m to 300m for most services—GKE provisioned larger nodes. Our costs went up. But our performance issues disappeared overnight. The Kafka lag graphs went from erratic sawtooth patterns to clean flatlines. Our API latency became predictable.

Was it worth the extra money? Absolutely. We spent more on infrastructure and less on engineering time chasing ghosts.

## Memory: The Non-Compressible Resource

One more thing worth mentioning: CPU is compressible—you can throttle it without crashing your application. Memory is not.

With CPU, if you hit the limit, your process just slows down. With memory, if you hit the limit, your process gets OOMKilled. Period. This is why you should always set hard limits for memory, and why memory allocation deserves more careful consideration than CPU.

In our case, we got lucky—our memory requests were actually reasonable. But I've seen teams apply the same "low request, high limit" strategy to memory and watch their pods get evicted constantly. Don't do that. Set memory requests and limits close together, and base them on actual measured needs, not wishful thinking.

## A Framework for Resource Decisions

Here's the mental model I use now when setting resource specs:

1. **Request = Guarantee**: This is what Kubernetes promises your container. It's also what bin-packing algorithms use to schedule your pods. Set it based on *actual measured needs*, not wishful thinking.

2. **Limit = Safety Net**: This is what prevents a runaway process from taking down the node. It's not a performance target. Don't set it 10x higher than your request unless you truly understand what you're doing.

3. **Measured Usage = Truth**: Your application uses what it uses. You can't negotiate with CPU requirements. Either give it enough resources, or accept the throttling consequences.

4. **Cost vs. Performance = Trade-off**: There is no free lunch. You can optimize for cost (low requests, dense packing) or you can optimize for performance (higher requests, more headroom). You can't optimize for both simultaneously.

---

**Further reading:**
- [GKE features to optimize resource allocation](https://cloud.google.com/blog/products/containers-kubernetes/gke-features-to-optimize-resource-allocation) - Google's official documentation on auto scale profile optimized
- [Optimizing Resource Utilization: the Benefits and Challenges of Bin Packing in Kubernetes](https://www.infoq.com/articles/kubernetes-bin-packing/) - Excellent deep dive on balancing density vs. isolation
- [Kubernetes CPU limits and CFS quotas](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)
- [GKE Autopilot resource management](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests)
- [Understanding CPU throttling in Kubernetes](https://medium.com/@ramangupta/understanding-kubernetes-cpu-limits-and-throttling-5e0c9d3f1c1e)
