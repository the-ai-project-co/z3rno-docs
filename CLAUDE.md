# CLAUDE.md

## Project

z3rno-docs is the public-facing product documentation site for Z3rno, built with Mintlify. It renders MDX content as an interactive documentation site at docs.z3rno.dev.

## Quick Reference

```bash
npm install -g mintlify          # Install Mintlify CLI
mintlify dev                     # Local dev server at http://localhost:3000
```

## Architecture

- `mint.json` — Mintlify configuration: navigation, colors (#00D4AA teal), logo, topbar links
- `introduction.mdx` — Landing page
- `quickstart.mdx` — Getting started guide
- Content is MDX (Markdown + JSX components)

## Key Conventions

- All content is MDX format
- Navigation structure defined in mint.json
- API reference auto-generated from OpenAPI spec (when configured)
- Markdown linting via markdownlint-cli2
- No code, no tests, no build step (Mintlify handles rendering)
- Brand colors: primary #00D4AA, dark #00A88D
- Currently a scaffold with 2 pages (needs significant content before launch)
