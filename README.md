# GameHub

A game-store and social platform built as a four-person university project (2-week sprint).
Users can browse a game catalogue, buy games from a wallet balance, review them, and manage friends;
admins moderate users, games, reviews and reports.

**Stack:** C# · .NET 8 · ASP.NET Core Web API · Entity Framework Core · SQLite · JWT · Serilog · Blazor WebAssembly · MudBlazor

> Team project. Original repository: [Grzechutek/Projekt-PAB](https://github.com/Grzechutek/Projekt-PAB).
> This is my fork, kept as a portfolio copy.

## Architecture

The solution follows a layered (clean-architecture style) layout with the dependency rule pointing inwards:
the Domain layer has no reference to EF Core, BCrypt or JWT libraries; it only defines contracts.

```
GameHub.sln
└── src/
    ├── GameHub.SharedKernel    BaseEntity, shared types
    ├── GameHub.Domain          Entities, IRepository<T>, IUnitOfWork, IPasswordHasher, ITokenService
    ├── GameHub.Application     Services (Auth, Wallet), DTOs with validation, use cases
    ├── GameHub.Infrastructure  EF Core DbContext, EfRepository<T>, EfUnitOfWork, BCrypt, JWT, seeding, migrations
    ├── GameHub.WebAPI          REST controllers, Swagger, Serilog, JWT authentication
    ├── GameHub.BlazorWASM      User-facing client (catalogue, library, wallet, friends, admin pages)
    └── GameHub.BlazorServer    Blazor Server project (template-level, not the main UI)
```

```
WebAPI ──► Application ──► Domain ◄── SharedKernel
   └─────► Infrastructure ─► Application
BlazorWASM ──(HTTP only)──► WebAPI
```

Key design decisions:
- **Ports and adapters.** `IPasswordHasher` and `ITokenService` live in Domain; `BcryptPasswordHasher` and `JwtTokenService` live in Infrastructure. Application code depends only on interfaces and is easy to mock.
- **Unit of Work + generic repository.** A single `SaveChangesAsync()` commits changes across repositories; explicit transactions (`BeginTransactionAsync` / `CommitTransactionAsync`) are available for multi-table operations such as buying a game.
- **Wallet model.** Balance is stored on the user (`WalletBalance`); every movement (top-up, purchase, refund) is logged in `Transactions`.
- **Database-level integrity.** Unique indexes on email, username, `(UserId, GameId)` in the library and reviews, and on friendship pairs; decimal(18,2) for money.
- **Logging.** Serilog with a daily rolling file for everything and a separate daily file for warnings and errors.

Further documentation lives in [`docs/`](docs): architecture and initialisation, ERD (Mermaid), backend feature notes, frontend notes, project plan.

## Features

- Registration, login, profile editing (JWT bearer authentication, roles `User` / `Admin`, BCrypt password hashing with work factor 12)
- Game catalogue with filtering, sorting and search; game details with reviews and ratings
- Wallet: top-up, transaction history, buying a game (balance check, library entry and transaction record written in one database transaction)
- User library and friend libraries
- Friends system (invite / accept / reject / remove)
- Reports and moderation (hide reviews, block users, manage games)
- Swagger UI for the API (about 35 endpoints across 9 controllers)

## My role

Dev A, backend lead and architect (see `docs/general/PLAN.md`):
- designed the solution structure and layer dependencies, wrote the project initialisation script and architecture documentation
- domain model and the `IRepository<T>` / `IUnitOfWork` contracts
- authentication (registration, login, profile), password hashing and JWT issuing
- wallet module: top-up, history, atomic game purchase
- Serilog setup (daily rolling log plus a separate error log)
- repository setup and integration: initial repo with docs (31 May), backend foundation (1–3 June), reviewed and merged teammates' pull requests (#1, #2) into `main`

The remaining modules (CRUD controllers, friends, reports, moderation, Blazor pages) were written by my teammates according to the roles in `PLAN.md`.

## Running locally

Requirements: .NET SDK 9.0.306 (pinned in `global.json`; projects target `net8.0`).

```bash
# 1. Start the API (applies EF Core migrations and seeds the database on startup)
dotnet run --project src/GameHub.WebAPI
# Swagger UI: http://localhost:5224/swagger

# 2. Start the client (in a second terminal)
dotnet run --project src/GameHub.BlazorWASM
```

The API allows CORS from the client origins configured in `src/GameHub.WebAPI/Program.cs`;
make sure they match the port the client starts on. The `Jwt:Key` in `appsettings.json` is a development placeholder;
for any real deployment set it through user-secrets or an environment variable.

### Seeded accounts

| Email | Password | Role |
|---|---|---|
| admin@gamehub.com | Admin123! | Admin |
| alice@gamehub.com | Alice123! | User (wallet 150) |
| bob@gamehub.com | Bob123! | User (wallet 80) |

## Known limitations and next steps

- **Concurrency in the purchase flow.** `BuyGameAsync` reads the balance and checks it before opening the transaction and has no optimistic-concurrency token, so two parallel purchases could in theory both pass the balance check. The unique `(UserId, GameId)` index already prevents duplicate library entries, but the balance update would need a concurrency token or a conditional `UPDATE`. This is the same class of problem (check-then-act on a shared resource) that I study in my master's thesis.
- No automated tests yet. The service layer depends on `IUnitOfWork`, so `WalletService` and `AuthService` are straightforward candidates for unit tests with mocked repositories.
- Blazor Server project is still close to the default template; the admin pages are implemented in the WASM client.
- Token claim `WalletBalance` can become stale until the next login; the balance should be read from the API.
- `IsBlocked` is checked at login only, not on every authenticated request.
