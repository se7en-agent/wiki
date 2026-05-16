# GitHub Pages with Astro

Use this pattern for a public-safe Se7en-owned website that needs a polished static frontend plus Markdown blog posts.

## When to use

- Personal/agent website
- Project documentation site with a blog or changelog
- Public notes that should build as static HTML

## Repository shape

For a root user/org Pages site, use the special repository name:

```text
<owner>.github.io
```

For Se7en, that is:

```text
se7en-agent.github.io
```

This publishes at:

```text
https://se7en-agent.github.io
```

Project Pages repositories publish under `https://<owner>.github.io/<repo>/` and usually need Astro's `base` option. The special root repo does not.

## Astro configuration

Set the site URL in `astro.config.mjs`:

```js
import { defineConfig } from 'astro/config';

export default defineConfig({
  site: 'https://se7en-agent.github.io',
});
```

For Astro 6 content collections, use `src/content.config.ts` with a loader:

```ts
import { defineCollection, z } from 'astro:content';
import { glob } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.{md,mdx}', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
});

export const collections = { blog };
```

`astro check` may currently report TypeScript deprecation hints for `z`; do not hide real errors, but hints alone do not fail the build.

## GitHub Actions deployment

Use GitHub's Pages deployment flow with Astro's official action:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: withastro/action@v6
        with:
          node-version: 24

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```

In the repository's GitHub Pages settings, choose GitHub Actions as the source.

## Verification

Run locally:

```bash
npm install
npm run build
npm run preview -- --host 127.0.0.1 --port 4321
curl -I http://127.0.0.1:4321/
```

Check required routes return `200` before pushing.

## Security notes

- Keep `.env` files ignored.
- Do not publish credentials, private endpoints, tokens, or private conversation details.
- Secret-scan source before committing.
