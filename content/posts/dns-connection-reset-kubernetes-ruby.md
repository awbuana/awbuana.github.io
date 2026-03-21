---
title: "The DNS Mystery: Five Factors and a Ruby Gem"
date: 2026-03-21
description: "A deep dive into debugging DNS connection resets in a Ruby application running on Kubernetes, uncovering the surprising interaction between conntrack, Alpine Linux, libc, and Ruby's DNS resolution."
tags:
  - kubernetes
  - dns
  - ruby
  - debugging
  - infrastructure
categories:
  - Kubernetes
  - Infrastructure
  - Engineering
---

"Connection reset by peer."

If you've run applications in Kubernetes long enough, you've probably seen this error. Usually it's a service that went away, a network hiccup, something transient. You retry, it works, you move on.

But what if it keeps happening? What if it only happens with DNS lookups, and only sometimes, and only in production?

That's where I found myself a few weeks ago.

## The Event

Our Ruby application started throwing intermittent `Errno::ECONNRESET` errors. Not on HTTP requests to external APIs — on DNS lookups. The stack trace pointed to `getaddrinfo`, the standard libc function for resolving hostnames.

The weird part? It wasn't consistent. Maybe 1 in 50 requests would fail. The failures clustered during high-traffic periods. And they only happened in our production Kubernetes cluster, not in staging, not locally with Docker Compose.

DNS is supposed to be boring. It's supposed to just work. When DNS becomes interesting, you know you're in for a long week.

## The Timeline

**Day 1: It's got to be the network**

I started with the obvious. Maybe our DNS servers were overloaded? We were using the default CoreDNS setup in Kubernetes. I checked the metrics — CPU and memory were fine. No errors in the CoreDNS logs.

I ran `kubectl exec` into a failing pod and tried `nslookup` manually. Worked every time. Of course it did. The problem was intermittent.

**Day 2: Alpine Linux enters the chat**

Our Docker images were based on Alpine Linux. Small, secure, everyone's using it. But I remembered reading something about Alpine and DNS issues.

Alpine uses musl libc instead of glibc. Musl is smaller, simpler, more correct — but different. I started digging into how musl handles DNS resolution.

Here's what I learned: musl's resolver queries all nameservers in `/etc/resolv.conf` **in parallel**, not sequentially like glibc. It takes the first response that comes back. This is actually great for performance, but it means more UDP packets flying around.

Also, until musl 1.2.4, it didn't support TCP fallback for DNS. If a UDP response was truncated (larger than 512 bytes), tough luck. But that wasn't our issue — we weren't getting truncated responses, we were getting connection resets.

**Day 3: The conntrack revelation**

I started searching for "kubernetes dns connection reset" and found it. The infamous conntrack race condition.

Here's what happens: Linux uses a connection tracking table (conntrack) to manage stateful connections. When you do a DNS lookup, your application sends a UDP packet to port 53. The kernel creates a conntrack entry for this "connection" (even though UDP is connectionless).

Now here's the race: if two DNS queries from the same source IP and port go to the same destination IP and port at nearly the same time, they might both try to use the same conntrack entry. The second one can trigger a "race" where the kernel drops the packet or sends a reset.

Why was this happening to us? A few factors converged:

1. **High pod density** — Our nodes were running many pods, each making DNS queries
2. **Ruby's threading model** — We were using multiple threads, each potentially doing DNS lookups concurrently
3. **Parallel A/AAAA queries** — Modern resolvers query for both IPv4 (A) and IPv6 (AAAA) records simultaneously
4. **Musl's parallel nameserver queries** — Hitting all three nameservers at once meant more packets
5. **NAT + conntrack** — Kubernetes networking involves NAT, which relies on conntrack

The result? A perfect storm where UDP DNS packets were getting dropped or reset at the kernel level.

**Day 4: Why Ruby? Why us?**

But wait — other applications in our cluster weren't having this problem. Why just Ruby?

I started looking at how Ruby resolves DNS. By default, Ruby uses libc's `getaddrinfo`. That's the same function that `ping`, `curl`, and most other tools use. So why was Ruby failing when other tools worked?

The answer was in the timing and volume. Ruby applications often do many DNS lookups in rapid succession — for database connections, Redis, external APIs, background job queues. Our app was resolving the same hostnames over and over, not caching results.

Also, Ruby's `getaddrinfo` calls are blocking. When you have multiple threads doing blocking DNS lookups, they can all hit the resolver at once during traffic spikes.

