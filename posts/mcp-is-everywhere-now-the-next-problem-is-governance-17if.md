> Originally published on [dev.to](https://dev.to/mukundakatta/mcp-is-everywhere-now-the-next-problem-is-governance-17if) on 2026-05-15.
>
> Canonical: https://dev.to/mukundakatta/mcp-is-everywhere-now-the-next-problem-is-governance-17if

# MCP Is Everywhere Now. The Next Problem Is Governance.

The Model Context Protocol has crossed the awkward line from "interesting integration idea" to "default part of the agent stack."

That is the good news.

The messy news is that the questions developers are asking have changed. Six months ago the question was:

> Can my agent call tools?

Now the question is:

> Which tools should this agent be allowed to call, who approved them, what did they do, and how do I revoke access when something looks wrong?

That shift matters. It means MCP is no longer just a developer convenience. It is becoming an enterprise control surface.

## The Current Pattern

The trend I keep seeing across GitHub, Hacker News, Reddit, and agent framework discussions is this:

- MCP servers are proliferating quickly.
- Browser automation MCPs are becoming default agent tools.
- Internal product data, docs, tickets, CRM data, logs, and code search are being wrapped as agent-accessible tools.
- Teams are starting to ask security questions after the first working demo, not before it.

That last point is the dangerous one.

When tools are easy to add, tool sprawl becomes the new dependency sprawl. An agent with access to five well-scoped tools is manageable. An agent with access to fifty loosely described tools becomes a permissions puzzle with a language model in the middle.

## Why MCP Governance Is Different From API Governance

It is tempting to say, "MCP is just another API layer."

Sometimes that is true. But agent tool use has a few extra wrinkles:

1. The caller may not be a deterministic app. It may be a model deciding dynamically.
2. Tool descriptions become part of the runtime interface.
3. Failure modes include not only bad responses, but bad interpretation.
4. A safe tool in one context may be risky when chained with another tool.
5. Tool output can shape the agent's next action.

That means the governance layer needs more than API keys and logs. It needs intent-aware traces.

## What Teams Should Build Now

If you are shipping MCP servers or agent-accessible APIs, I would prioritize four things.

## 1. Tool Manifests That Humans Can Review

Every tool should have:

- a clear purpose
- permission level
- data classification
- side-effect risk
- owner
- examples of safe and unsafe use

If your tool description is the only documentation, you are already underdocumented.

## 2. Runtime Tool Auditing

Log more than the endpoint call.

Capture:

- agent identity
- user identity
- tool name
- input payload shape
- output size/classification
- parent task
- next action after the tool call

That last piece is subtle but important. The value of a tool call is often only understandable in the context of what the agent did after reading it.

## 3. Scoped Tool Bundles

Do not give every agent every tool.

Create bundles like:

- support-readonly
- support-actionable
- code-review-readonly
- incident-response
- finance-readonly
- browser-sandboxed

This is boring. Boring is good here.

## 4. Kill Switches

You need a way to disable:

- one tool
- one MCP server
- one user-agent pairing
- one class of side-effecting actions

And you need that switch before the incident, not during it.

## The Opportunity

The next wave of useful agent products will not be won only by better models. It will be won by better control planes.

The teams that make MCP safe, observable, revocable, and understandable will have a real advantage. Everyone else will keep discovering that "the agent can use our tools now" is not the same thing as "we are ready for agents."

## Sources Worth Reading

- Hacker News discussion: "MCP is dead; long live MCP"  
  https://news.ycombinator.com/item?id=47380270
- Reddit r/mcp discussion on agentic web readiness  
  https://www.reddit.com/r/mcp/comments/1rv338m/im_convinced_the_agentic_web_is_coming_but_most/
- arXiv: "How are AI agents used? Evidence from 177,000 MCP tools"  
  https://arxiv.org/abs/2603.23802