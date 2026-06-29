# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Polla Mundial 2026** — a single-page World Cup prediction game. Users register by name, then predict scores for upcoming matches. Predictions lock automatically when the match starts.

## Stack

- **Frontend**: Single `index.html` with embedded CSS and vanilla JavaScript — no build step, no bundler, no framework.
- **Backend**: [Supabase](https://supabase.com/) (Postgres + REST API), loaded from CDN. Credentials are hardcoded in the script block (`SUPABASE_URL`, `SUPABASE_KEY`).

## Running Locally

Open `index.html` directly in a browser (file://) or serve it with any static server:

```
npx serve .
# or
python -m http.server 8080
```

No compilation or installation needed.

## Supabase Schema

Three tables (managed via the Supabase dashboard, not migrations in this repo):

| Table | Key columns |
|-------|-------------|
| `usuarios` | `id` (uuid), `nombre` (text, unique) |
| `partidos` | `id`, `equipo_1`, `equipo_2`, `fecha` (date), `hora` (time), `fase`, `bandera_1`, `bandera_2` (flag image URLs) |
| `predicciones` | `usuario_id`, `partido_id`, `goles_equipo_1`, `goles_equipo_2` — unique on `(usuario_id, partido_id)` |

Upserts use `onConflict: "usuario_id,partido_id"` so re-submitting a prediction updates rather than errors.

## Architecture

All logic lives in `index.html`'s `<script>` block. Key global state:

- `listaPartidos` — array of match objects fetched from Supabase, sorted by `fecha`/`hora`
- `usuarioActual` — `{ id, nombre }` of the logged-in user, persisted to `localStorage` under keys `pollita_usuario_id` / `pollita_usuario_nombre`
- `prediccionesUsuario` — map of `partido_id → prediction` for the current user

Key functions:

- `registrarUsuario()` — upserts by name via `obtenerOCrearUsuario()`, then loads predictions and re-renders matches
- `cargarPartidos()` — fetches all matches, calls `renderizarTarjetaPartido()` for each
- `abrirModal(index)` — populates and shows the prediction modal for a given match
- `guardarPrediccion()` — upserts to `predicciones`, updates local cache, re-renders
- `actualizarEstadoBotonesPartidos()` — called every 30 s via `setInterval` to lock buttons when match time passes

## Prediction Deadline Logic

`partidoAbiertoParaPrediccion(partido)` constructs a local `Date` from the match's `fecha` (YYYY-MM-DD) and `hora` (HH:MM:SS) fields using **local time** (no UTC conversion). Predictions are closed as soon as `new Date() >= matchStartTime`.
