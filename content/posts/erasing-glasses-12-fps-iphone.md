---
title: "Erasing glasses in real time: 12 FPS on an iPhone 12"
date: 2024-08-07T11:00:00+05:30
draft: false
description: "Existing spectacle wearers couldn't see themselves in new frames during AR try-on. The fix was to remove their glasses from the live camera feed, on-device, at 12+ FPS, with no server in the loop."
tags: ["Computer Vision", "AR", "Edge", "iOS"]
categories: ["Engineering"]
ShowToc: true
TocOpen: false
---

Virtual try-on has a problem nobody puts in the marketing video: it works beautifully for people who don't already wear glasses. For everyone else (which, at an eyewear company, is most of your customers), the experience is broken. You point the camera at your face, the app renders a gorgeous new frame, and it sits *on top of the glasses you're already wearing*. Two pairs of glasses. It looks ridiculous, and worse, it doesn't answer the only question the customer has: do these look good on *me*?

So we built the thing that sounds impossible when you say it out loud: remove the glasses the user is wearing, live, from the camera feed, before rendering the new ones. On the phone. Fast enough that it feels like video, not a slideshow.

<!--more-->

## Why it had to run on the device

The obvious architecture is to stream frames to a server, run a big model, stream them back. We never seriously considered it, for three reasons that I think generalize to almost any real-time CV product:

1. **Latency.** A try-on is a mirror. The moment the reflection lags your movement, the illusion dies. Round-tripping every frame to a server adds a hundred-plus milliseconds you cannot hide.
2. **Cost.** Millions of users, each generating a live video stream you'd have to run inference on, is a GPU bill that turns a feature into a liability. On-device inference costs you nothing per frame after the user has downloaded the model.
3. **Trust.** People are pointing a camera at their own face. "We process your video on our servers" is a sentence you don't want in your privacy policy if you can avoid it.

The catch is that "run it on the device" turns every problem into a budget problem. You have roughly 80 milliseconds per frame to hit a usable frame rate, and an iPhone 12 (a great phone, but a phone) is doing everything else at the same time: the camera pipeline, the AR rendering, the UI. The CV model gets a slice of that, and the slice is small.

## The naive pipeline, and why it was too slow

Removing an object from an image and convincingly filling in what was behind it is, in general, two hard problems stacked on top of each other: segmentation (find every pixel that is "glasses") and inpainting (reconstruct the face, eyes, and skin that the glasses were covering). The textbook approach is a heavy segmentation network followed by a generative inpainting model.

Run that per frame and you get maybe two or three frames per second on a phone. It's a tech demo, not a product. Three FPS feels *worse* than no feature at all, because now the user is watching a juddery, uncanny version of their own face.

Getting from three FPS to twelve-plus wasn't one clever trick. It was a stack of unglamorous ones.

## What actually got it to 12+ FPS

**Stop solving the whole frame every frame.** A face doesn't teleport between frames. We tracked the glasses region across frames and only ran the expensive work inside a tight, predicted bounding box, not the full image. Most of the frame is skin and background that didn't change; spending model budget on it is waste.

**Right-size the model brutally.** The instinct from research is to reach for the most accurate network. The instinct from shipping is to ask what's the *smallest* network that's still convincing. We used a compact segmentation backbone, quantized it, and accepted that it would be slightly worse at edge cases in exchange for being three times faster. On a live feed, the eye barely registers a single imperfect frame; it absolutely registers stutter.

**Inpaint cheaply, not generatively.** Full generative inpainting per frame was the budget killer. For the temple arms and the lenses over skin, a much lighter reconstruction, informed by the symmetric, un-occluded side of the face and a short temporal memory of recent clean frames, was good enough to be invisible at video speed. We saved the heavy generation for the hardest occluded regions, not the whole thing.

**Exploit temporal coherence.** This is the single biggest lever for any real-time CV system and it's the one people coming from image-models forget. Consecutive frames are almost the same frame. You can amortize work across them: run the full segmentation every N frames and *propagate* it on the frames in between using cheap optical-flow-style tracking. The user sees twelve fresh-looking frames a second; the model only did the expensive thing a few times.

**Lean on the hardware.** Core ML and the Neural Engine exist for exactly this. A model that crawls on the CPU flies on the NPU. A meaningful chunk of our speedup was simply making sure every operation in the graph was one the Neural Engine would actually accept, and reworking the few that weren't.

```text
frame N      ──▶ full segmentation + heavy inpaint  (every ~5th frame)
frames N+1…  ──▶ track region + propagate mask + light fill  (cheap)
                       │
                       └─ temporal memory of recent clean pixels
```

## The bugs that taught me the most

The interesting failures were never about accuracy on a benchmark. They were about the gap between a dataset and a human in their kitchen.

Lighting was the first humbling lesson. Models trained on well-lit catalogue-style faces fall apart under a yellow ceiling bulb at night, which is where a lot of people actually shop on their phone. We ended up doing a lightweight per-frame normalization before the model ever saw the pixels, just to drag the input back toward something the network recognized.

Thick frames and reflective lenses were the second. A glossy lens reflecting a window confuses a segmenter into thinking the reflection is part of the face. Temporal smoothing helped here too: a single confused frame gets outvoted by its neighbors.

And motion. People move their heads while evaluating glasses; that's the whole point. Fast motion blurs the temple arms into the hair and the tracker loses the region. The fix was less about the model and more about graceful degradation: when tracking confidence dropped, we'd briefly fall back rather than render something wrong, because a momentary "hold still" beats a glitch on your own face.

## What I'd tell anyone building real-time CV on the edge

The frame budget is the design. Everything follows from "you have ~80ms." Accuracy you can't afford is worse than slightly lower accuracy you can sustain, because stutter breaks the illusion harder than imperfection does. Temporal coherence is free performance and most people leave it on the table. And the device's accelerators are not optional: a model is only as fast as its slowest unsupported op.

We shipped this to existing spectacle wearers across iOS, and for the first time they could actually see themselves in new frames. It was part of an AR try-on program that moved online revenue by around 9%. But the number I remember is twelve. Going from three frames a second to twelve was the difference between a clever research result and something a person could look into and believe.
