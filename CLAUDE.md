# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Full-stack recipe/food application ("Foodies") with a React/TypeScript frontend and an Express/Sequelize backend, backed by PostgreSQL.

## Setup & Running

### First-time setup (from repo root)

```bash
npm --prefix backend install
npm --prefix frontend install
cp backend/.env.example backend/.env   # then edit credentials
```

### Start everything

```bash
# 1. Start Postgres
docker-compose -f backend/docker-compose.dev.yaml up -d postgres

# 2. Apply migrations
npm --prefix backend run db:migrate

# 3. Run both servers concurrently
npx concurrently -n backend,frontend -c green,blue \
  "npm --prefix backend run dev" \
  "npm --prefix frontend run dev"
```

Backend: `http://localhost:3000` · Frontend: `http://localhost:5173` · UI Kit: `http://localhost:5173/ui-kit`

### Backend scripts (`npm --prefix backend run <script>`)

| Script | What it does |
|---|---|
| `dev` | nodemon dev server |
| `start` | production server |
| `db:migrate` | apply pending migrations |
| `db:migrate:undo` | revert last migration |
| `db:seed` | run all seeders |
| `db:reset` | undo all migrations, then re-apply |

### Frontend scripts (`npm --prefix frontend run <script>`)

| Script | What it does |
|---|---|
| `dev` | Vite dev server |
| `build` | TypeScript check + Vite production build |
| `preview` | preview production build |

## Architecture

### Backend (`backend/`)

Layered Express app. Entry point is `app.js` — registers middleware, mounts routers, handles Sequelize errors globally, and starts only after `db.sequelize.authenticate()` succeeds.

```
app.js              ← entry point
config/config.js    ← Sequelize config (reads .env)
models/             ← Sequelize models (auto-loaded by models/index.js)
migrations/         ← Sequelize migrations
seeders/            ← Sequelize seeders
routes/             ← Express routers → mounted as /api/<resource>
controllers/        ← request/response handlers
services/           ← business logic
middleware/         ← Express middleware (auth, upload, etc.)
schemas/            ← Joi validation schemas
helpers/            ← shared utilities
public/             ← static files served by Express
```

Key env vars: `POSTGRES_*`, `APP_PORT` (default 3000), `JWT_SECRET`, `JWT_EXPIRES_IN`, `EMAIL_SERVICE_*`, `BASE_API_URL`.

### Frontend (`frontend/src/`)

Follows **Feature-Sliced Design (FSD)**:

```
app/            ← providers, router setup (AppRouter.tsx)
pages/          ← route-level components (one folder per page)
widgets/        ← composed UI blocks (e.g. SharedLayout)
features/       ← user-action slices: auth, favorites, follow, recipe-create
entities/       ← domain models: area, category, ingredient, recipe, user
shared/
  ui/           ← primitive components: Button, Input, TextArea, Select, Checkbox, Radio
  hooks/        ← typed Redux hooks (useAppDispatch, useAppSelector)
  constants/    ← APP_ROUTES and other shared constants
  styles/       ← global.css + variables.css (design tokens)
  types/        ← shared TypeScript types
  utils/        ← shared utilities
store/          ← Redux store, rootReducer, and slices
api/
  client.ts          ← apiClient with baseUrl from VITE_API_URL
  endpoints/         ← one file per resource (authApi, recipesApi, etc.)
```

**Redux slices**: `auth`, `users`, `recipes`, `categories`, `ingredients`, `areas`, `testimonials`, `favorites`, `followers`.

## Coding Rules

### API access

All HTTP calls go through `api/endpoints/` files — never call `fetch` directly in components, slices, or services.

### HTML element encapsulation (enforced)

Never use raw functional HTML elements in application code. Always use the wrappers from `shared/ui/`:

| Raw element | Use instead |
|---|---|
| `<input>` | `<Input />` |
| `<textarea>` | `<TextArea />` |
| `<select>` | `<Select />` |
| `<button>` | `<Button />` |
| `<input type="checkbox">` | `<Checkbox />` |
| `<input type="radio">` | `<Radio />` |

Structural/layout elements (`<div>`, `<section>`, `<article>`, etc.) may be used directly.

### Design tokens

Use CSS custom properties from `shared/styles/variables.css`, prefixed `--fd-*`. Prefer semantic tokens (e.g. `--fd-text-primary`) over primitives (e.g. `--fd-color-main`). Dark theme is applied via `[data-theme='dark']`.

### Constants & config

No hardcoded literals in business logic. Store reusable values in `constants/`, `config/`, or `enums/` modules.

### FSD layer imports

Imports must only flow downward: `pages` → `widgets` → `features` → `entities` → `shared`. Never import upward or across sibling layers at the same level.

## Git Commits

Conventional Commits (lowercase): `<type>(<scope>): <subject>`

Allowed types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `build`, `ci`, `perf`, `revert`

- Subject: imperative mood, under 72 chars
- One logical change per commit; no vague messages (`wip`, `fix stuff`)
- Use body to explain *why*; use `BREAKING CHANGE:` footer for breaking changes
