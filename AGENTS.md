# Repository Guidelines

## Project Structure & Module Organization
- `app/` contains the Next.js App Router entry points; `page.tsx` drives the divination flow and `server.ts` handles streamed AI responses.
- `components/` hosts reusable UI and interaction pieces, with primitives in `components/ui/`; pair shared logic and data with `lib/` (e.g., `lib/data/gua-list.json`) and keep static assets in `public/img/`.
- Root configuration files (`tailwind.config.ts`, `prettier.config.js`, `next.config.js`, `tsconfig.json`) define the build, styling, and TypeScript baselines; leave credentials in a local `.env.local`.

## Build, Test, and Development Commands
- `pnpm install` — install dependencies (pnpm is the canonical package manager).
- `pnpm dev` — launch the turbo-enabled dev server on localhost.
- `pnpm build` — produce the production bundle; run before shipping infrastructure changes.
- `pnpm start` — serve the built bundle to mirror production.
- `pnpm lint` — execute ESLint + Next checks; fixes are required before opening a PR.

## Coding Style & Naming Conventions
- TypeScript-first: export React components with PascalCase filenames, utilities and hooks in camelCase, constants in UPPER_SNAKE_CASE (`ERROR_PREFIX`), and JSON data in kebab-case.
- Prettier (with `prettier-plugin-tailwindcss`) governs formatting: 2-space indentation, double quotes, trailing commas where valid; enable format-on-save.
- Tailwind utilities should remain sorted automatically; share repeated variants through helpers in `clsx` or `class-variance-authority` modules inside `lib/`.

## Testing Guidelines
- Automated tests are not yet committed; when adding behavior, accompany it with focused component or utility specs (e.g., Vitest + Testing Library) saved alongside sources as `<name>.test.ts[x]`.
- Document manual verification for streamed responses (`pnpm dev`, coin toss flow, AI error handling) in the PR description until the suite exists.
- Target coverage of new logic paths, especially around `app/server.ts` error branches and stateful hooks (`Divination`, `ResultAI`); flag any gaps explicitly.

## Commit & Pull Request Guidelines
- Follow Conventional Commits (`feat:`, `chore:`, `build:`) as seen in history; keep subjects imperative and scoped to one concern.
- Rebase or squash to remove noisy WIP commits; reference linked issues when applicable.
- PRs should include a concise summary, screenshots for visual changes (`docs/screenshots.jpg` style), environment variable notes, and evidence of `pnpm lint` plus any tests or manual walkthroughs.

## Environment & Secrets
- Required variables: `OPENAI_API_KEY`; optional overrides `OPENAI_BASE_URL` and `OPENAI_MODEL`. Store them in `.env.local` and mirror them in Vercel project settings.
- Never commit secret material; rotate credentials after demos and document any new configuration keys in the PR.
