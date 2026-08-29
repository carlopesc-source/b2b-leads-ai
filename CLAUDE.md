# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Idioma

Responde en español.

## Gestión de contexto

Mantén el uso del contexto entre el 40% y el 60%. Al superarlo, avisa y propón
compactar antes de continuar.

Antes de una tarea larga o compleja, escribe el estado en un archivo de trabajo
(objetivo, enfoque, pasos hechos, fallo actual) para poder retomarla en una
sesión nueva sin arrastrar el historial entero.

Usa subagentes para buscar y localizar código (Glob/Grep/Read). Devuelve solo el
resumen al hilo principal, no el volcado de las búsquedas.

No vuelques logs de test/build ni JSON grandes completos en el contexto: extrae
la parte relevante y descarta el resto.

Para bugs pequeños y cambios de una línea: ve directo, sin fase de investigación.
Para cambios que tocan varios archivos: investiga, propón un plan corto, y espera
aprobación antes de implementar.

## Commands

- `npm run dev` — start the Next.js dev server (localhost:3000)
- `npm run build` — production build
- `npm run start` — run the production build
- `npm run lint` — run `next lint` (ESLint, `eslint-config-next`)

There is no test suite configured in this repo (no test runner in `package.json`).

## Environment variables

The app requires these at runtime (checked explicitly in each `lib/*.js` module, which throws a descriptive error if missing rather than failing silently):

- `OPENROUTER_API_KEY` (optional `OPENROUTER_MODEL`, defaults to `openrouter/free`) — used by `lib/ai.js`
- `SERPER_API_KEY` — used by `lib/serper.js`
- `GOOGLE_SERVICE_ACCOUNT_EMAIL`, `GOOGLE_SERVICE_ACCOUNT_PRIVATE_KEY`, `GOOGLE_SHEET_ID` — used by `lib/sheets.js`

## Architecture

Next.js 14 App Router project. The lead-generation flow is a 3-stage pipeline, each stage in its own `lib/` module, wired together by two API routes:

1. **`app/api/search/route.js`** (`POST /api/search`) — orchestrates the search:
   - `lib/ai.js` (`extractDataFromPrompt`) sends the user's free-text prompt to OpenRouter (streaming chat completion) and parses a JSON object out of the response: `{ location, category, intent, requires_missing_website }`. Response parsing strips markdown code fences and regex-extracts the first `{...}` block before `JSON.parse`.
   - `lib/serper.js` (`searchMapsViaSerper`) queries Serper.dev's Google Maps endpoint with `category` + `location`.
   - The route then filters results in-memory: if `requires_missing_website` is true, only leads with no `website`/`link` field survive ("Website Development Mode"); otherwise all results pass through ("Broad Mode").
   - Leads are reshaped into `{ name, address, phone, rating, website }` before being returned. Phone numbers are prefixed with `'` so Google Sheets doesn't reformat them as numbers.

2. **`app/api/save/route.js`** (`POST /api/save`) — takes the leads array (already shaped by `/api/search`) plus `category`/`location`, and calls `lib/sheets.js` (`saveLeadsToGoogleSheet`) to append rows to a Google Sheet via a service account (JWT auth). It checks `Sheet1!A1:H1` and writes a header row on first use, then appends data rows with a timestamp column.

3. **`app/page.jsx`** is the only frontend page (client component): a single prompt textarea drives `handleSubmit` → `/api/search`, then a "Push to Sheets" button drives `handleSave` → `/api/save`. All state (results, loading, save status) is local `useState`, no global store.

### Error handling convention

Each `lib/*.js` module wraps failures in a module-specific error factory (`buildOpenRouterError`, `buildSerperError`, `buildSheetsError`) that prefixes the message (e.g. `"Error: Failed Serper API call - ..."`) and attaches `.status`/`.details`. `app/api/search/route.js` has a matching `normalizeStatus`/`buildValidationError` pair. When adding a new external call, follow this same pattern rather than throwing raw errors, since the API routes and frontend rely on `error.message` / `error.details` to surface meaningful text to the user.
