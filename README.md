# DigiDex ID Service

The DigiDex ID Service is a modern authentication and identity management microservice built with Django and React. It provides JWT-based headless authentication with support for multi-factor authentication (MFA), social OAuth providers, and WebAuthn/passkey authentication.

**Production URL**: https://id.digidex.bio

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Quick Start](#quick-start)
- [Development Setup](#development-setup)
- [Configuration](#configuration)
- [Testing](#testing)
- [Production Deployment](#production-deployment)
- [Contributing](#contributing)
- [Troubleshooting](#troubleshooting)

## Architecture Overview

### Technology Stack

**Backend**:
- Django 6.0
- django-allauth (headless API mode)
- django-ninja (REST API framework)
- django-ninja-jwt (JWT token management)
- PostgreSQL (production) / SQLite (development)

**Frontend**:
- React 19
- Vite 7
- React Router v7
- JWT-based authentication (no cookies)

### Authentication Flow

The service implements a **stateless JWT authentication model** with the following characteristics:

1. **App Client Only**: Uses django-allauth's app client (`/_allauth/app/v1/*`), not the browser client
2. **JWT Tokens**: RS256 asymmetric encryption with separate access/refresh tokens
3. **Token Storage**: localStorage (access_token, refresh_token)
4. **Token Expiration**:
   - Access tokens: 5 minutes
   - Refresh tokens: 24 hours (rotating)
5. **CSRF-Safe**: JWT in Authorization headers eliminates CSRF vulnerabilities
6. **No Server Sessions**: Uses signed cookies for django-allauth internals only

### Key Features

- **Email-based authentication** (no username required)
- **Multi-factor authentication (MFA)**:
  - TOTP (Time-based One-Time Password)
  - WebAuthn/Passkeys
  - Recovery codes
- **Social OAuth providers**:
  - Google
  - GitHub
- **Passwordless authentication**:
  - Email magic links
  - WebAuthn login
- **Session management**: View and revoke active sessions
- **Account management**: Email verification, password reset, profile settings

### Service Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Traefik (Reverse Proxy)             │
│  Routes: /api, /accounts, /_allauth → backend:8000      │
│          /* → frontend:5173                             │
└─────────────────────────────────────────────────────────┘
                            │
           ┌────────────────┴────────────────┐
           │                                 │
    ┌──────▼──────┐                  ┌──────▼──────┐
    │   Backend   │                  │  Frontend   │
    │  Django     │                  │  React SPA  │
    │  Port 8000  │                  │  Port 5173  │
    └─────────────┘                  └─────────────┘
           │
    ┌──────▼──────┐
    │  PostgreSQL │
    │  Database   │
    └─────────────┘
```

## Quick Start

### Prerequisites

- **Python 3.11+** (backend)
- **Node.js 20+** (frontend)
- **Docker & Docker Compose** (optional, for containerized development)
- **PostgreSQL** (production) or SQLite (development)

### Local Development (Fastest)

```bash
# 1. Clone the repository
git clone <repository-url>
cd digidex/id

# 2. Backend setup
cd backend
pip install -r requirements.txt -r requirements-dev.txt
python manage.py migrate
python manage.py runserver 0.0.0.0:8000

# 3. Frontend setup (in a new terminal)
cd ../frontend
npm install
npm run dev

# 4. Access the application
# Frontend: http://localhost:5173
# Backend API: http://localhost:8000
```

### Docker Compose (Full Stack)

```bash
# Development with hot reloading
docker compose -f compose.yaml -f compose.override.yaml up

# Production build
docker compose -f compose.yaml up
```

The Docker Compose setup includes:
- Backend (Django)
- Frontend (React/Vite)
- PostgreSQL database
- Traefik reverse proxy
- Mailcatcher (development only, port 1080)

## Development Setup

### Backend Setup

1. **Create a virtual environment** (recommended):
   ```bash
   cd backend
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

2. **Install dependencies**:
   ```bash
   pip install -r requirements.txt -r requirements-dev.txt
   ```

3. **Set up environment variables**:
   ```bash
   # Copy the example file
   cp .env.example .env.dev

   # Edit .env.dev with your settings
   # Required: DJANGO_SECRET_KEY, DATABASE_URL
   ```

4. **Run migrations**:
   ```bash
   python manage.py migrate
   ```

5. **Create a superuser** (optional, for Django shell access):
   ```bash
   python manage.py createsuperuser
   # Note: Django admin is disabled in HEADLESS_ONLY mode
   ```

6. **Start the development server**:
   ```bash
   python manage.py runserver 0.0.0.0:8000
   ```

### Frontend Setup

1. **Install dependencies**:
   ```bash
   cd frontend
   npm install
   ```

2. **Initialize git submodules** (for shared styles):
   ```bash
   # From repository root
   git submodule update --init --recursive
   ```

3. **Start the development server**:
   ```bash
   npm run dev
   ```

The frontend will be available at `http://localhost:5173`.

### Reverse Proxy Setup (Development)

For local development with proper routing, you can use Traefik:

```bash
# From id/ directory
docker compose -f compose.yaml -f compose.override.yaml up traefik

# Or use the full stack
docker compose -f compose.yaml -f compose.override.yaml up
```

Traefik will route requests:
- `http://localhost:10000/api/*` → Backend
- `http://localhost:10000/_allauth/*` → Backend
- `http://localhost:10000/accounts/*` → Backend
- `http://localhost:10000/*` → Frontend

## Configuration

### Backend Configuration

All configuration is in `/backend/config/settings.py` and controlled via environment variables.

#### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DJANGO_SECRET_KEY` | Django secret key (generate with `django.core.management.utils.get_random_secret_key()`) | `your-secret-key-here` |
| `DATABASE_URL` | Database connection string | `postgresql://user:pass@localhost/dbname` |

#### Optional Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DJANGO_DEBUG` | Enable debug mode | `False` |
| `DJANGO_ALLOWED_HOSTS` | Comma-separated allowed hosts | `localhost,testserver` |
| `JWT_PRIVATE_KEY_PATH` | Path to JWT RSA private key | `config/keys/jwt_private_key.pem` |
| `JWT_PUBLIC_KEY_PATH` | Path to JWT RSA public key | `config/keys/jwt_public_key.pem` |

#### JWT Key Management

**Development**:
- Development keys are checked into git at `/backend/config/keys/`
- These are for local testing only and should NEVER be used in production

**Production**:
- Generate production keys with:
  ```bash
  # Generate private key
  openssl genpkey -algorithm RSA -out jwt_private_key.pem -pkeyopt rsa_keygen_bits:2048

  # Extract public key
  openssl rsa -pubout -in jwt_private_key.pem -out jwt_public_key.pem
  ```
- Store keys securely (secrets manager, environment variables, mounted volumes)
- Set environment variables to point to key files
- NEVER commit production keys to version control

#### Key Django Settings

```python
# Headless mode (API-only, no server-rendered pages)
HEADLESS_ONLY = True

# App client only (JWT-based, disables browser client)
HEADLESS_CLIENTS = ["app"]

# JWT Configuration
HEADLESS_TOKEN_STRATEGY = "allauth.headless.tokens.strategies.jwt.JWTTokenStrategy"
HEADLESS_JWT_ALGORITHM = "RS256"
HEADLESS_JWT_ACCESS_TOKEN_LIFETIME = 300  # 5 minutes
HEADLESS_JWT_REFRESH_TOKEN_LIFETIME = 86400  # 24 hours

# Custom User model (email-based, no username)
AUTH_USER_MODEL = "identity.User"
ACCOUNT_USER_MODEL_USERNAME_FIELD = None
ACCOUNT_EMAIL_REQUIRED = True
ACCOUNT_AUTHENTICATION_METHOD = "email"

# Signed cookie sessions (no database sessions)
SESSION_ENGINE = "django.contrib.sessions.backends.signed_cookies"
```

### Frontend Configuration

The frontend fetches runtime configuration from the backend at `/_allauth/app/v1/config`. No build-time environment variables are required.

**Key configuration points**:
- API base URL: `/_allauth/app/v1`
- Token storage: `localStorage`
- CSRF: Not required (JWT authentication)

## Testing

### Backend Tests

```bash
cd backend

# Run all tests
pytest

# Run with verbose output
pytest -v

# Run with coverage
pytest --cov

# Run specific test file
pytest identity/tests.py

# Run specific test class
pytest identity/tests.py::TestUserModel

# Run specific test method
pytest identity/tests.py::TestUserModel::test_user_creation

# Run tests by marker
pytest -m integration  # Only integration tests
pytest -m "not slow"   # Exclude slow tests
```

**Test markers**:
- `slow`: Tests that take significant time
- `integration`: Integration tests (require database, external services)

**Coverage reports**:
Coverage reports are generated in `htmlcov/`. Open `htmlcov/index.html` in a browser to view detailed coverage.

### Frontend Tests

```bash
cd frontend

# Unit tests (Jest)
npm test                 # Run all tests
npm run test:watch       # Watch mode
npm run test:coverage    # With coverage report

# E2E tests (Playwright)
npm run test:e2e         # Headless mode
npm run test:e2e:ui      # Interactive UI mode
npm run test:e2e:debug   # Debug mode with browser DevTools

# Linting
npm run lint
```

**E2E Test Documentation**:
See `/id/E2E_TESTING.md` for detailed E2E test coverage and status.

### Test Configuration

**Backend** (`pytest.ini`):
```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
python_files = tests.py test_*.py
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
```

**Frontend** (`package.json`):
- Jest for unit tests
- Playwright for E2E tests
- ESLint for code quality

## Production Deployment

### Build Process

**Backend**:
```bash
cd backend
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
```

**Frontend**:
```bash
cd frontend
npm install
npm run build
# Output: dist/
```

### Docker Images

Build production images:
```bash
# Backend
docker build -t digidex-id-backend:latest -f backend/Dockerfile backend/

# Frontend
docker build -t digidex-id-frontend:latest -f frontend/Dockerfile frontend/
```

### Environment Checklist

- [ ] `DJANGO_DEBUG=False`
- [ ] `DJANGO_SECRET_KEY` set to secure random value
- [ ] `DATABASE_URL` configured for PostgreSQL
- [ ] `DJANGO_ALLOWED_HOSTS` includes production domain
- [ ] `JWT_PRIVATE_KEY_PATH` and `JWT_PUBLIC_KEY_PATH` point to production keys
- [ ] Production JWT keys generated and stored securely (NEVER commit to git)
- [ ] CORS origins configured in `CORS_ALLOWED_ORIGINS`
- [ ] HTTPS/TLS enabled
- [ ] Database backups configured
- [ ] Monitoring and logging enabled
- [ ] Rate limiting configured (if applicable)

### Security Considerations

1. **JWT Private Key**: MUST be kept secret and NEVER exposed. Compromise requires key rotation and user re-authentication.
2. **Token Expiration**: Short-lived access tokens (5 min) limit damage from token theft.
3. **Refresh Token Rotation**: Each refresh issues a new refresh token, invalidating the old one.
4. **HTTPS Required**: Tokens transmitted over unencrypted connections can be intercepted.
5. **Content Security Policy**: Configure CSP headers in reverse proxy (Traefik).
6. **Rate Limiting**: Implement rate limiting on authentication endpoints.
7. **Secrets Management**: Use a secrets manager (AWS Secrets Manager, HashiCorp Vault, etc.) for production.

## Contributing

### Code Style

**Backend**:
- Follow PEP 8
- Use `ruff` for linting and formatting (see `/cms/backend/CLAUDE.md` for conventions)
- Type hints required for new code (`mypy`)

**Frontend**:
- ESLint with React plugins
- Functional components only
- TypeScript for new components (`.tsx`)
- Hooks for state management

### Workflow

1. Create a feature branch: `git checkout -b feature/description`
2. Make changes
3. Run tests: `pytest` (backend), `npm test` (frontend)
4. Commit with conventional commits: `feat:`, `fix:`, `docs:`, etc.
5. Push and create pull request

### Branching Strategy

- `main`: Production-ready code
- `feature/*`: New features
- `fix/*`: Bug fixes
- `docs/*`: Documentation updates

## Troubleshooting

### Common Issues

#### Backend Issues

**Problem**: `django.core.exceptions.ImproperlyConfigured: JWT private key file not found`

**Solution**: Ensure JWT keys exist:
```bash
cd backend/config/keys
ls jwt_private_key.pem jwt_public_key.pem
```

If missing, copy from another environment or generate new keys (development only).

---

**Problem**: `OperationalError: FATAL:  password authentication failed`

**Solution**: Check `DATABASE_URL` in `.env.dev`:
```bash
# Correct format:
DATABASE_URL=postgresql://username:password@localhost:5432/database_name
```

---

**Problem**: `CSRF verification failed`

**Solution**: CSRF middleware is removed in JWT authentication. If seeing this error, check that:
- `CsrfViewMiddleware` is NOT in `MIDDLEWARE` settings
- Requests include `Authorization: Bearer <token>` header, not cookies

---

**Problem**: Tests fail with `django.db.utils.OperationalError: database is locked`

**Solution**: SQLite concurrent access issue. Run tests with:
```bash
pytest --reuse-db  # Reuse test database between runs
```

Or use PostgreSQL for tests (recommended for CI/CD).

#### Frontend Issues

**Problem**: `Cannot find module '@/components/...'`

**Solution**: The `@` alias points to `/src`. Check `vite.config.js`:
```javascript
resolve: {
  alias: {
    '@': path.resolve(__dirname, './src'),
  },
}
```

---

**Problem**: `401 Unauthorized` on all API requests

**Solution**: Check token storage and refresh logic:
```javascript
// In browser console:
localStorage.getItem('access_token')
localStorage.getItem('refresh_token')
```

If tokens are missing, log in again. If tokens exist but requests fail, check backend logs for JWT validation errors.

---

**Problem**: Styles not loading (404 on SCSS imports)

**Solution**: Initialize git submodules:
```bash
git submodule update --init --recursive
```

Verify symlink exists:
```bash
ls -la frontend/src/styles
# Should point to: ../../shared/styles
```

---

**Problem**: E2E tests fail with timeout

**Solution**: Ensure backend is running during E2E tests:
```bash
# Terminal 1: Backend
cd backend && python manage.py runserver

# Terminal 2: Frontend dev server
cd frontend && npm run dev

# Terminal 3: E2E tests
cd frontend && npm run test:e2e
```

#### Docker Issues

**Problem**: `Error: Cannot find module 'vite'` in frontend container

**Solution**: Rebuild frontend image with clean install:
```bash
docker compose build --no-cache frontend
```

---

**Problem**: Backend container exits with "Permission denied" on key files

**Solution**: Check file permissions on JWT keys:
```bash
chmod 600 backend/config/keys/jwt_private_key.pem
chmod 644 backend/config/keys/jwt_public_key.pem
```

---

**Problem**: Traefik routing issues (404 on API calls)

**Solution**: Check Traefik dashboard at `http://localhost:8080` (development) to verify routes. Ensure:
- Backend router matches `/api`, `/_allauth`, `/accounts`
- Frontend router is fallback for all other paths

### Getting Help

- **Documentation**: Check `/id/CLAUDE.md`, `/id/backend/CLAUDE.md`, `/id/frontend/CLAUDE.md`
- **E2E Tests**: See `/id/E2E_TESTING.md` for test coverage
- **Logs**: Check Docker Compose logs: `docker compose logs -f <service>`
- **Django Shell**: `python manage.py shell` for interactive debugging

### Debug Mode

**Backend**:
```bash
# In .env.dev
DJANGO_DEBUG=True

# View detailed error pages and SQL queries
```

**Frontend**:
```bash
# Browser console shows auth state changes
# Look for: allauth.auth.change events
```

**Playwright E2E**:
```bash
# Run with visible browser and DevTools
npm run test:e2e:debug
```

---

## License

[Add license information here]

## Contact

[Add contact information here]
