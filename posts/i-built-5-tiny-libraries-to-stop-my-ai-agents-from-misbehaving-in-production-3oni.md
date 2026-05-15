> Originally published on [dev.to](https://dev.to/mukundakatta/i-built-5-tiny-libraries-to-stop-my-ai-agents-from-misbehaving-in-production-3oni) on 2026-05-06.
>
> Canonical: https://dev.to/mukundakatta/i-built-5-tiny-libraries-to-stop-my-ai-agents-from-misbehaving-in-production-3oni

# I Built 5 Tiny Libraries to Stop My AI Agents from Misbehaving in Production

I've been building production AI agents for a while now. RAG pipelines, agentic workflows, multi-model routers. I kept hitting the same five problems over and over. Not architectural problems. Plumbing problems.

The kind that don't show up in tutorials but wreck you in production:

- The agent stuffs too much into the context window and the API errors out
- A tool fires an HTTP request to some random domain it hallucinated
- You change a prompt and have no idea if the agent's behavior changed
- The LLM passes malformed args to a tool and it blows up downstream
- The model returns `{ "result": "here is the JSON: {...}" }` instead of clean JSON and your parser chokes

I fixed each of these once, copy-pasted the fix into three other projects, and then got tired of that. So I extracted them into five small libraries. Zero dependencies each. TypeScript-first. Under 300 lines per package.

Here's the stack, in the order you'd use it at runtime:

---

## 1. agentfit - fit messages to the context window

**Problem:** Your conversation history grows until the API throws a `context_length_exceeded` error at the worst possible moment.

**Fix:** Token-aware truncation before the API call.

```ts
import { fitMessages } from "@mukundakatta/agentfit";

const messages = await fitMessages(history, {
  maxTokens: 8000,
  strategy: "drop-middle", // keeps system + recent, drops stale middle
  tokenizer: "cl100k",
});

const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages,
});
```

Strategies: `drop-oldest`, `drop-middle`, `summarize-oldest`. Pluggable tokenizer if you're not on OpenAI's encoding.

[npm](https://www.npmjs.com/package/@mukundakatta/agentfit) · [GitHub](https://github.com/MukundaKatta/agentfit)

---

## 2. agentguard - network egress firewall

**Problem:** Your agent calls a tool that makes an HTTP request. The LLM hallucinates a URL. Your server pings something it shouldn't.

**Fix:** Declare an allowlist. Throw on anything outside it.

```ts
import { createGuard } from "@mukundakatta/agentguard";

const guard = createGuard({
  allow: ["api.openai.com", "your-internal-api.com", "s3.amazonaws.com"],
});

// wrap your fetch
const safeFetch = guard.wrap(fetch);

// now any attempt to hit an unlisted domain throws AgentGuardError
await safeFetch("https://hallucinated-domain.xyz/exfiltrate"); // throws
```

Useful in CI too - run your agent tests with a strict allowlist and catch unexpected egress before it hits prod.

[npm](https://www.npmjs.com/package/@mukundakatta/agentguard) · [GitHub](https://github.com/MukundaKatta/agentguard)

---

## 3. agentsnap - snapshot tests for tool-call traces

**Problem:** You tweak a prompt and have no idea if it changed the agent's tool-call behavior. No diff, no signal, just vibes.

**Fix:** Snapshot the tool-call trace, not just the final output.

```ts
import { snap } from "@mukundakatta/agentsnap";

test("research agent calls search before summarize", async () => {
  const trace = await runMyAgent("summarize recent AI papers");

  await snap(trace, "research-agent-trace");
  // First run: writes the snapshot.
  // Subsequent runs: diffs against it. Fails if tool calls changed.
});
```

Snapshots store the ordered sequence of tool names, arg shapes, and return types, not the raw values (which are flaky). Works with any agent runner: LangGraph, LlamaIndex, raw OpenAI function calls, whatever.

[npm](https://www.npmjs.com/package/@mukundakatta/agentsnap) · [GitHub](https://github.com/MukundaKatta/agentsnap)

---

## 4. agentvet - validate tool args before execution

**Problem:** The LLM calls `send_email` but forgets the `subject` field. Your handler throws a cryptic error. The agent retries blindly.

**Fix:** Validate before execution. Return an LLM-friendly error message so it can self-correct.

```ts
import { vetTool } from "@mukundakatta/agentvet";

const sendEmail = vetTool(
  {
    name: "send_email",
    schema: {
      to: { type: "string", required: true },
      subject: { type: "string", required: true },
      body: { type: "string", required: true },
    },
  },
  async ({ to, subject, body }) => {
    await mailer.send({ to, subject, body });
  }
);

// If the LLM omits `subject`, the call returns:
// "send_email rejected your args: missing required field: subject.
//  Please call again with the corrected arguments."
// - ready to feed straight back into the next turn.
```

[npm](https://www.npmjs.com/package/@mukundakatta/agentvet) · [GitHub](https://github.com/MukundaKatta/agentvet)

---

## 5. agentcast - structured output enforcer

**Problem:** You ask for JSON. You get `Sure! Here's the JSON you asked for: { ... }`. Or worse, truncated JSON. Your parser throws. The agent moves on as if nothing happened.

**Fix:** Validate-and-retry loop with schema enforcement.

```ts
import { cast } from "@mukundakatta/agentcast";

const result = await cast({
  prompt: "Extract the company name and founding year from this text: ...",
  schema: {
    company: { type: "string" },
    founded: { type: "number" },
  },
  llm: async (messages) => openai.chat.completions.create({ ... }),
  maxRetries: 3,
});

// result is guaranteed to match the schema or throw after maxRetries
console.log(result.company, result.founded);
```

On a bad response it builds a corrective prompt automatically and retries. BYO LLM client and BYO validator. The library is the loop, not the dependencies.

[npm](https://www.npmjs.com/package/@mukundakatta/agentcast) · [GitHub](https://github.com/MukundaKatta/agentcast)

---

## How they fit together

At runtime, the order makes sense:

```plaintext
fitMessages   →  trim history before the API call
    ↓
agentguard    →  wrap fetch so tools can't call arbitrary URLs
    ↓
agentvet      →  validate tool args before the handler runs
    ↓
agentcast     →  enforce structured output after the LLM responds
    ↓
agentsnap     →  in tests, snapshot the trace to catch regressions
```

You don't have to use all five. Each is a drop-in. I use all five in my production pipelines and two or three in smaller scripts.

---

## Why not one big framework?

Because one big framework becomes the thing you fight. Each of these solves exactly one problem, has no opinions about the rest of your stack, and disappears when you don't need it.

All five are on npm under `@mukundakatta/`. MIT licensed. PRs open.

If you're building agents in production and hitting these same walls or different walls, I'd like to hear about it in the comments.