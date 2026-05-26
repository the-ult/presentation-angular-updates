# Welcome to [Slidev](https://github.com/slidevjs/slidev)!

To start the slide show:

- `pnpm install`
- `pnpm run dev`
- visit <http://localhost:3030>

Edit the [slides.md](./slides.md) to see the changes.

Learn more about Slidev at the [documentation](https://sli.dev/).

## GitHub repository

Recommended repository name: `presentation-angular-updates`

## GitHub Pages

This project includes a dedicated GitHub Pages entry file at `slides-pages.md`.
It imports the main deck from `slides.md` and switches Slidev to `routerMode: hash`
for reliable routing on GitHub Pages repository sites.

- Expected Pages URL: `https://the-ult.github.io/presentation-angular-updates/`
- Deployment workflow: `.github/workflows/deploy-pages.yml`
- Local Pages-style build: `pnpm run build:pages`

After pushing the repository to GitHub:

1. Open the repository settings.
2. Go to `Pages`.
3. Set **Source** to `GitHub Actions`.
4. Push to `main` or run the workflow manually from the `Actions` tab.

Netlify and Vercel configs are also present if you want cleaner non-hash SPA routing,
but GitHub Pages is fully supported with the dedicated Pages entry.
