# GitHub Pages with Nuxt Content

Use this pattern when a public-safe website needs static generation, Vue components, and Markdown content collections.

## When to use

- A personal or agent website with a journal or notes shelf.
- A static documentation surface where Markdown should be first-class content.
- A GitHub Pages site that should still support direct URLs such as `/about/` and `/journal/first-signal/`.

## Repository shape

For an organization or user Pages root site, use the special repository name:

```text
<owner>.github.io
```

For Se7en, that is:

```text
se7en-agent.github.io
```

The special root repo publishes at:

```text
https://se7en-agent.github.io/
```

## Minimal Nuxt configuration

Use Nuxt Content for Markdown collections and generate a static output for Pages:

```ts
export default defineNuxtConfig({
  modules: ["@nuxt/content"],
  css: ["~/assets/css/main.css"],
  app: {
    head: {
      title: "Site title",
      htmlAttrs: { lang: "en" },
      meta: [{ name: "viewport", content: "width=device-width, initial-scale=1" }],
    },
  },
  nitro: {
    prerender: {
      crawlLinks: true,
      routes: ["/", "/about/", "/journal/", "/notes/"],
    },
  },
});
```

A simple content config keeps journal and note schemas explicit:

```ts
import { defineCollection, defineContentConfig, z } from "@nuxt/content";

export default defineContentConfig({
  collections: {
    journal: defineCollection({
      type: "page",
      source: "journal/*.md",
      schema: z.object({
        title: z.string(),
        date: z.string(),
        description: z.string(),
        tags: z.array(z.string()).default([]),
      }),
    }),
  },
});
```

## Scripts

For static hosting, keep the build target explicit:

```json
{
  "scripts": {
    "dev": "nuxt dev",
    "build": "nuxt generate",
    "generate": "nuxt generate",
    "preview": "npx serve .output/public"
  }
}
```

## Dependency manager choice

If the project uses pnpm, install pnpm before `actions/setup-node` enables pnpm cache. Otherwise the Pages workflow can fail before dependencies are installed.

Keep package-manager-specific commands consistent:

```json
{
  "packageManager": "pnpm@10.30.1",
  "scripts": {
    "build": "nuxt build",
    "generate": "nuxt generate"
  }
}
```

## Template attribution

When starting from a public template, keep the license file and credit the original template in the README. Template adoption is fine, but public surfaces should make provenance explicit and must replace demo copy, sample contacts, fake projects, and placeholder social links with accurate project content.

## GitHub Actions deployment

Upload `.output/public` as the Pages artifact. For pnpm-based Nuxt sites, set up pnpm before Node cache configuration:

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
      - uses: pnpm/action-setup@v4
        with:
          version: 10.30.1
      - uses: actions/setup-node@v6
        with:
          node-version: 24
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm generate
      - uses: actions/upload-pages-artifact@v4
        with:
          path: .output/public

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

In repository settings, GitHub Pages should use GitHub Actions as the source.

## Verification

Before committing:

```bash
pnpm generate
python - <<'PY'
from pathlib import Path
routes = ["/", "/about/", "/journal/", "/notes/"]
root = Path(".output/public")
for route in routes:
    page = root / (route.strip("/") or "") / "index.html"
    assert page.exists() and page.stat().st_size > 0, page
    print(route, "OK")
PY
```

Then run a source-only secret scan, excluding generated output and dependency directories.

## Security notes

- Ignore `.nuxt/`, `.output/`, `.data/`, `node_modules/`, and local env files.
- Do not scan or commit generated bundles as source; minified third-party code creates noisy false positives.
- Do not put secrets, tokens, private endpoints, or private conversations in Markdown content.
