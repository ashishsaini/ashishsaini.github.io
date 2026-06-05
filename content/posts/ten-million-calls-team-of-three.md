---
title: "Ten million calls a day with a team of three"
date: 2024-03-18T09:30:00+05:30
draft: false
description: "How a phone-number-verification product called CODAC scaled to 10M+ calls a day on a handful of telephony servers, and the boring tooling that made it possible."
tags: ["Distributed Systems", "Telephony", "Scaling", "Architecture"]
categories: ["Engineering"]
ShowToc: true
TocOpen: false
---

There's a version of a scaling story that's all dashboards and Kubernetes. This isn't that one. This is about a phone-number-verification product called CODAC that, at its peak, placed more than ten million calls a day, made around $12M a year, and was run by three people. We did it on a cluster of telephony servers that mostly looked like pets, not cattle, and the hardest problem we solved had nothing to do with the volume.

<!--more-->

## What CODAC actually did

The premise was unglamorous and lucrative. An e-commerce company (Lenskart, Snapdeal, Myntra, take your pick) wants to confirm that the phone number a customer typed at checkout is real and reachable before they ship a cash-on-delivery order to it. The cheapest way to do that, in India, at that time, was a "missed call" flow: the system places a call, the user's phone rings, they don't even have to pick up, and the act of the call connecting (or the user calling a number back) verifies the line.

Multiply that by every COD order across several of the country's biggest retailers and you get a firehose. Tens of millions of call attempts a day, each of which has to be placed, tracked, retried on failure, and reported back to the client's order system within seconds, because a customer is standing on the checkout page waiting.

## The first architecture was wrong, and that was fine

We started in PHP. I want to be honest about that because there's a temptation, years later, to pretend you reached for the perfect tool on day one. We didn't. PHP was what we knew, it got the first version in front of paying customers fast, and "it works and bills" beats "it's elegant and theoretical" every single time at a startup.

PHP held longer than you'd think. What eventually pushed us off it wasn't request throughput. It was the long-running, stateful nature of managing call legs and retries. A call isn't a request-response. It's a little state machine that lives for thirty seconds, can fail in six different ways, and needs to be reconciled against what the telephony hardware actually did. We ported the core to Python, kept MySQL as the system of record, and leaned hard on Redis and Memcached for the hot path: the per-number, per-second state that you cannot afford to hit the database for.

The telephony itself ran across ten-plus servers, each one wired to carrier trunks. From the outside it was one product. From the inside it was a small fleet, and fleets have a specific failure mode that took me a while to fully respect.

## The actual hard problem: distributing numbers across servers

Here's the part nobody warns you about. When you have ten telephony servers and a river of numbers to call, *which server calls which number* turns out to be the whole game.

Naively, you round-robin. Number comes in, hand it to the next server. This breaks in ways that are invisible until they're catastrophic:

- A carrier rate-limits *per trunk*, and trunks are tied to servers. Round-robin a hot batch of numbers from one client onto a server whose trunk is already near its ceiling and that whole batch fails, not because the system is overloaded, but because you put the wrong work in the wrong place.
- Retries have to remember where they came from. If number X failed on server 3 because of a carrier issue specific to server 3's route, retrying it on server 3 is the dumbest possible choice.
- Servers die. When server 3 goes down mid-batch, ten thousand in-flight numbers need to go *somewhere*, immediately, without double-dialing the ones that already connected.

So we wrote a distribution layer. Custom, boring, and the single most important piece of software in the product. It tracked per-server, per-trunk capacity in real time, kept a short memory of where each number had already been tried, and made placement decisions that balanced load while respecting the physical reality of the carrier routes. When a server fell over, its outstanding work drained to healthy peers without replaying anything that had already completed.

It was, in effect, a purpose-built scheduler for a resource (carrier trunk capacity) that you can't autoscale because it's a contract with a phone company. You can't spin up more trunk on a Tuesday. The whole architecture had to be designed around the fact that the scarce resource was fixed and lumpy.

```text
incoming numbers ──▶ distribution layer ──▶ server pool
                          │                   ├─ srv1 (trunk A, 78% util)
                          │                   ├─ srv2 (trunk B, 41% util)
                  per-trunk capacity          ├─ srv3  ✗ draining
                  + recent-attempt memory      └─ ...
```

That diagram is the entire trick. Everything else (the calling, the reporting, the billing) was comparatively easy.

## Why three people could run it

People hear "10M calls a day, three engineers" and assume heroics. It was almost the opposite. The team was small *because* the system was designed to not need babysitting, and it was designed that way because the team was small. The constraint and the architecture fed each other.

A few decisions that bought us our weekends back:

- **The database was the source of truth, and nothing else was allowed to be.** Caches were disposable. Any server could be rebuilt from MySQL plus a cold start. That meant a dead server was a non-event, not an incident.
- **Idempotency everywhere on the call path.** Placing the same call twice is worse than not placing it: you annoy a customer and you pay the carrier. Every operation that touched a number was safe to retry, which is what let the distribution layer be aggressive about reassignment.
- **We monitored the carrier, not just the servers.** Most of our real outages originated outside our walls: a route degrading, a trunk flapping. The alerts that mattered watched connection rates per route, so we found out before the client did.
- **We said no to features that would have added state.** Every fancy capability someone wanted usually meant another thing to reconcile when a server died. The discipline of a tiny team is that you feel the cost of complexity immediately.

## What I took with me

I've since built GenAI pipelines and agent fleets, and the lessons from CODAC keep showing up wearing different clothes. The scarce, lumpy resource that you have to schedule around isn't carrier trunk anymore: it's GPU, or an API rate limit, or a model's context window. The principle is identical: find the thing you can't just autoscale your way out of, and design the whole system around respecting it.

The other thing CODAC taught me is that revenue-per-engineer is a real engineering metric, not just a finance one. Three people made $12M a year not because we worked harder than everyone else, but because we spent our complexity budget on the one problem that mattered and ruthlessly avoided spending it anywhere else.

That's still the job. The river just carries different water now.
