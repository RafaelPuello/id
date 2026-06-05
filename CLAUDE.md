# CLAUDE.md - DigiDex ID Service

Authentication/identity microservice: user accounts, MFA, OAuth, WebAuthn/passkeys via Django-allauth (headless) + React/Vite frontend.

**Detailed guides:**
- **Backend**: See `/id/backend/CLAUDE.md` for Django architecture, JWT (RS256), authentication flow
- **Frontend**: See `/id/frontend/CLAUDE.md` for React/Vite SPA, AuthContext, routing, components

## Quick Setup

Backend: `cd backend && pip install -r requirements.txt -r requirements-dev.txt && python manage.py migrate && python manage.py runserver 0.0.0.0:8001`

Frontend: `cd frontend && npm install && npm run dev`

Docker: `docker compose -f compose.yaml -f compose.override.yaml up` (includes mailcatcher at localhost:1080)

See README.md for full setup and command reference.

## Architecture

- **Backend**: Django 6.0 + django-allauth headless API (no sessions); JWT (RS256); email-based User model; MFA (TOTP/WebAuthn)
- **Frontend**: React 19 + Vite 7 SPA; AuthContext state management; React Router v7 with basename routing
- **Auth flow**: Frontend calls `/_allauth/browser/v1/*` endpoints; receives JWT tokens; stores in localStorage; sends in Authorization header
- **Routing**: Traefik forwards `/id/*` without stripping; Vite `base: '/id/'`; React Router `basename: '/id'`
- **Network**: Both on `digidex-net` external Docker network (Traefik discovery)

## Conventions

- API: Headless django-allauth endpoints (not traditional session-based)
- Frontend: Pure JWT client; localStorage token persistence; localStorage-based cross-tab logout detection
- Auth state: Discriminated union type (loading | authenticated | unauthenticated)
- Shared styling: `src/styles` symlinks to `/shared/styles`

## Gotchas

- **Port 8001**: Non-standard Django port. Don't confuse with CMS (8000) or App (8000).
- **Sessions vs JWT**: Currently uses Django sessions for django-allauth refresh token validation (interim solution). Future direction: eliminate sessions entirely.
- **Traefik path forwarding**: Routes `/id/*` WITHOUT stripping. Vite and React Router must handle full path themselves.
- **Runtime config**: Frontend fetches app config from `/_allauth/app/v1/config` at runtime, not build-time env vars.
- **Styles submodule**: Frontend imports from git submodule (`src/styles` → `/shared/styles`). If missing, initialize with `git submodule update --init --recursive`.

## Key Files

- Backend config: `id/backend/config/settings.py`, `id/backend/config/keys/jwt_public_key.pem` (distributed via Docker secret/bind mount)
- Backend API: `id/backend/identity/models.py` (CustomUser), `id/backend/config/urls.py` (allauth endpoints)
- Frontend auth: `id/frontend/src/auth/AuthContext.jsx` (state machine), `id/frontend/src/hooks/useAuth.jsx`
- Frontend routing: `id/frontend/src/App.jsx` (basename: '/id'), route guards
