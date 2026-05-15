> Originally published on [dev.to](https://dev.to/mukundakatta/six-reliability-primitives-for-llm-agents-m13) on 2026-05-08.
>
> Canonical: https://dev.to/mukundakatta/six-reliability-primitives-for-llm-agents-m13

# Six Reliability Primitives for LLM Agents

Reliability concerns for LLM agents are typically bundled into one heavy framework that asks you to adopt prompting, tool routing, and runtime governance as a single dependency. Production teams want them à la carte. They want small primitives they can drop in around existing tool calls without buying into a new programming model.

That observation is the design centre of **agent-stack**: six small, single-concern reliability libraries published independently to **npm**, **PyPI**, and the **Model Context Protocol** registry. Each library is zero-dependency, under 500 lines of code, and addresses one specific failure mode that production agent teams have to handle.

This post is a tour of the six primitives, the cross-cutting invariants they enforce, and the trade-offs of "composable by inclusion" instead of "composable by framework."

## The six primitives

| Library | Concern | Failure mode it addresses |
| --- | --- | --- |
| **AgentFit** | Context-window fitting | Token-aware truncation. Pluggable tokenizers for OpenAI / Anthropic / open models. |
| **AgentGuard** | Network egress allowlist | Blocks "agent suddenly POSTs PHI/secrets to attacker.com" at the network layer. |
| **AgentSnap** | Snapshot tests for tool calls | Catches silent regressions when a model's tool-call shape changes after a deploy. |
| **AgentVet** | Tool-arg validation | Throws `ToolArgError` with an LLM-friendly retry hint, so the next turn can self-correct. |
| **AgentCast** | Structured-output retry | BYO-LLM JSON validator + retry loop for malformed responses during brown-outs. |
| **AgentBudget** | Token + dollar caps | Prevents runaway loops billing $1000 on a single user query. |

## Three runtimes per primitive

Every library ships in three runtime forms:

```bash
# TypeScript (npm)
npm i @mukundakatta/agentvet @mukundakatta/agentguard @mukundakatta/agentbudget

# Python (PyPI)
pip install agentvet agentguard agentbudget
```

```json
// MCP server (Claude Desktop config)
{
  "mcpServers": {
    "agentvet": { "command": "npx", "args": ["-y", "@mukundakatta/agentvet-mcp"] },
    "agentguard": { "command": "npx", "args": ["-y", "@mukundakatta/agentguard-mcp"] }
  }
}
```

The Python and TypeScript surfaces are 1:1 by design — a Python team and a TypeScript team can interoperate on the same primitives. The MCP variant lets a remote LLM use the primitive as a tool: ask AgentVet to validate a tool call before executing it, ask AgentGuard to check an outbound URL before sending the request.

## Composable by inclusion, not by framework

Every primitive has the same shape: a single class or function as the public surface, a typed error carrying retry-friendly context, and an opt-in automatic adapter for popular provider response shapes. Nothing depends on anything else. You can use AgentBudget without AgentFit. You can use AgentVet's MCP variant without ever touching the npm package.

Compare with framework-style libraries that bundle reliability concerns: you get them all or none, you adopt their orchestration model, you migrate to their abstractions. agent-stack inverts that.

## Why this matters in practice

Healthcare amplifies every reliability concern. A FHIR-querying agent has to be defensive about: only calling sanctioned endpoints, never leaking PHI in logs, abstaining when the right tool is not on the list, validating that a `patient_id` looks like a real FHIR id before it hits the tool, never exceeding a budget on a single patient query, and producing structured outputs the downstream system can actually parse. Existing agent frameworks address one or two of those concerns. agent-stack gives you all six in libraries you adopt independently.

The same pattern applies to any production setting where the cost of an agent silently failing is higher than the cost of a slightly-too-defensive primitive: financial services, internal corporate tools, customer support automation, anything billable.

## Artifact paper

The full design rationale, the cross-cutting invariants, and the operational questions that emerge when reliability is split across many small dependencies are documented in a peer-reviewable artifact paper:

- **Zenodo DOI:** [10.5281/zenodo.20074702](https://doi.org/10.5281/zenodo.20074702)
- **HuggingFace Hub:** [huggingface.co/mukunda1729/agent-stack](https://huggingface.co/mukunda1729/agent-stack) (HF DOI [10.57967/hf/8720](https://doi.org/10.57967/hf/8720))
- **Source:** [github.com/MukundaKatta/agent-stack](https://github.com/MukundaKatta/agent-stack)

The paper is currently under review at the **ASE 2026 Tools track**. Source for every library is archived in **Software Heritage**. Every package has CI on GitHub Actions, snapshot tests, and a `CITATION.cff`.

## What's next

The natural next primitive is **AgentTrace** for cost and latency telemetry. After that, a combined `@mukundakatta/agent-stack` meta-package that imports all six with sensible defaults for production agents. The healthcare-aware AgentGuard preset (curated allowlist of FHIR endpoints, PHI redaction policies) is a natural specialization.

If you ship LLM agents in production and you want one of the primitives without buying into a framework, install only the library you need. Each one stands alone.

Source on GitHub. Paper on Zenodo. DOI on HuggingFace. Six primitives, three runtimes, one unified surface for production reliability.