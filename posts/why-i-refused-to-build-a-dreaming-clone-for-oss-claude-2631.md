> Originally published on [dev.to](https://dev.to/mukundakatta/why-i-refused-to-build-a-dreaming-clone-for-oss-claude-2631) on 2026-05-15.
>
> Canonical: https://dev.to/mukundakatta/why-i-refused-to-build-a-dreaming-clone-for-oss-claude-2631

# Why I refused to build a Dreaming clone for OSS Claude

Anthropic shipped [Dreaming](https://letsdatascience.com/blog/anthropic-dreaming-claude-managed-agents-self-improving-may-6) at Code w/ Claude on May 6. Persistent agent memory as a managed default. A background consolidation pass that turns episodic conversation traces into semantic memory the next session can use.

The OSS reflex is to clone it. By the next weekend somebody will ship `dream-llama`, `dream-qwen`, `dream-mistral`. I considered building one for the local stack I already run. I sat with it for two days and walked away.

This post is about why. And what I built instead.

## What Dreaming actually is

The official description is light on internals, but the public picture is consistent:

1. While the agent is idle, a background process replays the recent session traces.
2. A separate consolidation pass distills those traces into shorter, semantically denser memory artifacts.
3. Those artifacts are stored, indexed, and made available to the next session as retrievable context.

In other words: episodic events go in, a model-driven summarization runs in the background, and what comes out is the kind of "you told me last week that..." continuity people have been wanting since GPT-3.5.

Anthropic owns the storage. Anthropic pays for the consolidation passes. Anthropic ate the eval problem internally and shipped the result. That is the bargain.

## The temptation

Every time a new closed-source primitive lands, OSS people get a small dopamine hit: *I can clone this in a weekend*. Dreaming looks especially clonable. The pieces are familiar. Vector store, scheduled job, summarization model, prompt template that reads consolidated memory back in. None of that is novel.

I have the parts on hand. I ship a [drift detection library](https://github.com/MukundaKatta/ragdrift) and 14 MCP servers. The retrieval, the storage adapters, the embedding scoring, the eval scaffolding. I could glue Dreaming together for any local Llama or Qwen in two evenings.

I drafted the architecture. I started the repo. I deleted it.

## Five reasons I walked away

### 1. Background compute economics do not work on a laptop

Dreaming runs because Anthropic has a fleet of GPUs that are paid for whether your agent dreams or not. The marginal cost of running a 200-token consolidation pass at 3 a.m. across a million users is real but absorbed by infrastructure they already own.

On a local box, the consolidation pass is the most expensive thing the rig does that day. A user running Qwen 2.5 32B on a 4090 is paying full power-draw cost for a background job that produces a summary they may not even read. Multiply by every agent you have. The economics are upside down.

### 2. The consolidator is the model

This is the part that broke me.

Dreaming's quality comes from Claude grading and rewriting Claude's own traces. The consolidation model is the same family as the runtime model. When you replace it with a smaller OSS model to run locally, you get a different summary. Often a worse one. Sometimes one that quietly invents details the original transcript never contained.

So an "OSS Dreaming" with Llama 3.1 8B as the consolidator is not Dreaming. It is a different feature with the same name and lower-quality outputs. Shipping it under the same label is misleading.

The honest version is: *here is a memory consolidation system, the quality depends on the consolidator, here is the eval to pick one*. That is a useful library. It is not Dreaming.

### 3. There is no public eval

When someone ships a feature like this, the first question every careful operator asks is: how do I know it is working? For Dreaming, Anthropic has internal benchmarks (the [Outcomes feature](https://www.mindstudio.ai/blog/code-with-claude-2026-new-agent-features) reports +10.1% on PowerPoint quality). They have not released the consolidation-quality eval, and they probably will not.

So an OSS clone has no anchor. You cannot say "this clone retains 85% of the source quality" because there is no source quality number. You ship vibes.

This is fine for a hobby project. It is not fine for something I would want a stranger to install and trust with their agent's memory.

### 4. Memory you cannot un-bake is a compliance trap

Dreaming consolidates traces into derived artifacts. Once those artifacts are baked, undoing them is hard. If a user says "delete that conversation" three weeks later, the consolidated memory may already encode the gist of what they wanted forgotten.

Anthropic gets to handle this in their privacy and retention policy. They publish a story, they take the legal exposure. An OSS library shipping the same pattern dumps that exposure on every developer who installs it. Most will not notice until a deletion request lands.

The mitigation is real (track provenance per memory, support cascading deletes, version the consolidator) but it is the most expensive part of the system to build correctly. None of the weekend-clone projects will ship it.

### 5. The pull model is almost as good and far simpler

Once I priced out the push model (background consolidation), I tried the pull model: do nothing in the background, summarize on demand when the next session starts and asks for context.

Pull is worse on latency. The first message of a new session pays a 200ms-2s tax for the on-demand summary. But pull is dramatically cheaper, fully reversible (no derived artifacts to un-bake), and you can show the user exactly what was retrieved before you send it.

For a local-first OSS user, pull is the right shape. The performance hit is the cost of doing memory honestly when you are not Anthropic.

## What I am building instead

A small library called `agentmemory` (working name; not shipped yet). Three pieces:

1. **Time-bucketed episodic store**. Append-only event log of agent interactions, embedded at write time, indexed by session and timestamp. No consolidation. Retention policy is per-event. Deletion is a real delete.

2. **On-demand summarizer**. When a new session starts and the prompt budget allows, retrieve the top-N relevant events for the current intent and summarize them inline using the same model the runtime uses. Latency lives in the cold-start, not in the background. The summary is shown in the trace, never silently injected.

3. **`ragdrift` integration**. Watch the retrieval distribution over time. If yesterday's "remember when we discussed X" stops returning anything because the user's intent has drifted, surface that as a drift signal instead of pretending the memory is healthy.

It is less magical than Dreaming. It is also auditable, reversible, and runs on a 16GB MacBook without scheduling background passes that wake the fans.

## Closing

If someone ships a careful, evaluated, deletion-aware OSS Dreaming clone, I will use it. I am not against the feature. I am against shipping the easy 70% of it under the same name and pretending the missing 30% is a polish issue.

The Anthropic version is good because Anthropic ate the hard parts. The OSS version of the same idea has to either eat those parts honestly, or ship under a different name with a different promise.

Until one of those happens, I will run the boring pull-model version, watch it with `ragdrift`, and let people who want the magical version pay the managed-platform price for it.

If you are about to spin up `dream-yourmodel-here` next weekend, three questions worth answering before you publish:

1. What is your eval for "did the consolidation preserve the right thing?"
2. What happens when a user asks you to delete a memory three weeks after consolidation?
3. Does your consolidator quality match your runtime quality, or are you silently degrading the agent every night?

If those have honest answers, ship it. If not, the world has enough thin libraries already.

---

*Personal project, not affiliated with my employer. Tools mentioned: [`ragdrift`](https://github.com/MukundaKatta/ragdrift), [`mcp-stack`](https://github.com/MukundaKatta/mcp-stack), and the [`@mukundakatta/agent*`](https://www.npmjs.com/~mukundakatta) reliability stack.*
