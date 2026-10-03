# Food Order Platform

A small food-ordering system for one sample restaurant:

- **Admin interface** (React + TypeScript) for staff to maintain the food catalog, process orders and manage staff.
- **REST API** (Node.js + Express + PostgreSQL) under `/api/v1`, serving both the admin interface and a future mobile app (`/api/v1/mobile`).

Payments, checkout, live notifications, the mobile app itself and production deployment are out of scope.

> What is done, what is partial and what is not done are listed honestly in [`docs/DELIVERY.md`](docs/DELIVERY.md). Read it before reviewing.

## Documentation

| Document | Contents |
| --- | --- |
| [`docs/API.md`](docs/API.md) | Conventions, auth and roles, error codes, curl walkthrough of the mobile flow |
| [`docs/openapi.yaml`](docs/openapi.yaml) | OpenAPI 3 description of every endpoint |
| [`docs/postman/`](docs/postman/food-order-platform.postman_collection.json) | Postman collection with tests; runs the mobile flow and the staff workflow against the seed data |
| [`docs/USER_MANUAL.md`](docs/USER_MANUAL.md) | How administrators and operators use the admin interface |
| [`docs/DELIVERY.md`](docs/DELIVERY.md) | Feature checklist, stack and versions, schema, test results, limitations, handover |

## Architecture

```
React admin UI (Vite, :5173)  --HTTPS/JSON-->  Express API (:8080)  --Sequelize-->  PostgreSQL
Mobile app (future)           --HTTPS/JSON-->        |
                                                     +-- /api/v1/auth, /staff-users, /foods, /orders   staff routes
                                                     +-- /api/v1/mobile                                customer routes
```

- **Auth:** signed JWT bearer tokens. A token carries a `type` (`staff` or `customer`) that the API checks, so a customer token can never be used on a staff route or the reverse. For staff the role is read from the database on every request (not trusted from the token), deactivation takes effect immediately, and logout / password reset revoke older tokens through a per-user `token_version`.
- **Order workflow:** `Placed -> Accepted -> Preparing -> Ready -> Completed`, with `Cancelled` (reason required) from the first three states. Invalid transitions are rejected by the API whatever the UI shows. A change locks the order row, updates the status and appends to `order_status_history` in one transaction; a database trigger makes that table append-only.
- **Money:** `numeric(10,2)` columns, computed in integer cents in code, exchanged as decimal strings (`"48.00"`). Order totals are always computed on the server from current prices; client-supplied prices are ignored. Each order line snapshots the food name and unit price.
- **Soft deletion:** foods are archived, not deleted (the foreign key from order lines is `RESTRICT`), so order history stays readable.
- **Single restaurant, single currency:** set with `CURRENCY` (default `AED`) and stored on each food.

```
backend/    Express API, Sequelize models, migrations, seed, tests
frontend/   React + TypeScript + Vite + Tailwind admin interface
docs/       API notes, OpenAPI, Postman collection, user manual, delivery notes
.github/    CI workflow and pull request template
```

## Prerequisites

- Node.js 20 or newer
- PostgreSQL 14 or newer (16 is used in CI). The database user must be able to create databases for the tests.

## Local setup

### 1. Database

```bash
createdb food_order            # or use any name and set DB_NAME below
```

### 2. Backend

```bash
cd backend
cp .env.example .env           # edit DB_* values and JWT_LOGIN_SECRET
npm install
npm run db:migrate             # create tables
npm run db:seed                # load sample data
npm run start:dev              # http://localhost:8080
```

`npm run db:reset` drops every table in the configured database, then migrates and seeds again. **It is destructive; use it on local databases only** (it refuses to run with `NODE_ENV=production`).

### 3. Frontend

```bash
cd frontend
cp .env.example .env           # optional: VITE_API_URL defaults to http://localhost:8080/api/v1
npm install
npm run dev                    # http://localhost:5173
```

### Environment variables (backend)

See [`backend/.env.example`](backend/.env.example).

