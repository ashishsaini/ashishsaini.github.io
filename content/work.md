---
title: "Work"
description: "Case studies and demos: AI media automation, Lenskart, MyOperator, and product experiments with the videos to prove they ran."
layout: "single"
---

A walk through the things I've built: what the problem was, what I did, and what it actually moved. Most of the AI/AR work has video, so where I can, I'll just show you.

---

## Comify: CTO & Co-founder

**2025 – 2026.** Intelligent, omnichannel customer-communication infrastructure for large consumer brands.

I co-founded Comify on a simple frustration: every brand spends enormous effort *delivering* messages and almost none deciding whether a message should be sent at all. We built the layer that decided what to say, to whom, and when, then delivered it at scale, cheaply.

- Architected a multi-agent communication platform handling **50M+ messages a day** (push, WhatsApp) for brands including Lenskart and Cars24.
- Designed a serverless backend of **50+ production AWS Lambda functions** to orchestrate and deliver at scale with minimal cost per message.
- Built a **product-photoshoot image & video generation tool** on vision LLMs (Gemini, Claude, Higgsfield) that turns product images into on-brand ad creatives in bulk from style guidelines, cutting catalogue creative cost ~30%.
- Designed **self-learning agents**, including a copywriting agent that researches a brand's domain, applies psychological triggers to generate templates, then continuously optimizes them against live click-through.
- Built a **clickstream recommendation system** (recently-viewed, popular, demography-based) that lifted revenue **60–80%** on targeted workflows versus untargeted sends.
- Hired and lead a cross-functional team across frontend, backend, AI, and data engineering.

---

## Lenskart: AI & AR Research Lead

**2022 – 2025.** Built and led the AI / AR / Data Science team (grew **1 → 12**), partnering directly with co-founders and senior product on roadmap.

I joined to start an AI/AR team from scratch and spent three years turning research demos into features millions of people actually used.

- Delivered best-in-class **AR virtual try-on** across Android, iOS, and web, with a **~9% lift** in online revenue since launch.
- Built **real-time eyeglass removal** on the live AR feed at **12+ FPS on an iPhone 12**, so existing spectacle wearers could see themselves in new frames.
- Launched **contact-lens try-on** for the Aqualens sub-brand, with a **20%+** sales lift in markets like the UAE.
- Architected a **GenAI product-photoshoot pipeline** (Stable Diffusion + ComfyUI) that raised model-photoshoot coverage from **73% → 100% in one month**.
- Rebuilt the **3D try-on asset pipeline** with Blender automation and custom QA apps: 3D coverage **65% → 92%** in three months, asset dev time down ~30%.
- Shipped a **real-time 2D try-on in a single week** that replaced a third-party vendor and saved **~$0.8M/year** in licensing.

{{< youtube eZIoZAU-zQE >}}

*Dynamic True Fit: animated glasses tracking on a live feed.*

{{< youtube cShisn7re7k >}}

*Real-time frame segmentation and removal, running on-device at the edge.*

{{< youtube UBaqPXoiaEo >}}

*AI eyeglass generator: product variations rendered for try-on.*

---

## MyOperator (formerly VoiceTree): Founding Engineer → Director of Technology

**2012 – 2022.** Founding engineer at one of India's fastest-growing cloud-telephony companies; grew from the first customer to **10,000+ paying customers**.

Ten years, two products built from scratch, and the architecture education of a lifetime.

- Owned end-to-end architecture for a high-availability platform spanning **100+ servers at peak**, sustaining a **99.9% uptime SLA**.
- Created **CODAC**, a phone-number-verification product for e-commerce that peaked at **10M+ calls/day** and generated **~$12M a year with a 3-person team**: clients included Lenskart, Snapdeal, and Myntra. Distributed architecture, PHP → Python, MySQL, Redis/Memcached, 10+ telephony servers.
- Designed multi-server, load-balanced, fault-tolerant clusters with custom tooling for **number distribution across servers**.
- Led early cloud-telephony AI research: raw-audio emotion/anger recognition, audio keyword detection, voicebots, TTS/ASR foundations, and neural-net anomaly detection on time-series data.
- Led and grew a **30+ person** engineering org with the **lowest attrition** across departments.

---

## Selected experiments & demos

The stuff I build on weekends. Some of it became real products; most of it just taught me something.

{{< youtube O_tqQRDYB5g >}}

*Digital Twin: fully automated talking-avatar news videos, end to end.*

{{< youtube 5oJH35BNlC8 >}}

*SnapStitch: apparel catalogue photoshoots generated on-model.*

A few more, by link rather than embed:

- **AI news automation**: [heatwave segment](https://www.youtube.com/shorts/FFx2iaX5aWc) · [markets segment](https://www.youtube.com/watch?v=Ue6pEFDcPFc) · [shorts version](https://www.youtube.com/shorts/wr4lj4q9LJc)
- **Apparel virtual try-on**: [demo](https://www.youtube.com/shorts/at7nBtZnUfE)
- **Catalog photoshoots for t-shirts**: [demo](https://www.youtube.com/shorts/7aF3LpNsWhE)
- **Contact-lens try-on**: [demo](https://www.youtube.com/shorts/K9RGf8yzjlE)
- **Real-time frame removal, take two**: [demo](https://www.youtube.com/shorts/ITNHm9CQYCI)

And a few without a camera pointed at them:

- **Videofarm**: an API-driven layout engine that generates video programmatically; the engine under most of the automation above.
- **Agentic development system** built on Claude Code: autonomous, AI-driven software development I use on my own projects.
- **Trained models**: audio anomaly detection, keyword and anger detection in speech, and a custom text-to-speech voice.
- **Home security on a Raspberry Pi**: on-device detection, no cloud round-trip.

---

Want the full history? My [résumé is here](/Ashish_Saini_Resume.pdf), and the [About page](/about/) has the narrative version. Or just [email me](/contact/).
