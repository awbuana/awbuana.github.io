---
title: "About Me"
description: "About awbuana"
draft: false
hidemeta: false
showToc: false
searchHidden: true
---

# Hello, I'm awbuana

I keep a note in my phone called "things that broke today."

I work at a unicorn startup. The kind where you go from "we need to scale" to "why is the latency spiking" in about six months. Over time I've touched most of the stack, but I keep getting pulled back to the same problems: slow queries on billion-row tables, Kubernetes clusters that cost too much, and services that need to handle thousands of requests per second without falling over.

## What I Actually Do

I spend a lot of time with Go. I've written services that handle serious traffic, and I've debugged enough distributed systems (and distributed problems).

The cloud bill is everyone's problem now. I got into cost optimization after seeing our database spend double in three months for no good reason. Turns out you can save a lot by actually looking at what queries are running and whether you need that instance size. I've cut database costs by tuning queries, adding proper indexing, and knowing when to say "this doesn't need to be in Postgres." I've also spent time right-sizing Kubernetes clusters.

## On-Call Lessons

I do SRE work, which means being professionally paranoid. Monitoring, alerts, incident response. I learned the hard way that an alert that fires 50 times a day isn't an alert anymore—it's just noise. I also learned that if you can't reproduce it locally, you're probably looking at a race condition, or data inconsistency, or both.

There's always more broken than you can fix. A hundred things are technically wrong. You can maybe fix ten. The rest you document, mentally bookmark as "this will bite us later," and keep moving. That part doesn't get talked about enough.

## Why Write This

Honestly? I need to remember what I learned before it evaporates. But also, I spent too many nights Googling "postgresql slow query" or "postgresql hit max connection pool" and finding answers from 2014. Maybe my notes help someone else.

These are real production problems, real money saved, real services kept alive. Nothing theoretical. Just the reality of keeping systems running when the scale gets uncomfortable.

## Connect

If you're dealing with similar things—high-traffic Go services, painful cloud bills, database optimization, or just want to commiserate about on-call—hit me up. I'm around.
