> Originally published on [dev.to](https://dev.to/mukundakatta/i-got-burned-by-prompt-injection-in-production-here-are-2-tiny-npm-libs-that-stopped-it-438i) on 2026-05-06.
>
> Canonical: https://dev.to/mukundakatta/i-got-burned-by-prompt-injection-in-production-here-are-2-tiny-npm-libs-that-stopped-it-438i

# I Got Burned by Prompt Injection in Production. Here Are 2 Tiny npm Libs That Stopped It.

A user pasted a help article into our agent. Three minutes later the agent silently rewrote a customer email, leaked an internal URL, and tried to fetch a `.zip` from a domain none of us had ever seen.

Nothing in the LLM was wrong. The problem was upstream. Retrieved text walked into the prompt with no inspection, and the agent treated it as gospel.

I wrote up the lessons as a short preprint. The two npm libs below are the working code behind it.

## The two libs

### `@mukundakatta/prompt-injection-shield`

A small-rule scanner for prompt-injection patterns in untrusted text. No heuristics, no ML, no weights. Just regex-grade rules with a typed `risk_reasons` array so you can log, gate, or strip lines.

```bash
npm install @mukundakatta/prompt-injection-shield
```

```js
import { scan } from '@mukundakatta/prompt-injection-shield';

const r = scan(retrievedDoc);
if (r.risk_score > 0) {
  console.warn('blocked:', r.risk_reasons);
  return;
}
```

What it catches:

- "ignore previous instructions" and family
- system-prompt impersonation
- tool-call hijack patterns
- url-based exfil hints
- secret patterns the model should not see

When a rule fires, you get the line, the rule id, and a recommendation. Strip, redact, drop, or feed it to your audit trail. Up to you.

### `@mukundakatta/vector-poison-score`

Same idea, retrieval side. Score chunks before they go into context.

```bash
npm install @mukundakatta/vector-poison-score
```

```js
import { score } from '@mukundakatta/vector-poison-score';

const s = score(chunk);
if (s.poison_score >= 0.5) skip(chunk);
```

What it scores:

- oversized chunks (token bloat attacks)
- secret-exfiltration patterns inside retrieved text
- suspicious link clusters
- mixed-language anomalies in technical docs

Weights are tunable. Defaults are conservative. Both libs have **zero runtime dependencies**.

## Why "small rules"

Big ML defenses are expensive, opaque, and hard to audit when something slips. Small rules are the opposite. You can read them. You can grep them. You can fork the file when your threat model is different from mine.

Same logic as a linter. Not perfect. Not sexy. Catches a huge chunk of the dumb stuff before the model has to think about it.

## Where they sit in the pipeline

```plaintext
retrieval -> [vector-poison-score] -> reranker
                                      |
                                      v
              tool output -> [prompt-injection-shield] -> prompt
                                                          |
                                                          v
                                                         LLM
```

Two checkpoints. Cheap. Easy to disable per request. No effect on latency above the noise floor.

## The preprint

Full writeup with threat model, rule design, and limitations:

- Zenodo DOI: [10.5281/zenodo.20057056](https://doi.org/10.5281/zenodo.20057056)
- Figshare DOI: [10.6084/m9.figshare.32193543](https://doi.org/10.6084/m9.figshare.32193543)
- GitHub bundle: [MukundaKatta/rag-guardrails-paper](https://github.com/MukundaKatta/rag-guardrails-paper)

License is CC BY 4.0 on the paper, MIT on the code. Both libs are tiny. Both are forkable in five minutes.

## What this is not

Not a replacement for a full security review. Not a benchmark claim. Not a model. The whole thesis is that an inspectable, boring baseline between retrieval and prompt construction is worth more than nothing, and most teams ship with nothing.

If you build agentic RAG, drop these in front of your prompt. Then run a real audit later.
