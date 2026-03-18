# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Geist Cruceros** is a B2B travel portal for cruise agencies. It consists of two sibling directories:
- `geist-cruises-backend/` — Node.js + Express REST API
- `geist-cruises-frontend/` — Vanilla HTML/CSS/JS (no build step)

---

## Design System & UI Guidelines

### Brand Identity
- **Client:** Geist (www.geist.tur.ar) — Argentine wholesale travel operator
- **Audience:** B2B — professional travel agents, NOT end tourists
- **Tone:** Professional, trustworthy, efficient. Never touristic or flashy.
- **Logo:** `https://www.geist.tur.ar/images/logochico.png` (white version: `logoblanco.png`)

### Color Palette
- **Primary:** Navy blue `#1a2f5e` (or close match from Geist site)
- **Surface:** White `#ffffff`
- **Neutrals:** Always tint grays with a hint of blue — never pure gray or pure black
- **Accents:** Use sparingly for CTAs and status indicators only

### Typography
- Avoid Inter as the sole typeface — it's overused and generic
- Prefer a font pairing: a professional sans-serif for UI + a humanist or geometric for headings
- Maintain a clear modular scale — don't set font sizes arbitrarily

### B2B UX Principles
- **Information density:** Agents need to scan many cruises fast. Show naviera, ship, departure date, ports, duration, and starting price in the card — no click required.
- **Speed over decoration:** Fewer animations, more responsiveness. Agents work under time pressure.
- **Form clarity:** Filter labels must be precise. Use professional terminology (naviera, puerto de salida, cabina, régimen).
- **Error states matter:** Empty search results, API timeouts, and invalid date ranges must have clear, actionable messages — not generic "algo salió mal".

### Anti-Patterns to Avoid (Impeccable rules)
- ❌ No purple gradients or generic hero gradients
- ❌ No gray text on colored backgrounds (contrast fail)
- ❌ No cards nested inside cards
- ❌ No bounce or elastic easing animations
- ❌ No Inter as the only font
- ❌ No pure black `#000` or pure gray — always tint with brand color
- ❌ No placeholder "Lorem ipsum" copy — write real B2B microcopy in Spanish

### Impeccable Commands (installed in `.claude/`)
Use these commands when working on frontend files (`geist-cruises-frontend/`):

| Command | When to use |
|---|---|
| `/audit` | Check accessibility, responsive behavior, performance |
| `/critique` | UX review: hierarchy, scan-ability, conversion clarity |
| `/normalize` | Fix inconsistent spacing, color, or type across the 3 HTML files |
| `/polish` | Final pass before showing to Geist stakeholders |
| `/clarify` | Improve filter labels, button copy, error messages, empty states |
| `/harden` | Handle edge cases: 0 results, API timeout, invalid dates, no cabins |
| `/extract` | Refactor repeated patterns into reusable CSS classes or JS components |

**Examples for this project:**
```
/audit index.html
/critique resultados-cruceros
/clarify filtros-busqueda
/polish card-crucero
/harden estado-sin-resultados
/normalize index.html detalle.html admin.html
```

### Key UI Components & Design Notes

**Cruise result card** (`index.html`)
- Must show: naviera logo or name, ship name, departure date, departure port, duration (nights), itinerary summary (ports), cabin categories with starting price
- Status badge for OFFLINE vs live Widgety cruises should be subtle, not distracting

**Detail page** (`detalle.html`)
- Day-by-day itinerary should use a timeline or table layout — not a flat list
- Cabin category grid: Interior / Exterior / Balcón / Suite — clear pricing hierarchy
- Ship info section should feel editorial, not like a database dump

**Admin dashboard** (`admin.html`)
- Optimize for efficiency: dense tables, quick filters, inline actions
- Use a consistent sidebar or top-nav pattern
- Stats cards should use tinted backgrounds, not pure white cards on white background

---

## Backend Commands

```bash
cd geist-cruises-backend

npm run dev          # Development server (nodemon, port 3001)
npm start            # Production server
npm test             # Run all tests (Jest)
npm run test:watch   # Watch mode
npm run migrate      # Apply DB migrations (knex migrate:latest)
npm run seed         # Run DB seeds
```

Run a single test file:
```bash
npx jest tests/unit/cruise.service.test.js --verbose
```

Run only unit or integration tests:
```bash
npx jest tests/unit/
npx jest tests/integration/
```

---

## Environment Variables (backend `.env`)

