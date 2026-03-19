---
title: "CPU Limits vs Memory Limits: When 'Survival' Means Different Things"
date: 2026-03-19
description: "I said memory limits are about survival. But what does 'survival' actually mean for stateless vs stateful workloads? The answer surprised me."
tags:
  - kubernetes
  - devops
  - architecture
  - stateful-apps
  - reliability
draft: false
---

In my previous post, I said: **"CPU limits are about performance. Memory limits are about survival."** I stand by that statement, but I oversimplified what "survival" actually means depending on what kind of application you're running.

Let me break this down.

## Stateless Applications: The "Easy" Case

When I wrote about memory limits being "about survival," I was thinking primarily of stateless services. You know, the typical microservices: APIs, web servers, queue consumers.

For these apps, an OOMKill looks like this:

1. **Request comes in**
2. **Memory spikes** (maybe a large payload, maybe a leak)
3. **OOM Killer terminates the pod**
4. **Kubernetes restarts it**
5. **Service continues**

The blast radius is limited. Yes, that one request failed. Yes, there's a brief interruption. But the system recovers automatically. The "survival" here is about keeping the service available, not about protecting data.

This is what I had in mind when I wrote about memory limits. And honestly? This is the best-case scenario for an OOMKill.

## Stateful Applications: Where It Gets Painful

Here's where I need to correct myself. For stateful applications, memory limits aren't just about survival—they're about **preventing catastrophic data loss and expensive recovery operations**.

### Databases

When your PostgreSQL or MongoDB pod gets OOMKilled:

- **Active connections drop** immediately
- **In-flight transactions rollback** (if you're lucky and have proper transactions)
- **Write-ahead logs might not be fully flushed**
- **Replication lag** increases while the replica restarts
- **Cache is cold** after restart, causing performance degradation

The pod restarts, sure. But now you're potentially dealing with:
- Data consistency issues
- Replication catch-up time
- Connection storms as clients retry
- Query performance degradation due to cold caches

### Message Queues (Kafka, RabbitMQ, NATS)

This is where I learned my lesson the hard way.

When a Kafka broker gets OOMKilled:

- **In-flight messages are lost** if they weren't replicated yet
- **Consumer lag spikes** while the broker restarts and recovers
- **Partition rebalancing** causes additional latency
- **Replication stalls** as followers catch up

We had this happen in production. A broker with insufficient memory got killed during a traffic spike. The cluster survived (Kafka is designed for this), but:
- We lost about 30 seconds of messages that hadn't replicated
- Consumer lag went from 0 to 50,000 messages
- It took 20 minutes for the cluster to fully rebalance
- Our "exactly once" processing guarantee became "at least once" (duplicate processing)

The pod survived. The data mostly survived. But our guarantees didn't.

### Caches (Redis, Memcached)

Redis getting OOMKilled is especially painful:

- **All in-memory data is gone** (unless you have persistence enabled)
- **Cache warming** takes time and hits your database hard
- **Session data lost** if you're using Redis for sessions
- **Rate limiting counters reset** if you're using Redis for throttling

I once saw a Redis instance with 40GB of cached data get OOMKilled. It took 3 hours to rebuild the cache from the database, during which our database CPU was pegged at 100%.

### Distributed Systems (etcd, ZooKeeper, Consul)

These are the worst-case scenario.

When an etcd member gets OOMKilled:

- **Quorum might be lost** if you lose enough members
- **Leader election** has to happen
- **All Kubernetes API operations stall** (if it's your cluster etcd)
- **Distributed locks get released** unexpectedly

I don't have personal war stories here, thank god. But I've read the post-mortems. Losing etcd quorum is a "page everyone, wake up the VP" kind of incident.

## Batch Jobs: The Silent Killer

I completely overlooked batch jobs in my original post.

For batch processing (ETL jobs, ML training, data migrations):

- **OOMKill = job failure**
- **Job failure = wasted compute resources**
- **Wasted compute = money down the drain**

A batch job that runs for 6 hours and then gets OOMKilled in the final stage? That's 6 hours of compute wasted. If it's a $5/hour GPU instance, you just burned $30 and have to start over.

The "survival" here is about completing the work, not just keeping the process running.

## Sidecars and DaemonSets: The Observability Trap

Here's one I didn't consider: what happens when your monitoring or logging sidecar gets OOMKilled?

Your main application keeps running. It looks healthy. But:
- **Logs stop flowing** to your aggregation system
- **Metrics aren't collected**
- **You have a blind spot**

I saw this happen with a Prometheus node-exporter. It got OOMKilled during a load spike. The application was "healthy" according to Kubernetes, but we had no metrics for 10 minutes. We didn't know there was a problem until the customer called.

## So What Does "Survival" Really Mean?

Let me revise my earlier statement:

**For stateless apps:** Survival means keeping the service available and responsive.

**For stateful apps:** Survival means preventing data loss, maintaining consistency guarantees, and avoiding expensive recovery operations.

**For batch jobs:** Survival means completing the work without wasting compute resources.

**For sidecars:** Survival means maintaining observability and control plane functions.

The memory limit isn't just protecting the process—it's protecting your data, your guarantees, your compute budget, and your visibility.

## Rethinking Resource Allocation by Workload Type

Given this, here's how I approach memory limits now:

### Stateless Services
```yaml
resources:
  requests:
    memory: "512Mi"
  limits:
    memory: "512Mi"  # Match request for Guaranteed QoS
```
Conservative but safe. I can scale horizontally if I need more capacity.

### Databases
```yaml
resources:
  requests:
    memory: "8Gi"
  limits:
    memory: "10Gi"  # Some headroom, but not too much
```
I leave a small gap (20-25%) for burst allocation, but I monitor closely. I want to know if we're approaching the limit so I can scale vertically before an OOMKill happens.

### Message Queues
```yaml
resources:
  requests:
    memory: "16Gi"
  limits:
    memory: "16Gi"  # Hard limit matching request
```
No room for error. I'd rather provision too much memory than risk losing messages.

### Batch Jobs
```yaml
resources:
  requests:
    memory: "4Gi"
  limits:
    memory: "8Gi"  # Generous headroom
```
Batch jobs often have spiky memory usage. I'd rather over-provision than restart a 6-hour job.

### Sidecars (logging, monitoring)
```yaml
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "256Mi"  # Generous headroom for log bursts
```
These need to survive traffic spikes in the main application. Don't let your observability die when you need it most.

## The Real Lesson

What you're surviving *depends on what you're running*.

For a stateless API, survival means the service stays up. For a database, survival means your data stays consistent. For a batch job, survival means the work completes. For a sidecar, survival means you can still see what's happening.

The cost of an OOMKill isn't just the restart time. It's:
- **Data loss** (stateful apps)
- **Consistency violations** (distributed systems)
- **Wasted compute** (batch jobs)
- **Blind spots** (observability)
- **Customer impact** (everything)

So yes, memory limits are about survival. But "survival" means different things for different workloads. Understanding your specific failure modes is what separates a working system from a reliable one.
That's the real difference between CPU and memory limits. CPU throttling gives you time to react. Memory limits give you... well, they give you a restart loop and a pager notification. Plan accordingly.

---

**Further reading:**
- [Quality-of-Service for Memory Resources](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/) - Kubernetes blog on Memory QoS
- [OOM Killer Deep Dive](https://www.kernel.org/doc/gorman/html/understand/understand016.html)
- [Redis Persistence Documentation](https://redis.io/docs/management/persistence/)
- [Kafka Replication and ISR](https://kafka.apache.org/documentation/#replication)
