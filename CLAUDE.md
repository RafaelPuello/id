# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DigiDex ID Service is the identity/authentication microservice for the DigiDex platform. It provides user account management with multi-factor authentication (MFA), social account providers (OAuth), and passwordless authentication via WebAuthn/passkeys. The service uses a headless django-allauth backend with a React SPA frontend.

Production URL: https://id.digidex.bio

## Commands

### Frontend (React/Vite)
```bash
cd frontend
npm install
npm run dev      # Development server on port 5173
npm run build    # Production build to dist/
npm run lint     # ESLint
```

### Backend (Django)
```bash
cd backend
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

### Docker Compose
```bash
# Development (with hot reloading)
docker compose -f compose.yaml -f compose.override.yaml up

# Production
docker compose -f compose.yaml up
```

Development includes a mailcatcher service at `localhost:1080` for email testing.

## Architecture

### Backend (Django 6.0)

**URL Structure:**
- `/api/` - Ninja JWT API (token endpoints)
- `/accounts/` - django-allauth standard URLs
- `/_allauth/browser/v1/*` - Headless allauth API (used by frontend)

**Key Configuration (`backend/config/settings.py`):**
- Custom User model with email as identifier (no username)
- `HEADLESS_ONLY = True` - No server-rendered auth pages
- MFA types: TOTP, recovery codes, WebAuthn
- Passkey login enabled, passkey signup disabled
- Signups currently disabled (`IdentityAdapter.is_open_for_signup` returns False)

**Apps:**
- `identity/` - Custom User model and account adapter

### Frontend (React 19 + Vite 7)

**Authentication Flow:**
- `AuthContext` (`/src/auth/AuthContext.jsx`) - Central auth state provider, fetches initial state on mount
- Custom hooks (`/src/auth/hooks.jsx`) - `useAuth()`, `useConfig()`, `useUser()`, `useAuthStatus()`
- `allauth` API client (`/src/lib/allauth.jsx`) - All backend communication
- Auth changes broadcast via `allauth.auth.change` custom events

**Route Guards:**
- `<AuthenticatedRoute>` - Protects authenticated-only pages
- `<AnonymousRoute>` - Protects anonymous-only pages (login, signup)
- `<AuthChangeRedirector>` - Handles auth state transitions

**Feature Directories:**
```
src/
├── account/        # Login, signup, password, email management
├── auth/           # Auth context, hooks, route guards
├── mfa/            # TOTP, WebAuthn, Recovery Codes
├── socialaccount/  # OAuth providers
├── usersessions/   # Active session management
├── components/     # Reusable components (forms/, layout/, nav/)
├── lib/            # allauth API, CSRF handling, utilities
```

**Key Patterns:**
- CSRF tokens sent via `X-CSRFToken` header (`/src/lib/django.jsx`)
- Path alias: `@` resolves to `/src`
- Runtime config fetched from backend (`/_allauth/browser/v1/config`), access via `useConfig()`
- TypeScript migration in progress: form components use `.tsx`, feature components use `.jsx`

**Auth Flow Constants (`/src/lib/allauth.jsx`):**
- `Flows.LOGIN`, `Flows.SIGNUP`, `Flows.VERIFY_EMAIL`
- `Flows.MFA_AUTHENTICATE`, `Flows.MFA_REAUTHENTICATE`
- `Flows.LOGIN_BY_CODE`, `Flows.PASSWORD_RESET_BY_CODE`
- `Flows.PROVIDER_SIGNUP`

## Backend Integration

The frontend uses relative paths and expects to run behind a reverse proxy (Traefik) with the backend at the same origin. In development, Traefik routes:
- `/api`, `/accounts`, `/_allauth` → backend:8000
- All other paths → frontend:5173

## Environment Files

- `backend/.env.dev` / `backend/.env.prod` - Django settings (DATABASE_URL, SECRET_KEY, DEBUG)
- `frontend/.env` - Frontend environment variables