| Variable | Required | Notes |
|---|---|---|
| `DATABASE_URL` | prod | PostgreSQL connection string |
| `JWT_SECRET` | yes | Sign JWT tokens |
| `WIDGETY_APP_ID` | yes | Widgety API credential |
| `WIDGETY_TOKEN` | yes | Widgety API credential |
| `ALLOWED_ORIGINS` | yes | CORS origins, comma-separated |
| `PORT` | optional | Server port (default 3001) |
| `OPENROUTER_API_KEY` | optional | Enables AI features (OpenRouter) |
| `WIDGETY_MARKET` | optional | Default `uk` |
| `WIDGETY_BASE_URL` | optional | Override Widgety API base URL |
| `WIDGETY_ENRICH_LIMIT` | optional | Max holidays to detail-enrich per search (default 10) |
| `WIDGETY_ENRICH_TIMEOUT_MS` | optional | Enrichment timeout in ms (default 20000) |
| `CACHE_TTL_CRUISES` | optional | Seconds (default 3600) |
| `CACHE_TTL_CRUISE_DETAIL` | optional | Seconds (default 7200) |
| `CACHE_TTL_SHIPS` | optional | Seconds (default 86400) |
| `CACHE_TTL_PORTS` | optional | Seconds (default 86400) |
| `JWT_EXPIRES_IN` | optional | Default `7d` |

In production, missing `JWT_SECRET`, `DATABASE_URL`, `WIDGETY_APP_ID`, or `WIDGETY_TOKEN` causes a hard crash at startup.

---

## Backend Architecture

### Layer Responsibilities

```
routes/index.js
  └─ controllers/*.controller.js   (HTTP: parse req, send res)
       └─ services/cruise.service.js  (orchestrator: business logic + normalization)
            ├─ providers/widgety.provider.js  (Widgety API + in-memory cache)
            └─ repositories/offlineCruise.repository.js  (PostgreSQL via Knex)
```

- **Controllers** handle HTTP only — no business logic, no DB calls.
  - `cruise.controller.js` — search, detail, ships, filters
  - `offlineCruise.controller.js` — offline cruise CRUD (admin)
  - `admin.controller.js` — dashboard stats, agencies, audit log
  - `budget.controller.js` — budget listing, creation, PDF data
  - `ports.controller.js` — port listing, add/delete custom ports
  - `ships.admin.controller.js` — ship override management
  - `ai.controller.js` — AI-powered search, content generation, translation
