# z3rno-docs

> The public-facing product documentation site for Z3rno. Built with [Mintlify](https://mintlify.com/). Hosted at `https://z3rno.dev/docs` (once configured in Week 6).

**License:** Apache 2.0
**Status:** Skeleton (full content lands in Phase 1 / Week 6)
**Part of:** [Z3rno](https://github.com/the-ai-project-co) — the database for AI agent memory

## What this repo holds

- `mint.json` — Mintlify site configuration (navigation, colours, logo, topbar)
- `introduction.mdx` — Landing page
- `quickstart.mdx` — Getting started guide
- `concepts/` — Core concepts (memory types, lifecycle, multi-tenancy, temporal versioning)
- `sdks/` — SDK reference (Python, TypeScript)
- `integrations/` — Framework integrations (LangChain, CrewAI, OpenAI Agents, Anthropic MCP)
- `self-hosting/` — Docker Compose and Kubernetes deployment guides
- `api-reference/` — Auto-generated from `z3rno-server` OpenAPI 3.1 spec

## What this is not

- **Not internal company documentation.** For PRD, BRD, task breakdown, architecture document, tech stack, investor pitch, gap analysis, etc., see [z3rno-process-docs](https://github.com/the-ai-project-co/z3rno-process-docs).
- **Not the SDK source code.** Those live in [z3rno-sdk-python](https://github.com/the-ai-project-co/z3rno-sdk-python) and [z3rno-sdk-typescript](https://github.com/the-ai-project-co/z3rno-sdk-typescript).

## Local development

```bash
# Install the Mintlify CLI (one-time)
npm install -g mintlify

# Run the docs site locally
mintlify dev
# Site at http://localhost:3000
```

## Contributing

Docs PRs are very welcome. Small fixes can be submitted directly; larger restructures should open an issue first. See `CONTRIBUTING.md` (to be added in Week 6).
