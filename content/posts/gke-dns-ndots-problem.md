---
title: "The DNS Query That Wouldn't Stop: Debugging GKE's Hidden Ndots Problem"
description: "How a DNS ndots setting in GKE led to cascading DNS failures, and the two ways we finally fixed it."
date: 2025-03-19T14:00:00+07:00
draft: false
tags:
  - kubernetes
  - gke
  - dns
  - infrastructure
  - debugging
showToc: true
categories:
  - Kubernetes
  - Infrastructure
  - Engineering
---

"Why is our DNS resolution so slow?"

I remember staring at that Slack message, coffee going cold, wondering if I'd missed something obvious. We'd been running on Google Kubernetes Engine for months without issues. Then suddenly, DNS lookups were timing out. Services couldn't reach each other. External APIs were failing.

## The Discovery

A teammate noticed intermittent 5xx errors from one of our microservices. "Network issues," they said. "Probably transient."

I wish it had been transient.

I checked the logs. The errors weren't random - they were clustered around DNS resolution failures.

## The Hunt

I started where any good debugging session begins: `kubectl exec` into a pod and start poking around.

```bash
kubectl exec -it my-pod -- /bin/sh
cat /etc/resolv.conf
```

That's when I saw it. The `ndots` value.

```
nameserver 10.0.0.10
search default.svc.cluster.local svc.cluster.local cluster.local google.internal
options ndots:5
```

Five. Five dots.

I knew about ndots - the option that tells the resolver how many dots a domain needs before it's considered "fully qualified." What I didn't know was how GKE was using it to create a DNS query cascade that would eventually bring our cluster to its knees.

## The Root Cause

Here's what happens when a pod in GKE tries to resolve `api-service`:

1. Check if `api-service` has 5+ dots. It doesn't (zero dots).
2. Try `api-service.default.svc.cluster.local` - fail
3. Try `api-service.svc.cluster.local` - fail
4. Try `api-service.cluster.local` - fail
5. Try `api-service.google.internal` - **this is where it breaks**

That last step. That's the killer.

GKE's default configuration includes `google.internal` in the search domains. When your pod tries to resolve an internal service name that doesn't exist in the first three search domains, Kubernetes falls through to querying the Google Compute Engine metadata server for `api-service.google.internal`.

The metadata server at `169.254.169.254`? It's not designed to handle high-volume DNS queries from hundreds of pods. When enough pods are doing this simultaneously - which they will be in any reasonably busy cluster - the metadata server starts dropping requests.

And here's the part that made me want to throw my laptop: this wasn't a new problem. Google's documentation even mentions it. But it's buried in a "concepts" page that nobody reads until they're already on fire.

## Solution 1: Use FQDN Everywhere

The first fix was immediate and surgical: stop relying on the search domain expansion entirely.

Instead of connecting to `api-service`, our services started connecting to `api-service.default.svc.cluster.local.`

Note that trailing dot. That's the secret. It tells the resolver "this is fully qualified, don't append any search domains."

We updated our service discovery to use FQDNs internally:

```yaml
# Instead of this
env:
  - name: API_SERVICE_URL
    value: "http://api-service:8080"

# Do this
env:
  - name: API_SERVICE_URL
    value: "http://api-service.default.svc.cluster.local.:8080"
```

This worked. Instantly. Our DNS query volume dropped by something like 80%.

But there's a catch. Actually, two catches.

**Catch #1:** You need to audit every single internal service call. Every environment variable, every config map, every hardcoded URL. We found references in places I'd forgotten existed. CronJob definitions. Init containers. Sidecars. It took two days of grep-fueled archaeology.

**Catch #2:** FQDNs are great for internal services you control. They're dangerous for external dependencies.

We learned this the hard way when a partner changed their API endpoint from `api.partner.com` to `api-east.partner.com`. Because we were using the FQDN `api.partner.com.` in our configs, the change broke everything. If we'd been using just `api.partner.com` and letting DNS resolve it, the transition would have been transparent.

My recommendation? Use FQDNs for your own services. For external dependencies, accept the risk of extra DNS queries or find another solution.

## Solution 2: NodeLocal DNSCache (With a Caveat)

The second approach we looked at was NodeLocal DNSCache. This runs a DNS caching agent on each node, intercepting DNS queries before they leave the node.

```yaml
# Enable in GKE cluster config
dnsCacheConfig:
  enabled: true
```

At first glance, this seems like it would solve our ndots problem. The DNS queries stay local, get cached, and avoid hitting the upstream DNS server for every request. No more hammering the metadata server, right?

Well, not quite.

Here's what I learned after digging into how NodeLocal DNSCache actually works: **it does NOT prevent the ndots search domain expansion.**

Let me explain why.

NodeLocal DNSCache runs CoreDNS as a DaemonSet on each node, listening on a local IP address (usually something like `169.254.20.10`). Your pod's `/etc/resolv.conf` gets updated to point to this local address instead of the kube-dns service IP.

But - and this is crucial - the search domain expansion happens in **your pod's glibc resolver**, not in the DNS server. When your application queries `api-service`, the resolver in your pod sees `ndots:5`, realizes the name has zero dots, and starts appending search domains:

1. Pod tries `api-service.default.svc.cluster.local` → sends to NodeLocal cache
2. Cache miss → forwards to kube-dns/CoreDNS
3. Pod tries `api-service.svc.cluster.local` → sends to NodeLocal cache
4. Cache miss → forwards to kube-dns/CoreDNS
5. Pod tries `api-service.cluster.local` → sends to NodeLocal cache
6. Cache miss → forwards to kube-dns/CoreDNS
7. Pod tries `api-service.google.internal` → sends to NodeLocal cache
8. Cache miss → forwards to metadata server 💥