- **`cruise.service.js`** is the central orchestrator. It combines results from Widgety + offline cruises using `Promise.allSettled` (so one failing provider doesn't break the other). It also contains all normalization logic (Widgety raw → frontend format). After combining results, it applies a **post-filter by region** as a safety net (in case Widgety returns unfiltered results).
- **Providers** (`src/providers/`) encapsulate external API calls and caching. `WidgetyProvider` extends `BaseProvider`. Adding a new data source means creating a new provider and plugging it into `cruise.service.js`.
- **Repositories** (`src/repositories/`) handle PostgreSQL access via Knex — one file per table/domain: `agency`, `auditLog`, `booking`, `budget`, `cabinDetails`, `contentOverride`, `offlineCruise`, `operator`, `pricing`, `provider`, `ship`.
- **`widgety.service.js`** in `src/services/` is an older implementation — the active one is `widgety.provider.js` in `src/providers/`.

### API Endpoints

#### Auth
- `POST /api/auth/login` — Login (rate-limited: 10 failed attempts / 15 min)
- `GET /api/auth/me` — Current user info
- `POST /api/auth/register` — Register agency (admin-only)

#### Cruises (authenticated)
- `GET /api/cruises/filters` — Filter options (operators, ports)
- `GET /api/cruises` — Search with filters
- `GET /api/cruises/dates/:date_ref` — Date-specific detail with prices + itinerary
- `GET /api/cruises/:ref` — Cruise detail + day-by-day itinerary

#### Ships (authenticated)
- `GET /api/ships` — List all ships
- `GET /api/ships/:ref` — Ship detail

#### Budgets (authenticated)
- `GET /api/budgets` — List budgets
- `POST /api/budgets` — Create budget
- `GET /api/budgets/:id/pdf` — Budget PDF data

#### AI (authenticated, requires `OPENROUTER_API_KEY`)
- `POST /api/ai/parse-search` — Free-text → structured search filters
- `POST /api/ai/generate-budget` — Generate sales content for budgets
- `POST /api/ai/generate-email` — Generate quote emails
- `POST /api/ai/translate-ship` — Translate ship data to Spanish
- `GET /api/ai/status` — Check if AI is configured

#### Admin (requires admin role)
- `GET /api/admin/dashboard` — Dashboard stats
- `GET /api/admin/agencies` — List agencies
- `PATCH /api/admin/agencies/:id/status` — Update agency status
- `GET /api/admin/audit-log` — Audit log
- `GET /api/admin/ports` — List ports (any authenticated user)
- `POST /api/admin/ports` — Add custom port
- `DELETE /api/admin/ports/:unlocode` — Delete custom port
- `GET /api/admin/offline-cruises` — List offline cruises
- `POST /api/admin/offline-cruises` — Create offline cruise
- `GET /api/admin/offline-cruises/:id` — Offline cruise detail
- `PATCH /api/admin/offline-cruises/:id/status` — Update status
- `POST /api/admin/offline-cruises/:id/inventory` — Upsert cabin inventory
- `GET /api/admin/ships/:slug/override` — Get ship override
- `POST /api/admin/ships/:slug/override` — Create/update ship override
- `PATCH /api/admin/ships/:slug/override` — Update ship override
- `GET /api/admin/cache` — Cache stats
- `POST /api/admin/cache/clear` — Flush cache (optional `?prefix=`)

#### Health
- `GET /api/health` — Health check (public, no auth)

### Cruise Data Sources

There are two types of cruises mixed together in search results:

1. **Widgety cruises** — live data from `https://www.widgety.co.uk/api`. Their `holiday_ref` is a string like `EMERALDEWAR1515713`. Search results are enriched with per-date detail in batches of 5.
2. **Offline cruises** — stored in PostgreSQL (`offline_cruises` table). Their `holiday_ref` and `date_ref` are prefixed with `OFFLINE:` (e.g., `OFFLINE:42`). Controlled through `/api/admin/offline-cruises`.

Controllers detect the `OFFLINE:` prefix to dispatch to the correct service method.

### Authentication & Roles

- JWT tokens are issued at `POST /api/auth/login`, expire in 7 days.
- Login is rate-limited: 10 failed attempts per IP per 15 minutes.
- `authenticate` middleware — any valid token → sets `req.user`.
- `requireAdmin` middleware — token + `role === 'admin'` required.
- Agency registration (`POST /api/auth/register`) is admin-only. Agencies and users share a relationship: one agency → one user (initially).

### Port Resolution

`resolvePort(nameOrCode)` in `widgety.provider.js` resolves a port name or UNLOCODE to a Widgety UNLOCODE:
1. Already a UNLOCODE regex match → return as-is
2. Check `src/data/custom-ports.json` (managed via `/api/admin/ports`)
3. Check `src/data/ports.js` (static map)
4. Fallback: call Widgety `/ports.json`

### AI Features

AI modules live in `src/ai/`:
- `ai.client.js` — OpenRouter API client (uses `meta-llama/llama-3.1-8b-instruct`)
- `search-parser.js` — Converts free-text search queries to structured filters
- `content-generator.js` — Generates budget sales content and quote emails
- `translator.js` — Translates ship data and amenities to Spanish
- `test-ai.js` — Testing utility script

AI endpoints (`/api/ai/*`) use OpenRouter (not Anthropic/OpenAI directly). If `OPENROUTER_API_KEY` is missing, AI endpoints return errors but the rest of the API continues normally. OpenRouter impone rate limits propios — si se recibe un error 429 ("too many requests"), simplemente esperar unos minutos y reintentar; el límite se resetea automáticamente.

### Data Files

- `src/data/custom-ports.json` — Custom port mappings (managed via admin API)
- `src/data/ship-overrides.json` — Ship content overrides (managed via admin API)
- `src/data/ports.js` — Static port name → UNLOCODE map

### Cache

In-memory `node-cache` lives inside `WidgetyProvider`. Cache keys follow patterns like `holidays:{JSON}`, `holiday:{ref}:{market}`, `ship:{ref}`, `operators:all`, `ports:{JSON}`. Cache can be inspected (`GET /api/admin/cache`) and flushed (`POST /api/admin/cache/clear?prefix=holidays`) by admins.

### Database Migrations

Knex migrations live in `src/db/migrations/` (8 migration files: agencies/users, operators/ships, providers/pricing, budgets/bookings, content/cabin, offline cruises, audit log, performance indexes). Run `npm run migrate` before starting for the first time or after pulling changes. Seeds in `src/db/seeds/` contain initial agency/user data.

### Testing

- Tests are in `tests/unit/` and `tests/integration/`.
- `tests/__mocks__/dbConnection.js` mocks the Knex DB connection — all unit tests run without a real database.
- `tests/setup.js` configures the test environment (sets `NODE_ENV=test`).
- Integration tests (`tests/integration/api.test.js`) use `supertest` against the Express app.

### Search Parameter Mapping (Frontend → Backend)

The frontend sends query params that the controller maps to internal filter names. Key mappings:
- `destination` → `regions` (controller accepts both `regions` and `destination`)
- `operator` → `operators` (controller accepts both)
- `departure_port`, `date_from`, `date_to`, `duration_min`, `duration_max` → passed as-is

When adding new search filters, ensure the frontend param name matches what the controller reads from `req.query` in `cruise.controller.js`.

### API Response Format

All responses follow `{ success: boolean, data?: any, error?: string }`.

---

## Frontend

Three standalone HTML files in `geist-cruises-frontend/`:
- `index.html` — Main cruise search and listing
- `detalle.html` — Cruise detail page
- `admin.html` — Admin dashboard

No build step, no npm dependencies, no bundler. Open directly in a browser or serve statically. The frontend communicates with the backend at `http://localhost:3001/api` (configurable via a `const API_URL` at the top of each file's `<script>` block).

### AI Search UI

The AI search bar in `index.html` is behind a collapsible toggle ("¿Querés buscar con IA?"). It checks `/api/ai/status` on load — if the AI is not configured or the user has no token, the entire `ai-search-wrapper` is hidden. The toggle is controlled by `toggleAISearch()` in the inline `<script>` block.

---

## Orquestación del Flujo de Trabajo

### 1. Predeterminado del Nodo de Planificación
- Entrar en modo de planificación para CUALQUIER tarea no trivial (3+ pasos o decisiones arquitectónicas)
- Si algo sale mal, PARAR y volver a planificar de inmediato — no seguir forzando
- Usar el modo de planificación para los pasos de verificación, no solo para la construcción
- Escribir especificaciones detalladas por adelantado para reducir la ambigüedad

### 2. Estrategia de Subagentes
- Usar subagentes generosamente para mantener limpia la ventana de contexto principal
- Descargar la investigación, exploración y análisis paralelo en subagentes
- Para problemas complejos, asignar más cómputo a través de subagentes
- Un enfoque por subagente para una ejecución centrada en resultados

### 3. Ciclo de Automejora
- Después de CUALQUIER corrección del usuario: actualizar `tasks/lessons.md` con el patrón
- Escribir reglas para ti mismo que eviten el mismo error
- Iterar sin piedad sobre estas lecciones hasta que disminuya la tasa de errores
- Revisar las lecciones al inicio de la sesión para el proyecto relevante

### 4. Verificación Antes de Finalizar
- Nunca marcar una tarea como completada sin demostrar que funciona
- Comparar el comportamiento entre el principal y los cambios cuando sea relevante
- Preguntarte: "¿Aprobaría esto un ingeniero senior?"
- Ejecutar pruebas, verificar registros, demostrar la corrección

### 5. Exigir Elegancia (Equilibrado)
- Para cambios no triviales: pausar y preguntar "¿hay una forma más elegante?"
- Si un arreglo se siente apresurado: "Sabiendo todo lo que sé ahora, implementa la solución elegante"
- Omitir esto para arreglos simples y obvios — no sobre-diseñar
- Cuestionar tu propio trabajo antes de presentarlo

### 6. Corrección Autónoma de Errores
- Cuando se te dé un informe de error: simplemente arréglalo. No pidas que te guíen de la mano
- Señala los registros, errores, pruebas fallidas y luego resuélvelas
- Cero cambio de contexto por parte del usuario
- Ve a arreglar las pruebas de CI que fallan sin que te digan cómo

---

## Gestión de Tareas

- **Planificar Primero:** Escribir plan en `tasks/todo.md` con elementos marcables
- **Verificar Plan:** Comprobar antes de comenzar la implementación
- **Seguir Progreso:** Marcar elementos completados a medida que avanzas
- **Explicar Cambios:** Resumen de alto nivel en cada paso
- **Documentar Resultados:** Añadir sección de revisión en `tasks/todo.md`
- **Capturar Lecciones:** Actualizar `tasks/lessons.md` después de las correcciones

---

## Principios Fundamentales

- **Simplicidad Primero:** Hacer que cada cambio sea lo más simple posible. Impactar el código mínimo.
- **Sin Perezas:** Encontrar las causas raíz. Sin arreglos temporales. Estándares de desarrollador senior.
- **Impacto Mínimo:** Los cambios solo deben tocar lo necesario. Evitar introducir errores.