**Day 5: The Resolv discovery**

I remembered something about a pure Ruby DNS resolver. It's in the standard library: `resolv`.

```ruby
require 'resolv'

# Instead of Socket.getaddrinfo or Resolv.getaddress
Resolv::DNS.new.getaddress("example.com")
```

The key difference: Ruby's `Resolv` doesn't use libc at all. It implements the DNS protocol directly in Ruby. It sends UDP packets manually, handles timeouts and retries itself, and most importantly — **it doesn't use `getaddrinfo`**.

This means it bypasses the conntrack path entirely? Not exactly. It still sends UDP packets, so conntrack is still involved. But `Resolv` has different timing characteristics:

1. It queries A and AAAA records **sequentially** by default, not in parallel
2. It implements its own retry logic with exponential backoff
3. It randomizes the source port for each query (more on this in a moment)
4. It can fall back to TCP if UDP fails

I also discovered `resolv-replace.rb`, which monkey-patches Ruby's socket classes to use `Resolv` instead of `getaddrinfo` globally:

```ruby
require 'resolv-replace'
```

We tried it. The connection resets stopped.

## Root Cause

Let me break down what was actually happening:

**The libc approach (default Ruby behavior):**

When Ruby calls `getaddrinfo("example.com", "80", ...)` with `AF_UNSPEC` (the default), libc does the following:

1. Reads `/etc/resolv.conf` for nameservers
2. Queries all nameservers for A records (IPv4)
3. Queries all nameservers for AAAA records (IPv6) — often in parallel
4. Combines results and returns them

With musl (Alpine), step 2 and 3 happen in parallel across all configured nameservers. So with 3 nameservers, you might have 6 UDP packets going out nearly simultaneously.

These packets often use the same source port (or ports that hash to the same conntrack bucket). When they hit the kernel's connection tracking table at just the wrong time, the race condition triggers. The kernel drops the packet or sends a RST, and Ruby sees `ECONNRESET`.

**Why `Resolv` worked:**

Ruby's `Resolv::DNS` does things differently:

```ruby
# From the Resolv source - each_address method
def each_address(name)
  if use_ipv6?
    each_resource(name, Resource::IN::AAAA) {|resource| yield resource.address}
  end
  each_resource(name, Resource::IN::A) {|resource| yield resource.address}
end
```

It queries AAAA first (if IPv6 is enabled), then A. Not in parallel. Less packet volume. Less chance of hitting the race.

Also, `Resolv` uses randomized source ports:

```ruby
# Resolv creates a new UDP socket and binds to port 0 (random ephemeral port)
sock = UDPSocket.new(af)
DNS.bind_random_port(sock, bind_host)
```

Each DNS query uses a different source port, which means different conntrack entries. The race condition requires the same source IP+port and destination IP+port. Randomizing the source port makes this collision much less likely.

## A Parallel Story: Go and the AAAA Record Problem

While debugging our Ruby issues, I remembered something a colleague mentioned about their Go services. They had a similar problem, but with a twist — it involved AAAA records.

**The scenario:** They were integrating with a partner bank's API. The bank's DNS infrastructure had a quirk: it would return `SERVFAIL` for AAAA (IPv6) queries, but A (IPv4) records worked fine.

Here's what happened with their Go application:

**With CGO enabled (default):**
```go
// Go uses getaddrinfo via CGO by default
resp, err := http.Get("https://api.partner-bank.com/")
// err: lookup api.partner-bank.com: getaddrinfow: ...
```

The lookup failed completely. Even though A records were present and valid, glibc's `getaddrinfo` — which Go calls when CGO is enabled — failed the entire query because the AAAA lookup returned `SERVFAIL`.

**With CGO disabled:**
```bash
CGO_ENABLED=0 go build
```

The same code worked perfectly.

**Why?**

When Go compiles **with CGO enabled**, it uses the system's libc resolver (`getaddrinfo`) for DNS lookups. This makes Go subject to all the same libc behaviors we discussed earlier — including glibc's strict handling of AAAA failures.

When Go compiles **with CGO disabled**, it uses Go's pure Go DNS resolver. This resolver is more lenient:

1. It queries A and AAAA records separately
2. If AAAA fails (even with SERVFAIL), it doesn't fail the entire lookup
3. It falls back to A records and continues
4. It handles the DNS protocol directly, similar to Ruby's `Resolv`

**The glibc behavior:**

