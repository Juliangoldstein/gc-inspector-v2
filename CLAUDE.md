# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

GC Inspector is a single-file Progressive Web App for construction-site inspection of the building
"Catamarán 12" (Goldstein Construcciones). Inspectors open architectural plan PDFs, drop observation
markers on them, and generate certification reports. The entire app is `index.html` (~2900 lines):
inline `<style>` blocks plus one inline `<script type="module">` that holds all logic. Everything else
in the repo is PWA chrome (`manifest.json`, favicons, `icon-*.png`, `logo.png`).

UI text, comments, and identifiers are in **Spanish (es-AR)** — e.g. `pantalla` (screen), `pestañas`
(tabs), `planos` (plans), `observaciones` (observations). Match this convention when editing.

## Build / run / deploy

There is **no build system, bundler, linter, or test suite** — nothing to compile and no `package.json`.

- **Run locally:** serve the directory over HTTP (e.g. `python -m http.server`) and open `index.html`.
  Opening via `file://` breaks ES-module imports and Firebase auth. Note that Google sign-in and the
  Cloud Functions expect authorized origins, so full auth flows generally only work on the deployed
  domain unless `localhost` is whitelisted in Firebase.
- **Dependencies** are loaded from CDNs at runtime (no local install): Firebase 10.12.0 and pdf.js
  5.4.149 as ES modules; jsPDF + jspdf-autotable as UMD `<script src>`. The CSP `<meta>` tag in
  `<head>` allowlists these hosts — **adding a new CDN or backend endpoint requires updating both the
  import and the matching CSP directive** (`script-src` / `connect-src` / `img-src` / etc.).
- **Deploy:** GitHub Pages serves this repo directly to the domain in `CNAME`
  (`inspector.goldsteinconstrucciones.com`). A commit to the default branch is the deploy — there is
  no staging step. Git history is almost entirely "Update index.html" commits.

## Backend architecture (Firebase + two Cloud Functions)

The browser talks to Firebase directly (config is inline near the top of the script — the Firebase
`apiKey` is public by design, gated by Firestore Security Rules that live outside this repo):

- **Auth:** Google sign-in popup. `onAuthStateChanged` (~line 600) is the entry point. A first-time
  user is written to `usuarios/{uid}` with `rol: 'inspector'`, `autorizado: false` and parked on the
  `espera` screen until an admin flips `autorizado`. Roles gate the UI: **`admin`** (user management,
  data-migration helpers, delete plans), **`inspector`** (default), **`lector`** (read-only — add/edit
  controls hidden). Check `currentUserData?.rol` when adding gated features.
- **Firestore collections:** `usuarios`, `planos`, `observaciones`, `certificados`, `logs_errores`
  (client-side error logs auto-written for admin review). Real-time UI uses `onSnapshot` subscriptions;
  remember to unsubscribe (see the logout handler and the `pestañas`/`planosUnsubscribe` cleanup).
- **Storage:** photo attachments for observations.
- **Two Cloud Functions proxy secrets — never put credentials back in the client:**
  - `obtenerTokenDropbox` returns a short-lived (~4h) Dropbox access token. The Dropbox App Secret and
    refresh token live in Firebase Secret Manager; the browser only ever sees the temporary token
    (cached in `getDropboxToken`, ~line 645). Plan PDFs are stored in Dropbox, not Firebase.
  - `proxyAppsScript` forwards writes to a Google Apps Script / Google Sheet, but only after verifying
    the caller's Firebase ID token and their authorization in Firestore. Call it via `llamarAppsScript`
    (~line 69), which attaches the ID token — do not fetch the Apps Script or Sheet directly.

## Front-end structure

Seven screens (`pantalla-login`, `-espera`, `-home`, `-planos`, `-inspector`, `-calidad`, `-ppi`) live
in the `<body>` and are toggled by the `mostrar(name)` helper via the `oculto` class — this is the
entire "routing" mechanism; there is no framework or router. Three functional modules:

