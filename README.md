# PosyMart multi-business POS

PosyMart is a responsive web/PWA foundation for multi-branch retail and restaurant operations. It includes tenant registration, role-scoped workspaces, public stores, waiter tables, kitchen tickets, cashier checkout, inventory, deliveries, refunds, reporting, audit, and a Super Admin console.

## Run with Docker

Install Docker Compose, then run `docker compose up --build` from the project root. The first database start applies [database/schema.sql](./database/schema.sql). Open:

- `http://localhost:4000` — public store directory
- `http://localhost:4000/admin` — staff and platform workspace
- `http://localhost:4000/tracking.html` — phone-verified order tracking
- `http://localhost:4000/api/health` — API/database health

The Compose password and session secret defaults are for local development. Override them before deployment. The database volume persists between local restarts.

## Run with Node.js and PostgreSQL

1. Use Node.js 20 or newer and PostgreSQL 15 or newer.
2. Create a `posymart` database.
3. Copy `server/.env.example` to `server/.env`; set `DATABASE_URL` and a unique `SESSION_SECRET`.
4. From `server/`, run `npm install`, `npm run db:setup`, then `npm run dev`.
5. Set `SUPER_ADMIN_EMAIL` and a unique `SUPER_ADMIN_PASSWORD` of at least 16 characters in `server/.env`, then run `npm run bootstrap:admin` once.
6. Create a business workspace at `/admin`. New tenants remain pending until a platform administrator activates them.

The public business link is `/store/{slug}`. Staff roles and business/branch memberships are validated by the API. Owners can switch branches from the workspace header.

## Deploy to Vercel

The root `Dockerfile.vercel` deploys the existing Express application as a Vercel container. Connect a hosted PostgreSQL database (Neon is available through the Vercel Marketplace), then configure these production environment variables in the Vercel project:

- `DATABASE_URL` — the provider's pooled PostgreSQL connection string
- `PGSSL=true`
- `SESSION_SECRET` — a unique random value of at least 32 characters
- `APP_ORIGIN` and `ALLOWED_ORIGINS` — the HTTPS origin used by the production deployment
- `SUPER_ADMIN_EMAIL` and `SUPER_ADMIN_PASSWORD` — set only when bootstrapping the first platform administrator

After connecting the database, pull the production variables into the ignored `server/.env` file and apply the schema once with `npm run db:setup` from `server/`. Bootstrap the platform administrator with `npm run bootstrap:admin`, then deploy from the project root with `vercel --prod`. Do not commit environment files, receipt PDFs, or local image files.

## Test and backup

Run unit tests with `npm test` from `server/`. The database acceptance suite is enabled when `TEST_DATABASE_URL` points to a dedicated disposable PostgreSQL test database; it covers authentication, permissions, tenant catalogue isolation, stock-safe/idempotent checkout, cash sessions, and payment webhook retries.

Back up PostgreSQL with `pg_dump --format=custom --file=posymart-backup.dump "$env:DATABASE_URL"` and verify restore procedures in a separate database. Store backups and provider secrets outside this workspace.

## Security and integrations

Passwords use scrypt hashes; sessions are opaque, revocable, stored as hashes, and protected by HttpOnly cookies and CSRF tokens. API reads and writes use the session's business and branch scope. Sensitive operations write audit records. Set `ALLOWED_ORIGINS` to the deployed origins and use HTTPS in production.

Set `NOTIFICATION_WEBHOOK_URL` and `NOTIFICATION_WEBHOOK_SECRET` to deliver signed password-reset, staff-invitation, and tracking-code events through an email/SMS adapter. In development, reset/invitation/OTP flows return a debug token or code.

Payment collection is not simulated as successful. Cash is recorded by staff; mobile-money and card payments remain pending until a cashier verifies them or a signed webhook is received. Configure one `PAYMENT_WEBHOOK_SECRET_<PROVIDER>` per provider, and optionally `PAYMENT_METHOD_<PROVIDER>` when its payment method name differs from the provider key. Never collect PINs, CVVs, or banking passwords.

Live GPS tracking and native mobile apps are outside this release. Production deployment still needs a TLS-terminating host, managed backups, external monitoring, a distributed rate limiter, and provider credentials. The Compose setup is a local deployment baseline, not a hosted production environment.
