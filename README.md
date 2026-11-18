# Arcadia — a live-ops game backend in .NET

Arcadia is the backend for a free-to-play game: player accounts, currencies, an item shop, inventories and global leaderboards. I'm building it step by step to take my 13 years of C#/.NET and C++ game engineering into backend services, and the repo records each step: the code, the decisions behind it, and the bugs I caused on purpose to understand them.

**Stack:** C# · .NET 10 · ASP.NET Core Minimal APIs · PostgreSQL 18 · Dapper · Npgsql · Docker Compose

---

## What this project demonstrates

**API design**
- A designed contract before any code: resources, methods, status codes and idempotency for every endpoint ([notes/1.1](notes/1.1-api-design.md)).
- Server-authoritative purchases: the client sends only the item id. Prices and the player identity never come from the client.
- An idempotency-key design so that client retries on a timeout can't buy the same item twice.

**ASP.NET Core**
- Minimal APIs with typed results (`Results<Ok<T>, NotFound, Conflict<string>>`), so every possible response is visible in the handler's signature.
- Custom middleware for correlation ids and request timing. Headers are written in `Response.OnStarting`, so they survive error responses too.
- Strongly typed options with data-annotation and cross-field validation, enforced at startup (`ValidateOnStart`).

**Dependency injection**
- Lifetimes chosen deliberately and documented in the code: the connection pool is a singleton, the repository is scoped, connections live for one operation.
- `ValidateScopes` and `ValidateOnBuild` are enabled, so captive dependencies stop the app at startup instead of leaking state at runtime.

**Database design (PostgreSQL)**
- Normalised schema with versioned migrations (`V001`–`V008`).
- Every foreign key's `ON DELETE` behaviour is a documented decision (`CASCADE` vs `RESTRICT`).
- The database is the source of truth for integrity. `CHECK` constraints prevent negative balances. Case-insensitive unique indexes on `lower(username)` and `lower(email)` prevent duplicate accounts.

**Data access (Dapper + Npgsql)**
- Every query is parameterised. Query text and values travel separately, so values are never parsed as SQL.
- Pooled connections through a singleton `NpgsqlDataSource`. Measured: 218 ms for a cold first connection versus about 4 ms for a pooled one.
- Duplicate registrations become **409 Conflict** with a specific message. This comes from catching Postgres unique-violation errors by constraint name, not from a "check first" query, which would lose a race between two concurrent requests.
- `CancellationToken` flows from the HTTP request down to every database call.

**Security, learned by breaking things**
- I deliberately built a SQL-injectable endpoint and exploited it three ways:
    - listing every player,
    - extracting every email with `UNION`,
    - running an `UPDATE` through a read-only `GET`.
- I then fixed it with parameterisation and verified the fix. The write-up covers blast radius and least-privilege database roles ([notes/1.6](notes/1.6-data-access.md)).

**SQL**
- Fifteen practice queries covering joins, anti-joins (`NOT EXISTS` vs the `NOT IN`/`NULL` trap), `GROUP BY`/`HAVING`, CTEs and window functions.
- Leaderboard ranking with `RANK`, `DENSE_RANK` and `ROW_NUMBER`, plus a "players around me" query using `LAG` ([queries](src/Arcadia.Api/db/queries/1.5-queries.sql)).

---

## Engineering notes

Each lesson ends with written answers to common interview questions, using this project's code and measurements as the examples:

| Topic | Notes |
|---|---|
| REST API design, status codes, idempotency | [1.1-api-design.md](notes/1.1-api-design.md) |
| ASP.NET Core pipeline, middleware, configuration, options | [1.2-aspnet-core.md](notes/1.2-aspnet-core.md) |
| Dependency injection, lifetimes, captive dependencies | [1.3-di.md](notes/1.3-di.md) |
| Relational modelling, keys, constraints, normalisation | [1.4-relational-fundamentals.md](notes/1.4-relational-fundamentals.md) |
| SQL: joins, aggregation, CTEs, window functions | [1.5-sql.md](notes/1.5-sql.md) |
| Data access: pooling, parameterisation, SQL injection, async I/O | [1.6-data-access.md](notes/1.6-data-access.md) |

---

## Current status

| Area | State |
|---|---|
| Player registration and profile (`POST /players`, `GET /players/{id}/profile`) | **Backed by PostgreSQL** |
| Currencies, inventory, shop, purchases, leaderboards | API contract implemented; returns placeholder data until the matching database work lands |
| Database schema: players, currencies, balances, items, inventory, leaderboard | Done, with migrations and seed data |

### Coming next
- **Indexes and query plans:** "top 100" and "players around me" over 5 million leaderboard rows, tuned with `EXPLAIN ANALYZE`.
- **Transactions and concurrency:** atomic purchases and race-free balance updates.
- **EF Core:** compared side by side with the Dapper implementation.
- **Authentication and validation:** JWT auth, request validation and `ProblemDetails` errors.
- **Integration testing:** against real PostgreSQL with Testcontainers, run in CI.
- **Later:** async internals, background jobs, distributed architecture, containerised deployment to the cloud.

---

## Running it locally

**Prerequisites:** .NET 10 SDK and Docker.

```bash
# 1. Database credentials for local development
cp .env.example .env

# 2. Start PostgreSQL. Migrations run automatically when the data volume is first created.
docker compose up -d

# 3. Point the API at the database (stored in user secrets, not in the repo)
cd src/Arcadia.Api
dotnet user-secrets set "ConnectionStrings:Arcadia" \
  "Host=localhost;Port=5433;Database=arcadia;Username=arcadia;Password=arcadia_dev"

# 4. Optional: load sample players, items, balances and leaderboard scores
docker exec -i arcadia-db psql -U arcadia -d arcadia < db/seed/seed.sql

# 5. Run the API
dotnet run
```

The API listens on `http://localhost:5293`. [`Arcadia.Api.http`](src/Arcadia.Api/Arcadia.Api.http) has ready-made requests for every endpoint, including the error cases and the SQL-injection payloads (which now fail safely). You can run them from Rider or Visual Studio.

To reset the database and run the migrations again: `docker compose down -v && docker compose up -d`.

---

## Project structure

```
src/Arcadia.Api/
├── Contracts/        # Request and response types: the API's public shape
├── Endpoints/        # Minimal API route groups (players, shop, leaderboards, diagnostics)
├── Players/          # Domain model, repository interface, Dapper implementation
├── Infrastructure/   # Correlation-id middleware, per-request context
├── Middleware/       # Request timing
├── Options/          # Validated configuration (leaderboard limits)
├── Diagnostics/      # Development-only probes (DI lifetimes, DB connectivity)
└── db/
    ├── migrations/   # Versioned schema scripts, V001–V008
    ├── seed/         # Sample data
    └── queries/      # SQL practice queries
notes/                # Design decisions and interview-question write-ups, one file per lesson
```

---

## About me

I'm Diogo Lacerda, a software engineer with 13 years' experience across games, digital TV and financial services. Most recently I was Lead Software Engineer at Outplay Entertainment, leading a team of five. This repo is where I'm moving into backend and cloud engineering in public.
