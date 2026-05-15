> Originally published on [dev.to](https://dev.to/mukundakatta/i-built-briefbench-a-gemma-4-tool-that-turns-messy-notes-into-model-decisions-3m5p) on 2026-05-07.
>
> Canonical: https://dev.to/mukundakatta/i-built-briefbench-a-gemma-4-tool-that-turns-messy-notes-into-model-decisions-3m5p

# I Built BriefBench: A Gemma 4 Tool That Turns Messy Notes Into Model Decisions

This is my submission for the **Build With Gemma 4** prompt in the DEV Gemma 4 Challenge.

BriefBench is a small local-first web app that helps builders turn rough project notes into a structured decision brief. The goal is not to hide Gemma 4 behind a generic chat interface. The goal is to make the model's work visible: read context, explain the model choice, surface risks, propose a build plan, and suggest a story angle for a technical post.

## What I Built

You paste messy notes about a project: users, constraints, privacy concerns, available hardware, and what you are trying to decide. BriefBench asks Gemma 4 to return strict JSON with:

- summary
- model rationale
- risks
- build plan
- user experience notes
- DEV post angle
- assumptions

The UI then renders those sections as a scannable brief and lets you copy the result as Markdown.

## Demo

Try the app here:

https://gemma-4-briefbench.vercel.app

The public demo runs in demo mode by default so anyone can test the full workflow immediately. If you run it locally with a Gemma 4 API key, the same interface can call Gemma 4 through Google AI Studio or OpenRouter.

## Code

Repository:

https://github.com/MukundaKatta/gemma-4-briefbench

## How I Used Gemma 4

The challenge page asks builders to make a clear case for model selection, so I designed the app around that idea.

Gemma 4 is a good fit because the work is context-heavy but not just raw generation. The model has to read incomplete notes, preserve constraints, reason about tradeoffs, and produce structured output that a user can inspect.

For the prototype, I defaulted to `gemma-4-31b-it` because the 31B dense model is the best fit for richer reasoning over messy planning notes. For production edge scenarios, the app also talks about when Gemma 4 E2B or E4B would be more appropriate, especially for privacy-sensitive workflows running on phones, small devices, or a Raspberry Pi.

## How It Works

The project is intentionally simple:

- a no-framework Node server
- a static HTML/CSS/JS frontend
- one `/api/analyze` endpoint
- provider options for demo mode, Google AI Studio, and OpenRouter
- API keys stay on the server

The system prompt asks Gemma 4 to behave as a local-first project brief analyzer and return strict JSON. The frontend turns that JSON into sections instead of displaying a wall of text.

Demo mode exists so anyone can review the app immediately, even without an API key. With a key configured, the same interface can call Gemma 4 through Google AI Studio or OpenRouter.

## What Gemma 4 Unlocks

The useful part is not that the app can summarize notes. The useful part is that it makes the decision process visible.

For hackathon builders, this matters. It is easy to say "I used an AI model." It is harder, and more valuable, to explain:

- why this model was chosen
- what constraints shaped that choice
- what risks remain
- what the user should do next
- how the model output becomes product UI

BriefBench turns that reasoning into the core product experience.

## What I Would Add Next

The next version would add:

- local Ollama or llama.cpp support for fully offline Gemma 4 runs
- file upload for Markdown, CSV, and PDF notes
- side-by-side model comparison between 31B, 26B MoE, and smaller edge models
- saved briefs for teams evaluating multiple AI project ideas
- export templates for DEV posts, README files, and product specs

## Try It

```bash
npm start
```

Then open:

```text
http://localhost:4174
```

The app runs in demo mode by default. To use Gemma 4 through an API, set `GEMMA_PROVIDER` and the matching API key in `.env`.

## Closing Thought

The best local AI tools should make users feel more capable, not more dependent. BriefBench uses Gemma 4 as a thinking partner for the unglamorous but important middle step: turning rough context into a decision someone can actually act on.