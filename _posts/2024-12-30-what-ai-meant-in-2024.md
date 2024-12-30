---
layout: post
title: "What AI Meant in 2024"
date: 2024-12-30 03:47:50 -0500
toc: true
categories: ai
---

I find myself giving a version of these definitions semi-regularly now to people outside of "the space," and don't really check them against other definitions from inside "the space." It will be fun to see what AI means in 2025 and beyond. (Not that all of the below is from 2024 only; some of these definitions basically go back to ~2018.)

### AGI / ASI

ASI seems to have appeared because of retreats from the over-burdened AGI...

Nobody agrees what AGI or ASI means, but [between Microsoft and OpenAI, it at least means $100 billion **profit**](https://www.theverge.com/2024/12/26/24329618/openai-microsoft-and-the-100-billion-agi-question).

With either, the cultural imaginary is still vaguely focused on the singularity. The more nearterm concerns about intelligence _augmentation_, which seem orders of magnitude more likely on short timelines, are not discussed enough to have SEO keywords.

I'll try one I guess:

### AAI / ACE

> "Artifically augmented intelligence"

(Or if I was writing a novel, perhaps "ACE" for "Artificial Cognitive Enhancement"...)

Just as Google Maps enabled Uber to exist by augmenting regular drivers with ["the knowledge,"](https://en.wikipedia.org/wiki/The_Knowledge_(film)) so must other industries be disrupted by making data analysis, programming, form filling, and other tasks require only specification (not technique). Some rumblings that BPO (business process outsourcing) will fall first. Apparently that's at least [38 billion in **revenue**](https://www.asiapacific.ca/publication/ai-disrupting-leading-philippine-industry-and-creating) if you count the Philippines. (But not _profit_: subtract inference costs.)

### AI

"AI" still currently means "Generative AI" which means "LLM".

### Generative AI / LLM

"Generative AI" means we're using machine learning not to regress or classify (ie. score stuff or bucket stuff), but to generate text, video, images, etc.

An LLM, a "large language model," was the breakthrough tech that allowed text generation that passes the Turing test. It's a deep neural network which utilizes attention "pre-trained" on ~the Internet; it predicts output tokens given input tokens.

### LLM (again)

An LLM is an interpolative database of the Internet.

You can not deterministically edit the entries inside this database ([yet](https://arxiv.org/abs/2202.05262)), but in trade, you can interpolate between the entries. You can imagine something like a massive stack of spreadsheets where we can even find values between the values in the cells.

It is not simply a memorization of the Internet, but can produce text that never appeared before. It jointly learns the database rows and a mixture of operations it can perform to "look" at the incoming tokens and what it stored in "the database" in order to predict the next most likely token.

The "stochastic parrot" meme seems mostly quiet now, but there's still some glib truth to calling it "auto-complete." One does begin to self-reflect though, the more and more powerful the auto-complete becomes.

But can they _reason?_ This question turned recently into "can autoregressive solve ARC?" [Yeah,](https://arcprize.org/blog/oai-o3-pub-breakthrough) probably so, if you want to pay that cloud bill. That trendline is extremely exciting to cloud providers though, because it really does mean "turn $ into intelligence."

Still, spend some time with a human child and then ask GPT-4o to use the pythonize crate (as of 2024-12-29 it can't so it will answer over and over confidently proclaiming here's how to use it without actually using it), and you'll still be impressed by humanity.

### Agents

An agent is i) an LLM we let use the Internet via API calls or a browser to take actions for us, ii) a multi-modal LLM that we let use a computer to further extend what actions it can take, iii) our only way of parallelizing LLMs today.

Success rates are low and as you must multiply those probabilities together per action, nobody is using agents for serious work yet.

### Agents (again)

An LLM can have a context window, which means it carries some state. And it can even do "in context learning," which lets it do tasks it didn't specifically learn in pre-training from that context. At various points you may want to fork or reset the state. Agents are the dominant frame for doing that - for parallelizing LLMs today.

That means we are still basically in a single-threaded world.

So things are very early...

### [bonus] "Pre-training" and "post-training"

An oddity of current language: nobody _trains_ LLMs! (Maybe this is why, briefly, some very funny posts about the "scandal" of [training on the train set](https://x.com/mikeknoop/status/1870583471892226343) for ARC-AGI went viral...)

We now only pre-train to create a "foundation model" and then we post-train via "fine tuning" for domain-specific tasks.
