# Contributing to Z3rno Docs

Thank you for helping improve Z3rno's documentation! Good docs make the difference between a project people try and a project people adopt.

## Getting started

### Prerequisites

- [Node.js](https://nodejs.org/) 18+
- [Mintlify CLI](https://mintlify.com/docs/development)

### Local development

1. Clone the repository:

   ```bash
   git clone https://github.com/the-ai-project-co/z3rno-docs
   cd z3rno-docs
   ```

2. Install the Mintlify CLI:

   ```bash
   npm install -g mintlify
   ```

3. Start the local dev server:

   ```bash
   mintlify dev
   ```

4. Open [http://localhost:3000](http://localhost:3000) in your browser. Changes to `.mdx` files will hot-reload.

## Content guidelines

### Format

- All content pages use **MDX** (Markdown with JSX components).
- Frontmatter is required: every page must have `title` and `description` fields.
- Use Mintlify components (`<Note>`, `<Warning>`, `<Card>`, `<CardGroup>`, `<CodeGroup>`, `<Tabs>`) for rich content.

### Style

- Write in second person ("you") and present tense.
- Keep sentences short and direct.
- Use code blocks with language identifiers (` ```python `, ` ```bash `, etc.).
- Prefer tables for reference data (environment variables, configuration options).
- Add `<Note>` blocks for important context and `<Warning>` blocks for destructive actions or common pitfalls.

### Navigation

The navigation structure is defined in `mint.json`. If you add a new page, add it to the appropriate group in the `navigation` array.

### Images and assets

- Place images in the `/images` directory.
- Use descriptive filenames: `memory-lifecycle-diagram.png`, not `screenshot1.png`.
- Prefer SVG for diagrams.

## Pull request process

1. **Fork** the repository and create a branch from `main`.
2. **Write** your content following the guidelines above.
3. **Test** locally with `mintlify dev` to verify rendering.
4. **Submit** a pull request with a clear description of what you added or changed.
5. A maintainer will review your PR. We aim to review documentation PRs within 3 business days.

### What makes a good PR

- One topic per PR (don't mix a new guide with typo fixes).
- Include screenshots or screen recordings for UI-related docs.
- Link to related issues if applicable.

## Reporting issues

Found a typo, broken link, or missing content? [Open an issue](https://github.com/the-ai-project-co/z3rno-docs/issues/new) with:

- The page URL or file path
- What's wrong or what's missing
- (Optional) A suggested fix

## License

By contributing, you agree that your contributions will be licensed under the [Apache 2.0 License](LICENSE).
