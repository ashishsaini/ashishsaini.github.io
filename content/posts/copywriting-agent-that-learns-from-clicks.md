---
title: "A copywriting agent that learns from clicks"
date: 2026-02-11T08:20:00+05:30
draft: false
description: "Most 'AI copywriting' is a fancy autocomplete that writes once and forgets. We built an agent that researches a brand, writes with intent, ships, and then tunes itself against real click-through: a closed loop, with all the danger that implies."
tags: ["Agents", "LLMs", "GenAI", "Product"]
categories: ["Engineering"]
ShowToc: true
TocOpen: false
---

There's a lot of "AI copywriting" out there and almost all of it does the same thing: you give it a prompt, it gives you five variations, you pick one, the end. It's a better autocomplete. It writes once and forgets everything the moment the message goes out, including, crucially, whether anyone clicked.

The agent we built at Comify is the opposite of forgetful. It researches a brand before it writes a word, generates message templates with actual intent behind the wording, ships them, watches what real people do, and then changes how it writes based on what worked. It's a closed loop. And closing the loop is where it stops being a writing tool and starts being a system, with all the upside and all the ways that can go wrong.

<!--more-->

## Three stages, and only the middle one looks like "AI writing"

**Research.** Before generating anything, the agent builds a model of the brand. What do they sell, to whom, in what register? A premium eyewear brand and a value mobile-recharge service should not sound alike, and a generic LLM left to its own devices will flatten both into the same pleasant, lifeless marketing voice. So the agent does homework first (the brand's domain, its products, its existing tone) and carries that context into everything downstream. This is the unsexy part that determines whether the output sounds like the brand or like ChatGPT wearing the brand's logo.

**Generation with intent.** This is the part that looks like AI copywriting, but the difference is that the agent isn't just writing pleasant sentences. It's applying deliberate psychological triggers (urgency, social proof, curiosity, loss aversion, the well-worn levers of persuasion), and it knows which one it's pulling on each template. That matters enormously for the next stage, because "this message got clicks" is useless feedback, but "messages using a curiosity hook for this audience got clicks" is something you can act on. The intent has to be structured, not vibes, or there's nothing to learn from.

**Optimization.** The templates go out as real messages. Click-through data comes back. The agent uses it to update which approaches it favours, for this brand, for this audience, at this time. A hook that lands for one brand's customers falls flat for another's, and the agent's whole job is to discover that empirically rather than assume it. Over weeks, it converges on what actually works for each specific brand instead of what works in general, which is the only kind of "works" that pays.

```text
brand research ──▶ generate templates ──▶ send
   (context)        (with tagged intent)     │
        ▲                                     ▼
        │                              click-through data
        └──────── update strategy ◀───────────┘
              (which intents win, for whom)
```

## Why "self-learning" is a promise you have to be careful with

A feedback loop that optimizes itself is exactly as dangerous as it is powerful, and anyone who builds one and isn't a little nervous hasn't thought about it hard enough. Here's what we had to design around.

**Clickbait is a local maximum, and the loop will find it.** If the only thing you optimize for is click-through, the agent will happily learn that the highest-clicking message is a misleading one. "Your order has a problem, click here" gets clicks. It also gets unsubscribes, complaints, and a brand that no longer trusts you. A naive optimizer walks straight into this, because the metric it's climbing genuinely does go up. We had to constrain the objective so the agent optimizes within the brand's voice and honesty guardrails, not just for the raw number. The guardrails aren't a nice-to-have bolted on the side; they're part of the objective, or the system optimizes itself into something embarrassing.

**Feedback is noisy and slow, and the agent must not over-fit to it.** Click-through depends on a hundred things that have nothing to do with the copy: time of day, the offer itself, the audience, what else was in the inbox. Treat one campaign's numbers as gospel and the agent learns superstitions. We had to make it update gradually and weight evidence by how much of it there was, so a single lucky send doesn't yank its whole strategy. This is the difference between learning and twitching.

**Exploration costs real money, so you can't explore recklessly.** To learn, the agent has to sometimes try a template it's *not* sure about. That's the explore side of explore-versus-exploit. But every message goes to a real customer of a real brand, so a bad exploratory send has a real cost. We kept exploration deliberate and bounded rather than letting it experiment freely on people who didn't sign up to be a test group.

## The result, and the honest caveat

On targeted workflows, this approach (the agent plus the clickstream recommendation system feeding it audience signal) lifted revenue 60–80% over untargeted sends. That's a big number and I want to be precise about what it means: it's the gap between blasting everyone the same generic message and sending the right intent to the right audience and improving on it over time. A lot of that lift is the targeting; a lot of it is the agent learning the brand. Pulling them apart cleanly is genuinely hard and I'd be lying if I claimed an exact split.

What I'm confident about is the shape of the win. The value isn't in any single clever message. It's in the loop: a system that gets measurably better at a specific brand the longer it runs, because it's learning from that brand's actual customers rather than from a model's general prior about marketing.

## What building it changed in how I think about agents

The hype around agents is mostly about capability: what can it do, how autonomous is it, how many steps can it chain. After building this, I think that's the less interesting axis. The interesting one is the feedback loop: what does the agent learn from, how fast, and what stops it from learning the wrong thing.

An agent without a feedback loop is just an expensive function call. An agent *with* one is a system that changes over time, which means it can get better on its own, and, if you're careless with the objective, worse on its own, confidently, while every metric you're watching goes up. The engineering that matters isn't the generation. It's the loop, the guardrails on the loop, and the humility to assume the loop will find every shortcut you left open.

We built an agent that writes copy. The hard part, and the part I'm proud of, was building one that can be trusted to keep doing it without us watching every word.
