---
title: "Why Your Pod Died (OOMKilled): The Difference Between CPU and Memory Limits"
date: 2026-03-19
description: "CPU throttles. Memory kills. Understanding this fundamental difference will save you from 3 AM pages."
tags:
  - kubernetes
  - devops
  - memory
  - performance
  - troubleshooting
draft: false
---

I used to think CPU and memory were the same.

Not literally, of course. I knew one was for processing and one was for... well, memory. But when it came to Kubernetes resource limits, I treated them identically. Set a request, set a limit, let the scheduler do its thing. If the app needs more, it uses more, right?

Wrong.

Very, very wrong.

And I learned this lesson at 2 AM on a Tuesday, when our primary API service went from "healthy" to `CrashLoopBackOff` in about 30 seconds. No warning. No graceful degradation. Just... dead. Then alive. Then dead again.

The cause? A memory leak in the application. The trigger? A memory limit that was technically being enforced.

## The Fundamental Difference

Here's what I wish someone had told me when I started with Kubernetes: **CPU is compressible. Memory is not.**

When your container hits its CPU limit, Kubernetes doesn't kill it. Instead, the CFS (Completely Fair Scheduler) starts throttling your process. Your app slows down, but it keeps running. It's like driving with the parking brake on—you're still moving, just slower.

But memory? Memory doesn't throttle. When your container hits its memory limit, the Linux kernel steps in and invokes the OOM (Out of Memory) killer. Your process gets terminated. Immediately. No warning, no gradual slowdown, just SIGKILL.

It's the difference between "you're going too fast, slow down" and "you're out of runway, eject now."

## What Happens When CPU Hits the Limit

Let me walk through what actually happens when your Go service (or Python, or Java, whatever you're running) exceeds its CPU request:

**The Setup:**
```yaml
resources:
  requests:
    cpu: "100m"
  limits:
    cpu: "500m"
```

Your container is happily processing requests, using about 200m of CPU on average. But then traffic spikes. Suddenly your app wants 600m.

Here's the sequence:

1. **Throttling begins**: The CFS scheduler allocates CPU time in 100ms periods. With a 500m limit, your container gets 50ms of CPU time per period.

2. **Your app slows down**: If your process needs 60ms of work in a period, it only gets 50ms. The remaining 10ms? It waits for the next period. Your response times increase.

3. **Metrics show the problem**: Your `container_cpu_cfs_throttled_periods_total` metric starts climbing. You can see it happening.

4. **The app stays alive**: Crucially, your pod doesn't restart. It's slower, but it's still serving traffic.

This is why CPU limits are "safer" in a sense—you get degraded performance, not downtime. It's not ideal, but it's survivable.

## What Happens When Memory Hits the Limit

Now let's look at memory with the same setup:

```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"
```

Your container is using 300Mi, well under the limit. But then that memory leak kicks in. Or a batch job starts processing larger files. Usage climbs: 400Mi, 450Mi, 500Mi...

Here's what happens:

1. **No throttling**: Unlike CPU, there's no mechanism to slow down memory allocation. Your app keeps requesting memory.

2. **The kernel notices**: When your container's memory usage approaches the 512Mi limit, the kernel's memory subsystem flags it.

3. **OOM Killer activates**: Once you hit that limit, the kernel's Out-of-Memory killer evaluates your process. It calculates an "oom_score" based on factors like memory usage and runtime.

4. **SIGKILL**: Your process receives SIGKILL (signal 9), which it cannot catch or ignore. The process terminates immediately.

5. **Kubernetes responds**: The kubelet notices the container exited with code 137 (128 + SIGKILL). It restarts the container if your pod has a restart policy.

6. **CrashLoopBackOff**: If your app immediately tries to allocate the same amount of memory (because, say, it's loading the same dataset), it gets killed again. After a few cycles, Kubernetes puts the pod in `CrashLoopBackOff`.

The worst part? From your application's perspective, this is indistinguishable from a power failure. One moment it's running, the next it's gone. No stack trace. No graceful shutdown. Just... nothing.

## Why This Matters for Your Debugging

Understanding this distinction completely changed how I approach performance issues.

**If you see slow response times** → Check CPU throttling. Look at `container_cpu_cfs_throttled_periods_total`. If it's increasing, you need more CPU requests, not limits.

**If you see restart loops** → Check memory. Look at `container_memory_working_set_bytes` and compare to your limit. If it's consistently near the limit, you're one allocation away from death.

**If you see intermittent failures** → Could be either, but memory is more likely to cause sudden, catastrophic failure. CPU throttling usually shows up as consistent slowness.

## The Mindset Shift

The biggest change for me was internalizing this: **CPU limits are about performance. Memory limits are about survival.**

When you set a CPU limit, you're saying "don't use more than this much CPU." When you set a memory limit, you're saying "if you use more than this, you die."

That changes how you should think about resource allocation:

- CPU requests/limits: Tune for performance and cost
- Memory requests/limits: Tune for stability and reliability

I'd rather over-provision memory and waste some node capacity than under-provision and have my services dying randomly. The cost of unused memory is lower than the cost of 2 AM pages.

---

**Further reading:**
- [Quality-of-Service for Memory Resources](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/) - Kubernetes blog on Memory QoS with cgroup v2
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [OOM Killer Deep Dive](https://www.kernel.org/doc/gorman/html/understand/understand016.html)
