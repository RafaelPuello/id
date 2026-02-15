# CLAUDE.md - DigiDex ID Service

**Service-specific guides:**
- **Backend**: See `/id/backend/CLAUDE.md` for detailed Django backend architecture, JWT setup, testing, and API details
- **Frontend**: See `/id/frontend/CLAUDE.md` for React/Vite frontend architecture and component details

This file provides overall service context.

## Project Overview

DigiDex ID Service is the identity/authentication microservice for the DigiDex platform. It provides user account management with multi-factor authentication (MFA), social account providers (OAuth), and passwordless authentication via WebAuthn/passkeys. The service uses a headless django-allauth backend with a React SPA frontend.

Production URL: https://id.digidex.bio

## Commands

### Frontend (React/Vite)
```bash
cd frontend
npm install
npm run dev              # Development server on port 5173
npm run build           # Production build to dist/
npm run lint            # ESLint
npm test                # Run Jest unit tests
npm run test:watch      # Jest watch mode
npm run test:coverage   # Jest with coverage report
npm run test:e2e        # Playwright E2E tests
npm run test:e2e:ui     # Playwright E2E tests (UI mode)
npm run test:e2e:debug  # Playwright E2E tests (debug mode)
```

### Backend (Django)
```bash
cd backend
pip install -r requirements.txt -r requirements-dev.txt
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
pytest                  # Run all tests
pytest identity/        # Run tests for identity app
pytest -v --cov         # With coverage report
```

### Docker Compose
```bash
# Development (with hot reloading)
docker compose -f compose.yaml -f compose.override.yaml up

# Production
docker compose -f compose.yaml up
```

Development includes a mailcatcher service at `localhost:1080` for email testing.

## Architecture Overview

**Backend**: Django 6.0 with django-allauth headless API, JWT-based authentication (RS256), custom email-based User model, MFA support (TOTP/WebAuthn/recovery codes). See `/id/backend/CLAUDE.md` for detailed configuration, API endpoints, and authentication flow.

**Frontend**: React 19 + Vite 7 SPA with AuthContext for state management, React Router v7 for routing, and direct integration with headless allauth API. See `/id/frontend/CLAUDE.md` for component organization, route guards, and auth hooks.

## Backend Integration

The frontend uses relative paths and expects to run behind a reverse proxy (Traefik) with the backend at the same origin. In development, Traefik routes:
- `/api`, `/accounts`, `/_allauth` → backend:8000
- All other paths → frontend:5173

## Environment Setup

### Backend
```bash
cd backend
# Required environment variables (see .env.dev or .env.prod):
# - DJANGO_SECRET_KEY: Secret key for Django
# - DATABASE_URL: PostgreSQL connection string
# - DJANGO_DEBUG: Set to "True" for development
# - DJANGO_ALLOWED_HOSTS: Comma-separated list of allowed hosts

# Setup:
python manage.py migrate           # Create database tables
python manage.py runserver 0.0.0.0:8000
```

### Frontend
- Runtime config fetched from backend (`/_allauth/browser/v1/config`), not build-time env vars
- Frontend expects backend at same origin (via Traefik reverse proxy in development)
- `.env` file optional; most config comes from backend at runtime

### Styles (Git Submodule)
Frontend imports SCSS from a git submodule:
```bash
# The submodule is auto-initialized with typical git clone
# If styles are missing, initialize explicitly:
git submodule update --init --recursive

# Frontend then imports styles via symlink: src/styles → /shared/styles
```
