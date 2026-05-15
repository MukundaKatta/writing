> Originally published on [dev.to](https://dev.to/mukundakatta/22-oss-packages-in-24-hours-what-each-one-earned-its-slot-503) on 2026-05-11.
>
> Canonical: https://dev.to/mukundakatta/22-oss-packages-in-24-hours-what-each-one-earned-its-slot-503

# 22 OSS packages in 24 hours: what each one earned its slot

This week I shipped twenty-two open source packages in roughly twenty-four hours. All MIT or Apache-2.0. All publicly installable. Distributed across three package registries (crates.io, PyPI, npm) and one Model Context Protocol registry.

Counting it up:

- 7 Rust crates on crates.io
- 1 Python wheel on PyPI (PyO3-bound to a Rust core)
- 14 Node MCP servers on npm, all listed in the official MCP Registry

If that sounds suspicious, it should. Shipping volume is easy. Shipping volume without padding your portfolio with bloat is the harder problem. This post is about how I drew that line and what I refused to ship even when it was tempting.

## The principle: every package earns its slot

The rule I held myself to was this: for each package, before publishing, I had to be able to answer "what specific problem does this solve that the alternative does not?" If the answer was vague or amounted to "well, an LLM can do this but maybe not perfectly," I left it on the floor.

Concrete examples of things I shipped:

**`csv-tools-mcp`** earned its slot because LLMs reliably mishandle CSV edge cases (embedded commas inside quoted fields, escaped double quotes, BOMs, CRLF vs LF). Routing through state-machine parsing avoids the failure mode. The next tool downstream gets data it can trust.

**`shellquote-mcp`** earned its slot because LLMs constantly get shell escaping wrong. They confidently emit `bash -c "echo $HOME"` and the variable interpolates instead of staying literal. The right escape is non-obvious enough that even experienced devs forget the bash single-quote dance (`'\''`). A tool that returns shell-correct output every time has a real audience.

**`ragdrift`** (Rust crate plus PyPI wheel) earned its slot because the existing options for production RAG drift detection are either too heavy (Arize Phoenix), too tabular-focused (Evidently), or only a slice (the Wasserstein papers without an integration layer). Five-dimensional drift in one library, with the math in Rust, fills a real gap.

## What I refused to ship

The list of MCP servers I refused is more interesting than the list I shipped:

- `base64-mcp`: any LLM does base64 in two tokens of inference. Adding tool overhead is anti-value.
- `uuid-mcp`: same.
- `slugify-mcp`: same.
- `hash-mcp`: same.
- `markdown-table-mcp`: real but tiny audience. Less than 1% of agent sessions invoke this.
- `embedrank-mcp`: I shipped the Rust crate `embedrank` but refused the MCP wrapper. LLMs don't have raw embedding vectors at hand mid-conversation. The tool would never get called.

Each of these would have padded my count from 14 to 20 in mcp-stack. None of them would have made a real user's life better. Some would have made the registry browsing experience worse for everyone (more thin packages to scroll past).

This is the part where Hacktoberfest writing matters most. Open source contribution is celebrated by volume, but volume optimized for the wrong metric (number of repos, number of stars, number of packages) leads to ecosystem garbage. The number that matters is what fraction of your shipped work would a stranger genuinely install given a clear problem.

## The Rust + Python hybrid: where the math actually goes

Of the 22 packages, the most opinionated piece was `ragdrift`. The thinking:

The drift detection literature is in Python, the production retrieval systems are in Python, but the math (MMD with RBF kernels, sliced Wasserstein-1, KS, PSI, k-means cluster assignment shift) is the kind of thing that benefits from being native code. So I built the core in Rust (`ragdrift-core` on crates.io), wrapped it with PyO3 + maturin so it ships as a Python wheel (`ragdrift-py` on PyPI), and gave the Python side a thin facade plus adapters for the storage backends most people actually use (OpenSearch, pgvector, Pinecone) and exporters for the metric backends most people actually use (CloudWatch, Prometheus, Datadog).

Python users `pip install ragdrift-py` and never touch the Rust toolchain. Rust users `cargo add ragdrift` and never touch Python. The math sits in one place and stays consistent across both.

This pattern (numerical core in Rust, ergonomic facade in Python) is increasingly the right answer for ML and data tooling. The Rust ecosystem has matured to the point where `ndarray`, `rayon`, and PyO3 + maturin make the wrapping trivially worth it.

## The MCP angle

Of the 14 MCP servers in `mcp-stack`, the four I'd actually wire into my own Claude Desktop config tomorrow are:

- `promptbudget-mcp`: token-budget pre-flight, truncation, and chunking. The "do I even need to send this?" check.
- `csv-tools-mcp`: CSV the LLM cannot accidentally break.
- `shellquote-mcp`: shell escaping the LLM cannot accidentally break.
- `regex-test-mcp`: regex semantics the LLM cannot hallucinate.

Notice the pattern. The MCP servers that earn permanent install slots are the ones where the cost of getting it wrong is high enough that you'd rather pay a tool-call round trip than trust the model.

The other ten are real but situational. `sqlfmt-mcp` only matters if you're constantly handling SQL. `timezone-mcp` only matters if your agent does scheduling. `html-to-markdown-mcp` only matters if you scrape pages. Each is the right tool for a real workflow but not part of the universal default config.

## Why I'm writing this for Hacktoberfest

This post lives under the Hacktoberfest writing tag because the original 2025 challenge was about reflecting on open source contribution while the experience was still fresh. The submission window has closed, and this is not an entry for prize judging. But the writing prompt holds: the act of explaining why each package shipped, what each earned its slot, and what got refused is itself a useful practice.

If you're a maintainer reading this, I'd encourage the same discipline. The pressure to ship and tag and link is real. The pressure to refuse to ship is harder to talk about and probably the more valuable discipline.

The 22 packages I shipped this week are listed at:

- crates.io: https://crates.io/users/MukundaKatta
- PyPI: https://pypi.org/user/mukundakatta/
- npm: https://www.npmjs.com/~mukundakatta
- MCP Registry: https://registry.modelcontextprotocol.io/v0/servers?search=io.github.MukundaKatta

Pull what helps. Skip the rest.

---

*Personal project, not affiliated with my employer. Tagging this for the [2025 Hacktoberfest Writing Challenge](https://dev.to/challenges/hacktoberfest), whose submission window has closed. Posting under the tag because the topic genuinely fits the challenge's prompt about reflecting on open source contribution.*
