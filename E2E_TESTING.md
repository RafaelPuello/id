# E2E Testing Guide - ID Service

This document describes end-to-end testing for the DigiDex ID Service using Playwright.

## Setup

### Prerequisites

1. **Docker & Docker Compose** - Required to run the full stack
2. **Node.js** - Development machine (not required in container)
3. **Browsers** - Playwright will install them automatically

### Installation

```bash
cd id/frontend
npm install
npx playwright install  # Install browser binaries
```

## Running Tests

### Full Test Suite
```bash
npm run test:e2e
```

### With UI (Interactive Mode)
```bash
npm run test:e2e:ui
```

### Debug Mode
```bash
npm run test:e2e:debug
```

### Single Test File
```bash
npx playwright test e2e.spec.js
```

### Specific Test
```bash
npx playwright test -g "login with valid credentials"
```

### Generate & View Report
```bash
npx playwright test
npx playwright show-report
```

## Environment

The tests expect:
- **Frontend**: http://localhost:10000
- **Backend**: http://localhost:8001 (accessed via reverse proxy)
- **Test Account**: test@example.com / test123

The `playwright.config.js` file automatically starts Docker Compose if needed.

## Test Categories

### Authentication Tests
- Login with valid credentials
- Login with invalid credentials
- Login with nonexistent email
- Logout

### Account Features Tests
- Access account settings
- Email management
- Password change
- MFA settings
- Connected accounts/providers
- Session management

### Social Account Tests
- Initiate social login (Dummy provider)
- Complete social authentication

### Protected Routes Tests
- Unauthenticated users redirected to login
- Authenticated users can access account
- Logout clears authentication

### App Initialization Tests
- App loads without JavaScript errors
- Backend API connectivity

## Test Structure

Tests are organized in `e2e.spec.js` using Playwright Test's describe/test blocks:

```javascript
test.describe('ID Service - Authentication', () => {
  test('login with valid credentials', async ({ page }) => {
    // Test implementation
  })
})
```

## Helper Functions

### `login(page, email, password)`
Fills login form and submits, waits for redirect to account area.

### `logout(page)`
Logs out user, waits for redirect to home page.

### `socialAccountUIDFactory()`
Generates random social account ID for testing.

## Configuration

### playwright.config.js
- **Base URL**: http://localhost:10000
- **Timeout**: 30 seconds
- **Retries**: 2 in CI, 0 locally
- **Browsers**: Chromium, Firefox, WebKit
- **Screenshots**: Only on failure
- **Traces**: On first retry

## Notes

### Email Testing (Not Implemented)
Email verification flows are skipped because:
- Signups are disabled in production
- Mailcatcher integration requires additional setup
- Password reset/change don't require email verification in dev

Future email testing can be added by:
1. Enabling mailcatcher service in docker-compose
2. Adding `getLinkFromMail()` helper function
3. Implementing email verification flows

### Known Limitations
1. Social account provider flows depend on Dummy provider being configured
2. Some account pages may not exist if features are disabled
3. MFA setup flows require testing against actual backend configuration

## Debugging

### See Browser During Test
Use `test.only()` to run a single test and pause:

```javascript
test.only('my test', async ({ page }) => {
  await page.pause()
  // ... test code
})
```

### Check Network Requests
```javascript
page.on('request', request => console.log(request.url()))
page.on('response', response => console.log(response.status(), response.url()))
```

### View Screenshots & Traces
```bash
npx playwright show-report
```

Reports are in `playwright-report/` directory.

## CI/CD Integration

For GitHub Actions or other CI systems:

```yaml
- name: Run E2E Tests
  run: |
    cd id/frontend
    npm ci
    npm run test:e2e
```

The config will use CI settings (1 worker, 2 retries) automatically.

## Troubleshooting

### Tests fail with "Connection refused"
- Ensure Docker Compose is running: `docker compose up`
- Check port 10000 is accessible: `curl http://localhost:10000`

### Tests timeout on page navigation
- Increase `timeout` in playwright.config.js
- Check backend logs: `docker compose logs id-backend`

### CORS errors in browser console
- Backend CORS settings need to allow localhost
- Check `CORS_ALLOWED_ORIGINS` in Django settings

### "Invalid credentials" on login
- Ensure test database has test@example.com with password test123
- Check database: `docker compose exec id-backend python manage.py shell`

## Future Enhancements

1. **Visual Regression Testing** - Add screenshot comparisons
2. **Performance Testing** - Monitor load times
3. **Accessibility Testing** - Automated a11y checks
4. **API Testing** - Direct backend API tests
5. **Email Verification** - Enable when signups are allowed
6. **MFA Flows** - Comprehensive MFA testing (TOTP, WebAuthn)
7. **Error Scenarios** - Network failures, backend errors
8. **Load Testing** - Multi-user concurrent scenarios

## References

- [Playwright Documentation](https://playwright.dev)
- [Playwright Test API](https://playwright.dev/docs/api/class-test)
- [Best Practices](https://playwright.dev/docs/best-practices)
