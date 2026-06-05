---
title: "Fifty million messages a day on fifty Lambda functions"
date: 2025-09-30T18:45:00+05:30
draft: false
description: "Building Comify's communication backend serverless wasn't about avoiding ops. It was about making cost-per-message a first-class design constraint, and serverless is the only architecture that bills the way the business actually works."
tags: ["Serverless", "AWS", "Architecture", "Cost"]
categories: ["Engineering"]
ShowToc: true
TocOpen: false
---

When people hear that Comify's backend is fifty-some AWS Lambda functions moving more than fifty million messages a day, the usual reaction is "isn't serverless expensive at that scale?" It's the right question with the wrong assumption baked in. Serverless got *chosen* because of the scale and the economics, not in spite of them. But you only get the good version if you design for it deliberately. The naive version is genuinely a trap.

<!--more-->

## The shape of the problem

Communication traffic is spiky in a way that punishes always-on infrastructure. A brand fires a campaign and ten million push notifications need to go out in a few minutes. Then, for hours, almost nothing. Then a WhatsApp flow for a different brand. Then a quiet overnight stretch. Then a morning surge.

If you provision servers for the peak, you're paying for a fleet that sits at 5% utilization most of the day. If you provision for the average, you fall over the moment a real campaign launches, which is the one moment that matters, because a campaign that goes out late is worse than one that doesn't go out at all. Autoscaling groups help, but they scale in minutes and the spikes happen in seconds, so you're perpetually either over-provisioned or behind the wave.

This traffic shape is *exactly* what serverless is for. You pay for invocations, the platform absorbs the spike, and when nothing's happening you pay nothing. The cost curve follows the business: we pay per message because the platform charges us per unit of work, and we charge per message too. That alignment is the whole reason the architecture works.

## Why fifty functions and not five services

There's a fair critique of "Lambda everything": you can end up with a sprawl of tiny functions that's impossible to reason about: a distributed monolith with worse tooling. We pushed back on that by being deliberate about boundaries. The fifty-odd functions aren't fifty microservices; they're a smaller number of *pipelines*, each decomposed into stages that have genuinely different scaling and failure characteristics.

The decision rule was simple: a stage becomes its own function when it scales differently or fails differently from its neighbours. Audience resolution (read-heavy, bursty, cacheable) is not the same workload as channel delivery (I/O-bound, rate-limited by external providers) which is not the same as click ingestion (high-volume, write-heavy, must never block a send). Splitting those means each can scale to its own shape, and a slow downstream provider can't back-pressure into the part of the system that's deciding *who* to message.

```text
campaign trigger
      │
      ▼
 audience resolution ─▶ queue ─▶ content / personalization ─▶ queue ─▶ delivery
      (read-heavy)                  (agentic, LLM-backed)              (rate-limited)
                                                                          │
 click / event ingestion ◀──────────────────────────────────────────────┘
      (write-heavy, async)
```

The queues between stages are not decoration. They're the shock absorbers. A campaign dumps ten million people into the front of the pipeline; the queue holds them while delivery drains at whatever rate the downstream providers (WhatsApp's API, the push gateways) will actually accept. The queue depth becomes your natural backpressure and your natural retry buffer at the same time.

## Where serverless will bite you, and what we did about it

I don't want to sell this as free. Serverless at this volume has sharp edges, and pretending otherwise is how people end up with a surprise bill and a 3am page.

**Cost is invocations × duration × memory, and duration is where you bleed.** A function that's slow because it's waiting on a network call is paying for the wait. We spent real effort making sure functions did work, not waiting: pushing I/O-bound waits into queue-driven steps rather than holding a function open while a slow API responded. A 200ms function and a 2-second function that do the same useful work cost 10x apart, and at fifty million invocations that gap is the difference between a sustainable product and a dead one.

**The downstream rate limits are the real ceiling.** You can invoke Lambda almost arbitrarily fast. You cannot send WhatsApp messages arbitrarily fast: the provider has limits, and blowing through them gets you throttled or blocked, which is catastrophic for a communication company. So the delivery stage is deliberately *not* trying to go as fast as Lambda can. It's pacing itself to the provider's limits, with the queue absorbing the difference between how fast we *could* go and how fast we're *allowed* to.

**Cold starts matter at the edges, not the middle.** During a campaign, everything's warm. The cold-start tax shows up on the low-traffic, latency-sensitive paths. We kept those functions lean and reached for provisioned concurrency only where a cold start would actually be felt by a user, not as a blanket policy, because provisioned concurrency is just renting an always-on server again, and if you sprinkle it everywhere you've thrown away the entire reason you went serverless.

**Idempotency is non-negotiable.** Retries are a feature of every queue and every Lambda, which means every message-affecting operation will, eventually, run twice. Sending a customer the same push notification twice because a retry fired is a real, visible failure. Every stage that could cause a duplicate send is built to be safe to re-run.

## The part people underrate: the bill is observability

Because we pay per invocation and per millisecond, the AWS bill is a remarkably honest profiler. A function that quietly got slower shows up as a line item that quietly got bigger. A pipeline stage that's retrying too much shows up as invocation count drifting away from message count. We watch cost-per-message the way a different team might watch p99 latency, because for this business they're nearly the same signal. A regression in efficiency *is* a regression, and the billing dashboard catches it.

## Would I do it again

For this workload, without hesitation. The traffic is spiky, the unit economics demand that cost track usage, and the team is small enough that not running a server fleet is a genuine multiplier. If the traffic were flat and predictable and huge, I'd reconsider. At constant high utilization, reserved instances win on raw cost, and serverless stops paying for its premium. Architecture is a response to a workload, not a religion.

But the lesson that transfers regardless of platform is this: at scale, cost-per-unit-of-work is not a finance concern you bolt on later. It's a design constraint you put on the whiteboard on day one, next to latency and reliability. We can move fifty million messages a day on fifty functions because we treated the bill as a feature. Most of the architecture decisions that look clever in hindsight were really just us refusing to pay for work we weren't doing.