- **Inspector** (`iniciarPantallaPlanos` → `pantalla-inspector`): browse the Dropbox plan folder, open
  PDFs rendered with pdf.js into a canvas, then a tab system (the `pestañas` Map, `pestañaActiva`) with
  zoom, drag-to-pan, marker placement, an observation panel and observation-list panel, "Certificar"
  (sync the active plan's state), and jsPDF report generation.
- **Calidad** (`iniciarPantallaCalidad` → `pantalla-calidad`, the large block from ~line 1835): quality
  module, seeds its reference "empresas" into Firestore on first admin login (`sembrarEmpresasSiFalta`).
- **PPI** (`iniciarPantallaPPI` → `pantalla-ppi`): the Plan de Puntos de Inspección — see its own
  section below. Deliberately **independent of the other two**: it never reads `observaciones` or
  `certificados`, and a pending observation does not gate any PPI approval.

The nomenclature constants near the top — `DISCIPLINAS`, `NIVELES`, `BLOQUES`, `UNIDADES`, and the
`CROQUIS DE CALIDAD` data — encode the real building's structure (blocks, floors, unit codes) and the
plan-file naming scheme. They are the source of truth for parsing plan filenames and building unit
pickers; a comment notes they are meant to migrate to a shared `config/estructura` Firestore doc later.

## Observation lifecycle (business rules)

- **A resolved observation must not be reopened.** The expected workflow is to leave the resolved
  observation as-is and create a **new** observation for anything that reappears — do not flip a
  `Resuelta` obs back to `Pendiente` as the normal path. The save handler still clears `resueltaEn` to
  `null` if an obs ever leaves the `Resuelta` state (defensive, so no stale "resolution date" leaks into
  the Sheet or certificates), but that is a safety net, not the intended flow.
- **`notaResolucion` is always preserved.** Never wipe the resolution note on save (not even when an obs
  leaves `Resuelta`). It is kept as historical record.

## PPI — Plan de Puntos de Inspección (business rules)

`RUBROS_PPI` (near the top, after `UNIDADES`) is the source of truth: 12 rubros across 3 phases, each
with its checklist `items`. It transcribes the paper PPI — **do not reword an item without checking
the source PDFs**, since inspectors read these in the field.

- **Hard blocking, strictly linear.** Rubro *N* cannot be loaded until *N-1* is released. `ppiBloqueo()`
  returns `null` (open) or the reason string; it is the single place that encodes the chain.
- **`ambito` is per-rubro, not per-phase.** Losa/Columnas/Vigas (1-3) and Contrapiso (8) are `'nivel'`;
  the rest are `'unidad'`. Every unit-scoped rubro also gets **`EC`** (espacio común) as one more unit
  of the level (`ppiUnidadesDeNivel` appends it).
- **The 7→8 gate.** Because ámbitos alternate, Contrapiso (level-scoped) requires Plomería approved in
  **all** units of the level, EC included — you cannot pour a floor's contrapiso with one unit's pipes
  still open. This is the one non-obvious edge; `ppiBloqueo` handles it generically.
- **`no_aplica` releases the chain like `aprobado`.** Without it, a rubro that doesn't exist in a given
  ámbito (Carpintería de Aluminio in Bauleras) would block that ámbito forever.
- **Approval requires the full checklist** and records a signature (`aprobadoPor`, `aprobadoPorNombre`,
  `aprobadoEn`). Only `admin` can `Reabrir`; it warns when the successor is already approved but allows
  it, leaving `reabiertoPor`/`reabiertoEn`.
- **Firestore:** collection `ppi`, one doc per (ámbito, rubro) with a deterministic ID —
  `T-02-losa` (level) / `T-02-A-yeso`, `T-02-EC-plomeria` (unit). Writes are `setDoc(..., {merge:true})`.
  Roles: `inspector`/`admin` write, `lector` read-only.

## Security conventions to preserve

Because all logic is inline, the CSP must keep `script-src 'unsafe-inline'`, so the CSP alone does **not**
stop injected `onerror=`/`onclick=` in `innerHTML`. The real XSS defense is the `escapeHtml()` family
(~line 512) — **always escape user/Firestore-derived strings before interpolating them into `innerHTML`.**
pdf.js is pinned to ≥5.x deliberately (older versions had a PDF-triggered RCE, CVE-2024-4367); do not
downgrade it.