| Variable | Purpose | Default |
| --- | --- | --- |
| `NODE_ENV` | `development`, `test`, `staging`, `production` | `development` |
| `PORT` | API port | `8080` |
| `CORS_ORIGIN` | Comma-separated allowed browser origins | `http://localhost:5173,http://localhost:5174` |
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD` | PostgreSQL connection | local defaults |
| `DB_SSL` | `true` for hosted databases that require TLS | `false` |
| `TEST_DB_NAME` | Database that `npm test` creates and rebuilds | `food_order_test` |
| `JWT_LOGIN_SECRET` | Token signing secret. **Required** when `NODE_ENV=production` (the server refuses to start otherwise) | insecure dev default |
| `JWT_LOGIN_TOKEN_EXPIRY` | Token lifetime | `2d` |
| `LOGIN_RATE_LIMIT_MAX` | Failed logins per IP per 15 minutes | `5` |
| `CURRENCY`, `RESTAURANT_NAME` | Single restaurant and currency | `AED`, `Sample Restaurant` |
| `TRUST_PROXY` | Number of trusted reverse proxies; leave unset locally | unset |

## Test accounts (local development only)

Created by `npm run db:seed`. **Never use these outside a local database.**

| Role | Email | Password |
| --- | --- | --- |
| Administrator | `admin@example.com` | `password123` |
| Operator | `operator@example.com` | `password123` |
| Customer (mobile API) | `customer@example.com` | `password123` |
| Second customer (to check ownership rules) | `customer2@example.com` | `password123` |

Seed data: 2 staff users, 2 customers, 6 foods in 4 categories (Truffle Fries is unavailable), and 6 orders covering every status (Placed, Accepted, Preparing, Ready, Completed, Cancelled) with full status histories.

## Commands

| Where | Command | What it does |
| --- | --- | --- |
| `backend` | `npm run start:dev` | Run the API with Babel (development) |
| `backend` | `npm run build` then `npm start` | Compile to `dist/` and run the compiled build |
| `backend` | `npm test` | 13 unit tests and 52 API tests against a real PostgreSQL test database (`TEST_DB_NAME`, rebuilt automatically) |
| `backend` | `npm run test:unit` | Only the unit tests | Make test:unit skip the database setup so it really works without Postgres |
| `backend` | `npm run lint` | Lint with `standard` |
| `backend` | `npm run db:migrate` / `db:seed` / `db:reset` | Migrations, seed data, full rebuild |
| `backend` | `BASE_URL=http://localhost:8080 ./scripts/smoke-test.sh` | End-to-end API check against a running, seeded server (needs curl and node) |
| `frontend` | `npm run dev` | Start the admin UI |
| `frontend` | `npm test` | 26 component and unit tests (Vitest, Testing Library) |
| `frontend` | `npm run lint` | Lint with oxlint |
| `frontend` | `npm run build` | Typecheck (`tsc -b`) and build |
| repo root | `npx newman run docs/postman/food-order-platform.postman_collection.json` | Run the Postman collection against a running, seeded server |

## Continuous integration

[`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs on every pull request and every push to `main`:

- **Backend:** install, lint, tests against a PostgreSQL service container, build, rebuild the database from scratch with `db:reset`, start the built server, run the smoke script and the Postman collection with newman.
- **Frontend:** install, lint, tests, typecheck and build.

## Git workflow

- `main` is protected by convention: no direct pushes; changes arrive through pull requests.
- Branch names: `feature/<topic>` or `fix/<topic>`.
- Focused commits; each pull request uses the template in `.github/pull_request_template.md` (summary, test evidence, screenshots for UI changes).
- Intended branch protection if available: require the `backend` and `frontend` CI checks to pass before merging.

## Phase time log

Target: 20-24 working hours over 4 calendar days. Record actual hours by phase.

| Phase | Target | Actual | Notes |
| --- | --- | --- | --- |
| 0 Plan and setup | 2-3 h | TODO | |
| 1 Data and security | 4-5 h | TODO | |
| 2 Admin interface | 6-7 h | TODO | |
| 3 Mobile API and workflow | 4-5 h | TODO | |
| 4 Verification and handover | 4-5 h | TODO | |

### Assumptions
- One restaurant, one currency (AED), set in config. No multi-branch.
- Customers order only through the mobile API. Staff cannot create orders.
- Prices are fixed at the moment of ordering. Later price changes do not affect old orders.
- No payments, taxes, delivery fees or discounts. Total = sum of the lines.
- Two staff roles only (admin, operator), so a simple role column is enough. No permissions table.
- Foods are never hard-deleted once ordered. They are archived so order history stays readable.
- Customer accounts are seeded. No sign-up screen.
- Plain JavaScript on the backend (from my existing boilerplate) to save setup time;
  TypeScript on the frontend as required.

### Milestones
| Phase | Deliverable | Done when |
| --- | --- | --- |
| 0 Plan | This plan, repo, scaffolding | Both apps start locally |
| 1 Data & security | Tables, migrations, seed, staff login, roles, validation, error format | Operator gets 403 on staff users; bad input returns VALIDATION_ERROR |
| 2 Admin UI | Login, dashboard, foods, orders, order detail, staff users | Admin can process a seeded order from Placed to Completed in the browser |
| 3 Mobile API | Customer login, foods, place order, my orders | Postman flow places an order; customer A cannot read customer B's order |
| 4 Handover | Tests, CI, API docs, manual, screenshots, recording | CI green on a PR; reviewer can follow the README from scratch |

Order of work: backend rules first, then UI on top, because "a polished UI without working
authorization or order rules does not satisfy this assignment."

### Risks
| Risk | Plan |
| --- | --- |
| Two staff click "Accept" on the same order at once | Lock the order row in a transaction; test with two parallel requests |
| Customer token used on staff routes | Put a token type (staff/customer) in the token and check it on every route |
| Money rounding errors (0.1 + 0.2) | Store numeric(10,2), calculate in whole cents |
| Leftover code or secrets in the boilerplate | Audit and remove unused parts in phase 0; keep secrets only in .env |
| Deleting a food breaks old orders | Archive instead of delete; database refuses hard delete |
| Running out of time | Core flow first; mark unfinished extras in DELIVERY.md |

### What changed from the plan
- The boilerplate's email one-time-code login was insecure (codes stored in plain text), so I removed it
  instead of reusing it. Cost about 2 extra hours.
- A test caught that customer tokens worked on staff routes. Fixed with the token-type claim.
- A concurrency test found a database deadlock when two admins deactivate each other at once.
  Fixed by always locking rows in the same order.
- Search uses simple ILIKE, which is fine for a restaurant menu. A faster index was skipped
  as out of scope.
- I did not get to screenshots or a mobile/accessibility audit on time. They are listed in DELIVERY.md.
