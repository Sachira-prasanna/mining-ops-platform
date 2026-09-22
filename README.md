# Mining Operations Data Platform

A backend REST API for managing mining drilling operations: equipment tracking,
drill-hole management, and shift production reporting. Built as a self-directed
project to a production-style standard — normalized schema, layered architecture,
JWT/RBAC security, request validation, OpenAPI documentation, containerization,
and CI.

## Tech Stack

- **Runtime / Language:** Node.js, TypeScript
- **Framework:** Express
- **Database / ORM:** PostgreSQL, Prisma
- **Auth:** JWT (jsonwebtoken, bcryptjs), three-tier RBAC (ADMIN, SUPERVISOR, OPERATOR)
- **Validation:** Zod, on every mutating route
- **Docs:** OpenAPI/Swagger, served at `/api/docs`
- **Testing:** Jest, Supertest
- **Containerization:** Docker (multi-stage build), docker-compose
- **CI/CD:** GitHub Actions (lint → test → build → Docker image)

## Architecture

The app is split into distinct layers so each piece has one responsibility:

```
src/
  routes/       Express routers — URL + method + middleware wiring only
  controllers/  Parse the request, call a service, shape the response
  services/     Business logic and all Prisma/database access
  middleware/   authenticate, authorize, validate, error handling
  config/       env loading, Prisma client, Swagger spec
```

## Data Model

- **User** — email, hashed password, role (ADMIN / SUPERVISOR / OPERATOR)
- **Site** — a mine site; owns equipment and drill holes
- **Equipment** — asset tag, type, status, service history (`EquipmentUsageLog`)
- **DrillHole** — hole code, planned/actual depth, dip, azimuth, status lifecycle
- **ProductionReport** — tonnes mined per shift, linked to a drill hole and the reporting user

See [`prisma/schema.prisma`](./prisma/schema.prisma) for the full normalized schema.

## Getting Started

### Option A — Docker (recommended)

```bash
cp .env.example .env
docker compose up --build
```

The API will be available at `http://localhost:3000`, with interactive docs at
`http://localhost:3000/api/docs`.

### Option B — Local Node

```bash
cp .env.example .env          # then point DATABASE_URL at your own Postgres
npm install
npm run prisma:migrate
npm run prisma:seed           # optional: creates a demo admin user + site
npm run dev
```

Default seeded admin: `admin@miningops.local` / `ChangeMe123!` — change this
immediately in any non-local environment.

## API Overview

All endpoints except `/auth/register` and `/auth/login` require
`Authorization: Bearer <token>`.

| Method | Route | Roles | Purpose |
|---|---|---|---|
| POST | `/api/auth/register` | — | Create a user |
| POST | `/api/auth/login` | — | Get a JWT |
| GET | `/api/equipment` | any | List equipment |
| POST | `/api/equipment` | ADMIN, SUPERVISOR | Register equipment |
| PATCH | `/api/equipment/:id/status` | ADMIN, SUPERVISOR | Change equipment status |
| POST | `/api/equipment/:id/usage-logs` | any | Log usage hours |
| GET | `/api/drill-holes` | any | List drill holes |
| POST | `/api/drill-holes` | ADMIN, SUPERVISOR | Create a drill hole |
| PATCH | `/api/drill-holes/:id/status` | any | Update status / actual depth |
| GET | `/api/production-reports` | any | List production reports |
| POST | `/api/production-reports` | any | Submit a shift report |
| GET | `/api/production-reports/summary/:siteId` | any | Aggregate tonnes for a site |

Full request/response schemas are in Swagger at `/api/docs`.

## Testing & CI

```bash
npm run lint
npm test
npm run build
```

`.github/workflows/ci.yml` runs lint, tests (against a real Postgres service
container), a production build, and a Docker image build on every push and
pull request to `main`.

## Deployment Notes

The Dockerfile is a multi-stage build: dependencies and TypeScript compile in
a `build` stage, and only the compiled `dist/` output plus production
dependencies ship in the final image. It's designed to run on AWS EC2 with an
RDS PostgreSQL instance — set `DATABASE_URL` to the RDS endpoint and
`JWT_SECRET` to a securely generated value at deploy time.

## License

MIT