When you call `getaddrinfo("api.partner-bank.com", "443", ...)` with `AF_UNSPEC` (which asks for any address family), glibc:

1. Sends parallel queries for both A and AAAA records
2. Waits for both to complete
3. **Fails the entire call if either query fails** [man7.org/linux/man-pages/man3/getaddrinfo.3.html]

This is standards-compliant behavior per [RFC 2553](https://www.rfc-editor.org/rfc/rfc2553) — if you ask for IPv6 and the DNS server says "I can't answer that," libc treats that as an error. But it's not always what you want in practice.

**Go's workaround:**

Go lets you force the pure Go resolver at runtime without recompiling:

```bash
GODEBUG=netdns=go ./myapp
```

Or force CGO resolver:
```bash
GODEBUG=netdns=cgo ./myapp
```

**The connection to our Ruby problem:**

Both issues stem from the same root: libc's `getaddrinfo` is the default, and it has strict, sometimes surprising behavior. Whether you're using Ruby or Go, when you rely on libc for DNS:

- You inherit all its quirks
- AAAA failures can break IPv4 connectivity
- UDP race conditions in conntrack affect you
- You're at the mercy of `/etc/resolv.conf` and nameserver behavior

The fix is the same: bypass libc. Ruby has `resolv`, Go has its pure Go resolver. Both implement DNS directly and handle edge cases more gracefully.

---

## The Fix

We made three changes:

**1. Switched to `resolv-replace`**

In our application startup:

```ruby
require 'resolv-replace'
```

This globally replaces libc DNS resolution with Ruby's pure-Ruby resolver. It's a one-line change that fixed the immediate problem.

**2. Migrated from Alpine to Debian slim**

We moved our Docker base images from `ruby:3.2-alpine` to `ruby:3.2-slim`. This gave us glibc instead of musl, which has slightly different DNS behavior — it queries nameservers sequentially by default, not in parallel.

This wasn't strictly necessary after adding `resolv-replace`, but it removed one variable from the equation. Alpine's musl has other quirks that can cause DNS issues (like the lack of TCP fallback in older versions).

**3. Added NodeLocal DNSCache**

We deployed [NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) to our cluster. This runs a DNS cache on each node as a DaemonSet. Pods send DNS queries to the local cache (127.0.0.1:53), which then forwards to CoreDNS.

This helps in two ways:
- Reduces load on CoreDNS
- The local cache uses TCP for upstream queries when needed, avoiding UDP conntrack issues

**4. DNS caching in the application**

We added a simple memoization cache for DNS results within our application process. This reduced the total number of DNS lookups significantly.

```ruby
class CachedResolver
  def initialize
    @cache = {}
    @resolver = Resolv::DNS.new
  end

  def resolve(hostname)
    @cache.fetch(hostname) do
      @cache[hostname] = @resolver.getaddress(hostname)
    end
  end
end
```

## Lessons Learned

**DNS is more complex than it looks**

What started as "connection reset by peer" turned into a multi-day investigation touching:
- Linux kernel connection tracking
- Differences between glibc and musl
- Ruby's DNS resolution internals
- Kubernetes networking
- UDP vs TCP DNS behavior

**The abstraction leaks**

We think of DNS as "just call `getaddrinfo` and get an IP." But that function hides a lot of complexity — and that complexity can leak. When it does, you need to understand what's actually happening under the hood.

**Intermittent problems are the hardest**

If this had been a consistent failure, we would have found it faster. The intermittent nature — 1 in 50 requests, only during high load — made it hard to reproduce and debug. We needed to understand the race condition to understand why it was intermittent.

**One line can save you**

`require 'resolv-replace'` — that's all it took to fix the immediate problem. But finding that one line required understanding why the default behavior was failing. There's no substitute for digging deep.

For Go, it's `CGO_ENABLED=0` or `GODEBUG=netdns=go`. Same principle: bypass the system resolver.

**Consider your base image carefully**

Alpine is great for many use cases. Small, secure, fast to pull. But musl's different behavior around DNS resolution bit us hard. For applications that do a lot of DNS lookups or have strict reliability requirements, glibc-based images might be worth the extra size.

**This isn't just a Ruby problem**

Go, Python, Node.js — any language that can call `getaddrinfo` can hit these issues. The Go AAAA/SERVFAIL problem shows that libc's strictness affects everyone. If you're having mysterious DNS failures, check whether your language is using libc or a native resolver.
