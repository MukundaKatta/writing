> Originally published on [dev.to](https://dev.to/mukundakatta/i-shipped-14-mcp-servers-this-week-gemma-4-changes-which-ones-matter-1klk) on 2026-05-11.
>
> Canonical: https://dev.to/mukundakatta/i-shipped-14-mcp-servers-this-week-gemma-4-changes-which-ones-matter-1klk

# I shipped 14 MCP servers this week. Gemma 4 changes which ones matter.

This week I shipped a workspace called [`mcp-stack`](https://github.com/MukundaKatta/mcp-stack): fourteen small MCP servers for everyday agent work. Ten of them are reliable transforms that LLMs reach for tools instead of imagining (CSV parsing, regex, JMESPath, diff, SQL formatting, shell escaping, JSON5, TOML/YAML/JSON, IANA timezones, HTML to Markdown). Four are RAG and agent helpers (drift diagnosis, citation markers, retrieval IR metrics, prompt budget).

Then [Gemma 4 dropped](https://blog.google/technology/developers/gemma-4/) and I had to re-rank the toolbox.

Local AI is having a moment, and Gemma 4 sharpens the question every agent author is now quietly asking: which utility tools matter when the model is small enough to run on a Pixel?

## The cloud-model assumption I had been making

I built mcp-stack with a model like Claude or GPT-4o in mind. Big context window. Server-side. Network round-trip per tool call. In that world, the cost of a tool call is the network hop, not the inference. So a tool that exists to *save the model from doing something it can do but does badly* is a clear win, even if the tool's input/output is small.

Example: `regex-test`. The LLM can technically reason about a regex pattern. It does it confidently and wrong about a third of the time. Routing through the actual JS engine costs roughly 5ms and removes a class of bugs. Worth it.

Gemma 4 changes that calculus in two ways.

## Change one: the prompt-budget cliff is closer

Cloud models have been racing context windows up: 128K, 200K, 1M. You can stuff a lot in the prompt and let attention sort it out.

The Gemma 4 family is **2B, 4B, 31B dense, and a 26B MoE**. The 2B and 4B variants exist specifically so the model can run on a phone, a Raspberry Pi, or a browser. They have a 128K context window, which is generous, but the practical context for a multi-turn agent on a phone is much smaller because every token is RAM you actually paid for.

This makes my [`promptbudget-mcp`](https://www.npmjs.com/package/@mukundakatta/promptbudget-mcp) noticeably more important. The three tools in that package are:

```typescript
count_tokens(text, max_tokens?)        // pre-flight: does this fit?
truncate_to_token_budget(text, ...)    // make it fit, lose info
chunk_to_budget(text, max_tokens, ...) // split into N under-budget pieces
```

On a cloud model, `count_tokens` was a "you might want to know" tool. On Gemma 4 running on a Pixel, it becomes a routine pre-flight. Every prompt assembly should ask "does this fit before I send it" because the cost of overflowing is not a 429, it is the model thrashing or quietly dropping context. When inference is server-side and metered, you sometimes find out from the bill. When inference is on-device, you find out from a thermal throttle.

## Change two: tools the cloud model could compensate for, the small model cannot

This is the real surprise.

When you're running a 200B-parameter model, you can lean on it to clean up messy structured data. Pass it ragged JSON and it will sort it out. Pass it a poorly-formatted SQL query and it will normalize it. The model is so capable that the *transform* tools feel optional.

When you're running a 2B-parameter Gemma, that compensation budget is gone. The same messy JSON either parses cleanly or it does not. The same SQL is either valid or breaks the next tool downstream. So the transform tools that felt optional move into the "wire this in by default" bucket.

Concretely, on a Gemma 4 stack I would now wire in by default:

- [`json5-mcp`](https://www.npmjs.com/package/@mukundakatta/json5-mcp): parse the JSON-with-comments the model sometimes emits, round-trip to strict JSON before the next tool sees it.
- [`csv-tools-mcp`](https://www.npmjs.com/package/@mukundakatta/csv-tools-mcp): the small model is more likely to mishandle quoted commas; route through a real parser.
- [`shellquote-mcp`](https://www.npmjs.com/package/@mukundakatta/shellquote-mcp): the small model is more likely to forget how to escape `$VAR`; the safety value goes up when the model has fewer "remember the rule" parameters.
- [`regex-test-mcp`](https://www.npmjs.com/package/@mukundakatta/regex-test-mcp): same logic, harder for the small model to reason about lookahead semantics.

## Change three: tools that were obvious wins look smaller

A few of my MCP servers were strong picks for cloud-model agents but I would think harder about including them in a Gemma 4 stack:

- [`jmespath-mcp`](https://www.npmjs.com/package/@mukundakatta/jmespath-mcp): deep JSON queries. On a 128K-context cloud model, useful. On an on-device model, the query language itself is what the model has to learn to use, and that's a load you might not want to pay.
- [`html-to-markdown-mcp`](https://www.npmjs.com/package/@mukundakatta/html-to-markdown-mcp): for a cloud model, an obvious "scrape then read" tool. For an on-device model, you might just stream the relevant text directly to the model without the tool intermediary.

The point isn't that these tools are wrong. The point is that "useful" is a function of the model on the other side of the stdio pipe.

## Picking your Gemma 4

Gemma 4 ships in three flavors. My read for an agent author:

- **2B / 4B**: the "tool belt + small brain" architecture. The MCP servers do the heavy lifting; the model orchestrates. Lean on transform-heavy tooling (csv-tools, regex-test, json5). Skip query-language tools (jmespath, sqlfmt) where the model would have to learn the query syntax to use the tool.
- **31B dense**: the "server but private" sweet spot. Closest to a cloud model in capability, runs on a workstation. All fourteen MCP servers earn their slot here.
- **26B MoE**: high throughput, advanced reasoning. Best paired with multi-step agent loops where the cost saving over a cloud API matters at volume.

## Try the stack

If you want to try the fourteen MCP servers with whatever Gemma 4 setup you have:

```jsonc
// In your MCP client config (Claude Desktop / Cursor / Cline / Windsurf / Zed)
{
  "mcpServers": {
    "promptbudget":   { "command": "npx", "args": ["-y", "@mukundakatta/promptbudget-mcp"] },
    "csv-tools":      { "command": "npx", "args": ["-y", "@mukundakatta/csv-tools-mcp"] },
    "regex-test":     { "command": "npx", "args": ["-y", "@mukundakatta/regex-test-mcp"] },
    "json5":          { "command": "npx", "args": ["-y", "@mukundakatta/json5-mcp"] },
    "shellquote":     { "command": "npx", "args": ["-y", "@mukundakatta/shellquote-mcp"] },
    "diff":           { "command": "npx", "args": ["-y", "@mukundakatta/diff-mcp"] },
    "sqlfmt":         { "command": "npx", "args": ["-y", "@mukundakatta/sqlfmt-mcp"] },
    "jmespath":       { "command": "npx", "args": ["-y", "@mukundakatta/jmespath-mcp"] },
    "toml-yaml-json": { "command": "npx", "args": ["-y", "@mukundakatta/toml-yaml-json-mcp"] },
    "timezone":       { "command": "npx", "args": ["-y", "@mukundakatta/timezone-mcp"] },
    "html-to-md":     { "command": "npx", "args": ["-y", "@mukundakatta/html-to-markdown-mcp"] },
    "ragdrift":       { "command": "npx", "args": ["-y", "@mukundakatta/ragdrift-mcp"] },
    "ragmetric":      { "command": "npx", "args": ["-y", "@mukundakatta/ragmetric-mcp"] },
    "citecite":       { "command": "npx", "args": ["-y", "@mukundakatta/citecite-mcp"] }
  }
}
```

All fourteen are listed in the [official MCP Registry](https://registry.modelcontextprotocol.io/v0/servers?search=io.github.MukundaKatta) and installable via `npx -y` from any MCP-compatible client.

## What I would build for the Build track

I picked the Write track for this challenge because the angle felt cleaner. If I were doing the Build track, the project I would scope is this:

**An on-device RAG sentry that runs locally**. Gemma 4 (2B or 4B) as the planning brain, Ollama or local inference as the runner, my [`ragdrift`](https://crates.io/crates/ragdrift) Rust crate for drift detection, and my [`ragmetric-mcp`](https://www.npmjs.com/package/@mukundakatta/ragmetric-mcp) for retrieval evaluation. The pitch: a small model can monitor a bigger RAG system's quality continuously without round-tripping to a cloud API.

I'm leaving that for someone else, or for a follow-up post if there's interest.

## Honest takeaways

1. **Tool selection is a function of the model.** A toolbox optimized for GPT-4o is not the right toolbox for Gemma 2B. The "obvious wins" change.
2. **Transform tools become more important on small models.** The model has less spare capacity to compensate for messy inputs.
3. **Query-language tools become less obviously useful on small models.** The model now has to learn two query languages (yours plus its own).
4. **Pre-flight tools become routine.** When you cannot just throw tokens at the problem, you have to know how many tokens are coming.

If you're building agents and Gemma 4 is on your radar, look at the toolbox you have and re-rank it. The fourteen MCP servers I shipped this week are open source under MIT or Apache-2.0; pull what helps and skip the rest.

---

*Personal project, not affiliated with my employer. Submitting to the [Gemma 4 Challenge](https://dev.to/challenges/google-gemma-2026-05-06) sponsored by Google AI.*
