# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Layout

The repository root is `4Season-project/` and has no manifest of its own. It holds two independent workspaces that share no tooling:

- `frontend/` — Vite + React + TypeScript. **All npm commands must be run from here**; running them at the root will fail.
- `backend/` — AWS Chalice (serverless Python). **All Python work requires activating `backend/.venv` first.**

The two are unrelated at the tooling level — separate package managers, separate dependency files, separate dev servers. Neither directory knows about the other.

## Frontend

Run from `frontend/`:

```bash
npm run dev      # Vite dev server with HMR
npm run build    # tsc -b (typecheck, project references) then vite build -> dist/
npm run lint     # oxlint
npm run preview  # serve the built dist/
```

There is no test runner installed and no test script. If tests are needed, one has to be added (Vitest is the natural fit for this Vite setup).

Toolchain: Node v26.5.1 via nvm, npm 11. There is no `.nvmrc`, so the nvm default version is what gets used.

`npm run build` runs `tsc -b` first, so a type error fails the build before Vite ever runs. To typecheck alone, use `npx tsc -b`; add `--force` to bypass the incremental cache in `node_modules/.tmp/`.

## Backend

Every command below assumes the virtualenv is active. It is not activated automatically, and `python`/`pip`/`chalice` all resolve to the wrong thing without it:

```bash
cd backend
source .venv/bin/activate    # prompt gains a (.venv) prefix; `deactivate` to exit
chalice local --port 8000    # local dev server -> http://127.0.0.1:8000
```

`backend/.venv` runs **Python 3.12.13**, deliberately older than the system's 3.14 — some libraries do not yet publish wheels for 3.14. Note that two separate 3.12.13 installs exist on this machine: `~/.local/bin/python3.12` (uv-managed, wins on PATH) and `/usr/bin/python3.12` (apt/deadsnakes). The venv was built from the uv one. Recreating it with a bare `python3.12 -m venv` picks the uv build again, which is consistent — but be aware the two are not the same files.

Layout of the Chalice app (generated from the `rest-api` template):

- `app.py` — the `Chalice` instance and all `@app.route` handlers. This is the deployment entry point.
- `chalicelib/` — everything else. Chalice only ships `app.py` plus this package to Lambda, so shared logic must live here, not in loose top-level modules.
- `.chalice/config.json` — app name and per-stage deployment config. Required; the app will not deploy without it.

Two dependency files with different jobs, and neither is currently in sync with the venv:

- `requirements.txt` is what gets bundled into the Lambda deployment package. **It is empty**, while `psycopg2-binary`, `python-dotenv`, and `sqlmodel` are installed in the venv. Anything the app imports at runtime must be added here or deployment will fail at import time even though local dev works.
- `requirements-dev.txt` pins `chalice` and `pytest`. `pytest` is not installed yet, so `tests/test_app.py` cannot run until `pip install -r requirements-dev.txt`.

## TypeScript configuration

`tsconfig.json` is a solution file with no sources of its own — it only references `tsconfig.app.json` (covers `src/`) and `tsconfig.node.json` (covers `vite.config.ts`). Compiler options must be edited in the referenced file that owns the code in question, not in the root config.

Several options in `tsconfig.app.json` change how code must be written:

- `verbatimModuleSyntax` — type-only imports must use `import type`.
- `erasableSyntaxOnly` — no enums, namespaces, or constructor parameter properties.
- `allowImportingTsExtensions` — imports carry their extension (`import App from './App.tsx'`), matching the existing style in `src/main.tsx`.
- `noUnusedLocals` / `noUnusedParameters` — unused bindings are build failures, not warnings.

## Assets

Two distinct paths, and the choice determines the reference syntax:

- `src/assets/` — imported in TS/TSX (`import heroImg from './assets/hero.png'`), processed and content-hashed by Vite.
- `public/` — copied verbatim, referenced by absolute URL. `public/icons.svg` is an SVG sprite consumed as `<use href="/icons.svg#github-icon">`; adding a new icon means adding a symbol to that sprite, not adding a file.

## Current state

Both workspaces are still at their generated defaults and nothing connects them yet.

`frontend/src/` is the unmodified Vite + React scaffold — `App.tsx` is the template landing page with the counter demo, and there is no routing, state management, or data layer. `frontend/dist/` holds a build of that scaffold and is disposable.

`backend/app.py` is the Chalice starter with a single `GET /` returning `{"hello": "world"}`; `chalicelib/` is empty. Despite `psycopg2-binary` and `sqlmodel` being installed, there is no database connection, model, or config loading yet.

The frontend makes no calls to the backend. Wiring them up means choosing a dev-time approach for cross-origin requests (Vite proxy, or CORS on the Chalice routes) — that decision is still open.

The repository is not a git repository yet (`git init` has not been run).
