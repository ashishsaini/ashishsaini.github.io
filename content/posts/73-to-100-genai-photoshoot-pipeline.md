---
title: "From 73% to 100%: a GenAI photoshoot pipeline that cleared the catalogue"
date: 2024-11-26T10:15:00+05:30
draft: false
description: "A quarter of the catalogue had no model photo, and the physical photoshoot couldn't catch up. We closed the gap in a month with Stable Diffusion, ComfyUI, and a lot of unglamorous QA."
tags: ["Generative AI", "Diffusion", "Pipelines", "Computer Vision"]
categories: ["Engineering"]
ShowToc: true
TocOpen: false
---

Every e-commerce catalogue has a dirty secret, and it's a number nobody likes to say out loud: the percentage of products that have a proper photo of a real person wearing them. Ours was 73%. Which means more than a quarter of what we sold, thousands of frames, was represented online by a flat shot of the product on a white background, while its neighbour had a crisp image of a model looking great in it. The model shots convert better. Everyone knows this. The problem is that closing the gap means a physical photoshoot, and a physical photoshoot does not scale.

We took catalogue coverage from 73% to 100% in about a month. Not by shooting faster. By not shooting at all for the long tail.

<!--more-->

## Why the bottleneck was real, not just expensive

It's tempting to frame this as a cost problem: model shoots are pricey, generate them instead, save money. The cost was real, but the actual constraint was throughput. New frames land in the catalogue continuously. A studio shoots a finite number of SKUs a day. The backlog doesn't shrink; it's a queue with an arrival rate that beats its service rate, which means it grows forever. You can't hire your way out of an unbounded queue.

So the goal wasn't "cheaper photos." It was "make coverage a solved problem instead of a permanent backlog." That reframing mattered, because it told us what we could and couldn't compromise on. We could tolerate a generated image being *slightly* less perfect than a studio shot. We could not tolerate a pipeline that still needed a human in the loop for every single SKU, because then we'd just have rebuilt the bottleneck with extra steps.

## The pipeline, honestly

The headline tools were Stable Diffusion and ComfyUI, but the model was maybe a third of the work. The other two-thirds was everything around it.

**Input conditioning.** You can't just prompt "a model wearing these glasses" and get the *actual* glasses. The product has to be preserved pixel-faithfully: the brand will not accept a frame that's "inspired by" their product. So the real product image is the anchor, and generation happens around it: we composited the genuine product onto a generated person and scene, using the diffusion model for the parts that need to be invented (the face, the body, the lighting, the environment) while protecting the parts that must not change (the product itself).

**The ComfyUI graph as a product, not a notebook.** ComfyUI is brilliant for exploration and a trap for production if you treat it like a sketchpad. We turned the graph into a parameterized, versioned pipeline: fixed nodes, controlled randomness, and inputs driven by catalogue metadata rather than a human dragging sliders. A new SKU enters as data (product type, color, intended demographic) and comes out the other end as a finished frame without anyone opening the UI.

**Demographic and brand consistency.** A model image isn't neutral. The "right" model for a frame depends on the market and the brand's guidelines. We drove that from metadata too, so a value-line product and a premium sub-brand didn't both get the same generic face. Getting this wrong is the kind of mistake that looks like a small aesthetic slip and is actually a brand-safety incident.

```text
SKU metadata + product image
        │
        ▼
 conditioning  ──▶  diffusion (SD + ComfyUI graph)  ──▶  candidate frames
        │                                                      │
        └────────────── brand / demographic params             ▼
                                                        automated QA gates
                                                               │
                                          pass ──▶ catalogue    fail ──▶ human review
```

## QA was the actual hard part

Here's where most "we used GenAI to make images" stories quietly skip ahead. Generating a *plausible* image is easy now. Generating a *correct* one, ten thousand times, without a human checking each one, is the entire engineering problem.

Diffusion models fail in specific, repeatable ways, and at catalogue scale you will hit every one of them many times a day: the extra finger, the melted product edge, a frame subtly distorted into the wrong shape, a face that wandered into uncanny territory, lighting that doesn't match the product's real material. If even 2% of generated frames have a defect and you're publishing thousands, you're shipping dozens of broken images a day straight to customers. That's not a coverage win; that's a trust loss.

So we built automated QA gates and treated them as first-class. The product region in the output was checked against the original to catch distortion. If the glasses came out warped, the frame was rejected before a human ever saw it. We checked for the common anatomical failure modes. We checked that the generated lighting was consistent enough to look like a real shot. Anything that failed a gate didn't go to the catalogue; it went to a small human review queue.

That review queue is the key to the whole thing. The pipeline didn't have to be perfect. It had to be *good enough to auto-pass the easy majority and route only the genuinely hard cases to a person*. That's what turned an O(n) human bottleneck into an O(hard-cases) one, and the hard cases are a small, shrinking fraction. That's the difference between a demo and a system.

## What closing the gap actually bought

Coverage hit 100% in about a month, and it stayed there, which is the part I care about more than the headline. New SKUs no longer joined a backlog; they joined a pipeline. Photoshoot cost came down, but the durable win was that "products without a model image" stopped being a category that existed.

## The general lesson

I've now built a few of these: image generation at Lenskart, a product-photoshoot engine at Comify that does the same trick for ad creatives. The pattern is always the same, and it's the opposite of where the excitement is.

The generative model is a commodity. It's good, it's getting better, and it's not your moat. Your moat is the conditioning that makes it faithful to a real product, the parameterization that lets it run on data instead of vibes, and, above all, the automated QA that lets you trust the output at a scale no human could review. The model makes the image. The system around the model is what lets you actually publish it.

Everyone wants to talk about the prompt. The prompt was the easy part.
