# Quickstart: z3rno-docs

A detailed getting-started guide for running the Z3rno documentation site locally.

## Prerequisites

- Node.js 18+
- npm (comes with Node.js)

## Step-by-step Installation

### 1. Clone the repository

```bash
git clone https://github.com/the-ai-project-co/z3rno-docs.git
cd z3rno-docs
```

### 2. Install the Mintlify CLI

```bash
npm install -g mintlify
```

### 3. Run the docs site locally

```bash
mintlify dev
```

The site will be available at http://localhost:3000.

## First Working Example

Once running, you can:

1. Open http://localhost:3000 in your browser
2. Navigate the sidebar to see all documentation sections
3. Edit any `.mdx` file and see changes hot-reload in the browser

### Editing a page

Open `quickstart.mdx` in your editor, make a change, save. The browser will auto-refresh.

### Adding a new page

1. Create a new `.mdx` file in the appropriate directory (e.g., `concepts/new-topic.mdx`)
2. Add the page to the navigation in `mint.json`
3. The page appears in the sidebar on refresh

## Directory Structure

```
z3rno-docs/
  mint.json           # Site configuration (nav, colors, logo)
  introduction.mdx    # Landing page
  quickstart.mdx      # Getting started guide
  concepts/           # Core concepts
  sdks/               # SDK reference docs
  integrations/       # Framework integrations
  self-hosting/       # Deployment guides
  api-reference/      # Auto-generated API reference
```

## Common Issues / Troubleshooting

### 1. "mintlify: command not found"

The CLI was not installed globally or your npm global bin is not on PATH:

```bash
# Check where npm installs global binaries
npm config get prefix

# Add to PATH if needed (add to ~/.zshrc or ~/.bashrc)
export PATH="$(npm config get prefix)/bin:$PATH"
```

### 2. Port 3000 already in use

Another process (React dev server, etc.) is using port 3000. Kill it or use a different port:

```bash
mintlify dev --port 3001
```

### 3. Hot reload not working

Try stopping and restarting `mintlify dev`. If editing `mint.json` (navigation changes), a full restart is required.

### 4. MDX syntax errors

Mintlify uses MDX (Markdown + JSX). Common pitfalls:
- Curly braces `{}` must be escaped or used inside code blocks
- HTML-style comments `<!-- -->` are not supported; use `{/* comment */}`
- Components are case-sensitive

### 5. Images not loading

Place images in a `images/` or `static/` directory and reference them with relative paths:

```mdx
![Alt text](/images/diagram.png)
```