**The ndots expansion still happens.** NodeLocal DNSCache just makes each individual query faster by avoiding iptables DNAT rules and conntrack overhead. Plus, subsequent identical queries are served from cache.

So does it help with our metadata server problem? Somewhat. The first time each search variation is tried, it still generates upstream queries. But once those negative responses are cached, subsequent queries for the same name variations are served locally without hitting the metadata server.

The caching helps, but it doesn't eliminate the root cause. The search domain expansion and the resulting query cascade still happen - they're just faster and slightly less painful.

We ended up running both solutions in parallel. FQDNs for critical internal services where we wanted guaranteed single-query resolution. NodeLocal DNSCache as a safety net to reduce the impact of any queries that do slip through.

## Solution 3: Override ndots at the Deployment Level

There's a third approach that doesn't require modifying application code or enabling cluster-wide features. You can override the DNS configuration directly in your Kubernetes deployments.

### The Quick Fix

Add a `dnsConfig` section to your pod spec:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    spec:
      dnsConfig:
        options:
          - name: ndots
            value: "1"
      containers:
        - name: my-service
          image: my-service:latest
```

With `ndots:1`, any hostname with at least one dot is considered fully qualified. This means `api.partner.com` gets resolved immediately without triggering the search domain cascade. Only single-label names like `api-service` (zero dots) will attempt search domain expansion.

Here's what the pod's `/etc/resolv.conf` looks like after this change:

```
nameserver 10.0.0.10
search default.svc.cluster.local svc.cluster.local cluster.local google.internal
options ndots:1
```

Same search domains, but now the resolver is much less aggressive about using them.

### Standardizing with Helm

If you're running multiple services, you don't want to copy-paste this configuration into every deployment. A Helm chart template solves this elegantly.

Create a reusable template in your Helm chart:

```yaml
# templates/_helpers.tpl
{{/* Define standard DNS config */}}
{{- define "mychart.dnsConfig" -}}
dnsConfig:
  options:
    - name: ndots
      value: {{ .Values.dns.ndots | default "1" | quote }}
{{- end -}}
```

Reference it in your deployment template:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      {{- include "mychart.dnsConfig" . | nindent 6 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

With default values in `values.yaml`:

```yaml
# values.yaml
dns:
  ndots: "1"
```

Now every deployment using this chart automatically gets the optimized DNS configuration. If a specific service needs different behavior, you can override it at deploy time:

```bash
helm upgrade my-service ./mychart --set dns.ndots=2
```

### The Trade-offs

Setting `ndots:1` isn't free. You need to understand what you're changing:

**Works great for:** External services with dots in their names (`api.partner.com`, `db.postgres.svc.cluster.local`). These resolve immediately without search domain expansion.

**Requires attention for:** Internal single-label service names. If you reference `api-service` (without namespace), it will try to resolve as-is first. If that fails, it will still fall through the search domains. But since most internal services include at least one dot in their FQDN anyway, this rarely causes issues.

**Doesn't help with:** The fundamental problem of `google.internal` being in the search path. That fourth search domain still exists in `/etc/resolv.conf` - we've only reduced how often we hit it.

Here's the critical point: **this problem only occurs if you use unqualified, single-label service names.** If someone references just `api-service` (zero dots) instead of `api-service.default.svc.cluster.local`, the resolver will still append all search domains including `google.internal`. The metadata server at `169.254.169.254` still gets hammered whenever those unqualified lookups occur.

With `ndots:1`, any hostname containing at least one dot bypasses the search domains entirely. The issue is completely avoided for external APIs (`api.partner.com`), database endpoints (`postgres.database.svc.cluster.local`), or any other dotted hostname. But for bare service names without dots, the problematic search domain cascade still happens. This is why combining `ndots:1` with FQDNs for internal services gives the best protection.

The Helm approach gives you consistency across all deployments. Every pod gets the same DNS behavior, which makes debugging easier and prevents surprises when one service has different resolution behavior than another.

## What I Learned

Three days of my life I'll never get back, but I learned some things:

**Defaults are dangerous.** GKE's default `ndots:5` with that `google.internal` search domain makes sense for Google's infrastructure.

**DNS is "solved" until it isn't.** We treat DNS like plumbing - it's just supposed to work. But Kubernetes DNS is complex. You've got CoreDNS, kube-dns, node-local caches, host resolver configurations, and cloud provider metadata servers all interacting. When it breaks, it breaks in ways that look like "network issues" but are actually configuration problems.

**The metadata server is a single point of failure you didn't know you had.** Every GKE cluster depends on that `169.254.169.254` endpoint for various things. DNS is just the most obvious. If it goes down or gets overwhelmed, weird things happen.

**NodeLocal DNSCache doesn't fix ndots - it just makes it faster.** This was my biggest misconception going in. I thought caching would short-circuit the search domain expansion. It doesn't. The expansion happens at the pod level before the query ever reaches the cache. The cache helps with performance and reduces load on upstream DNS servers, but the fundamental issue of generating multiple queries per resolution remains.

**Debugging DNS requires patience.** The tools are there - `dig`, `nslookup`, `tcpdump`, CoreDNS logs. But DNS issues are often timing-dependent and intermittent. We had to run packet captures for hours to catch the failing queries in the act.

I won't say I'm grateful for the experience. But I am glad I now know to check `ndots` and search domains whenever I see "mysterious" network timeouts in a Kubernetes cluster.

If you're running on GKE, go check your `/etc/resolv.conf` right now. Look at that ndots value. Consider whether you really need five search domains. And maybe - just maybe - start using FQDNs for your internal services before you end up spending three days debugging DNS like I did.
