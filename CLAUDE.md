# Work Project

Vue 3 + TypeScript + Vite SPA deployed to GitHub Pages.

## Commands

| Task | Command |
|------|---------|
| Dev server | `pnpm dev` |
| Build | `pnpm build` (runs `vue-tsc -b && vite build`) |
| Preview | `pnpm preview` |
| Lint | `pnpm lint` (ESLint via `@antfu/eslint-config`) |
| Lint fix | `pnpm lint:fix` |

## Stack

- **Framework:** Vue 3 (Composition API, `<script setup lang="ts">`)
- **Build:** Vite 8, `@vitejs/plugin-vue`
- **Routing:** Vue Router 4 (hash history, base: `/work/`)
- **Utilities:** VueUse
- **TypeScript:** strict mode, ES2020 target
- **Lint:** ESLint flat config (`@antfu/eslint-config`)
- **Package manager:** pnpm 10.33.4

## Notes

- `vue-tsc -b` runs before every production build — type errors block the build.
- The app base path is `/work/` (GitHub Pages).
- Git hooks (simple-git-hooks + lint-staged) run ESLint on staged files.
- Deploy is automated via `.github/workflows/deploy.yml` on push to `main`.
