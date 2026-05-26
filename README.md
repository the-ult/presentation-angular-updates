# Angular 21 & 22 — Slidev presentation

This repository contains a [Slidev](https://sli.dev/) presentation about **Angular 21** and **Angular 22**.
The deck is aimed at medium-to-advanced Angular developers and focuses on the biggest framework shifts between late 2025 and mid 2026.

The presentation covers topics such as zoneless change detection, Signal Forms, the OnPush-by-default change, Vitest as the default test runner, Angular's MCP tooling, and the early WebMCP / agentic UI story.

## What this deck covers

- Angular 21 as the **consolidation** release
- Angular 22 as the **graduation** release
- Zoneless change detection and migration gotchas
- Signals primitives, `linkedSignal()`, and the Resource APIs
- Signal Forms moving from experimental to stable
- Template and compiler improvements
- Vitest migration and testing trade-offs
- Angular MCP and WebMCP
- Breaking changes, deprecations, and upgrade guidance

## Project structure

- `slides.md` — main Slidev deck
- `slides-pages.md` — GitHub Pages entry that imports the main deck and enables `routerMode: hash`
- `angular_notest.md` — long-form research and source notes behind the talk
- `assets/` — images and presentation assets
- `components/` — reusable Vue components for the deck
- `pages/` — additional imported slide content
- `snippets/` — code snippets imported into slides
- `styles/index.css` — global Slidev styling
- `.github/workflows/deploy-pages.yml` — GitHub Pages deployment workflow
- `netlify.toml` / `vercel.json` — alternative hosting configs

## Run locally

Prerequisites:

- Node.js (current LTS recommended)
- `pnpm`

Install dependencies and start the local Slidev dev server:

- `pnpm install`
- `pnpm run dev`

Then open <http://localhost:3030>.

## Available scripts

| Command | Description |
| --- | --- |
| `pnpm run dev` | Start the Slidev development server and open the deck in the browser |
| `pnpm run build` | Build the main deck as a static SPA in `dist/` |
| `pnpm run build:pages` | Build the GitHub Pages entry using the repository base path |
| `pnpm run export` | Run Slidev's export command for PDF/print-style output |

> Note: exporting with Slidev may require a browser runtime in your environment, depending on your local setup.

## Editing the presentation

- Update the talk content in `slides.md`
- Use `angular_notest.md` as the research/source document for the narrative and examples
- Keep GitHub Pages-specific routing in `slides-pages.md`
- Put shared deck styling in `styles/index.css`
- Add images to `assets/` and reusable interactive pieces to `components/`

The deck uses the `seriph` theme and standard Slidev features such as Markdown slides, Vue components, syntax highlighting, presenter notes, and static hosting.

## Deployment

### GitHub Pages

This project includes a dedicated GitHub Pages setup:

- Entry file: `slides-pages.md`
- Workflow: `.github/workflows/deploy-pages.yml`
- Local Pages build: `pnpm run build:pages`
- Expected URL: `https://the-ult.github.io/presentation-angular-updates/`

The Pages entry imports `slides.md` and uses hash-based routing so the presentation works reliably on a repository-scoped GitHub Pages site.

To enable GitHub Pages deployment:

1. Push the repository to GitHub.
2. Open the repository settings.
3. Go to **Pages**.
4. Set **Source** to **GitHub Actions**.
5. Push to `main` to deploy automatically, or run the workflow manually from the **Actions** tab.

### Other hosting options

`netlify.toml` and `vercel.json` are included if you want to deploy the deck outside GitHub Pages with cleaner SPA routing.

## License

This repository is currently published as **UNLICENSED / all rights reserved**.
See [`LICENSE`](./LICENSE) for details.

Third-party names, logos, trademarks, and brand assets remain the property of
their respective owners.

## Built with Slidev

This project uses Slidev's standard workflow:

- `slidev --open` for local development
- `slidev build` for static output
- `slidev export` for presentation export

If you want to learn more about the platform itself, see the official Slidev documentation at <https://sli.dev/>.
