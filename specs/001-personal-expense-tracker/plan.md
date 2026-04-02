# Personal Expense Tracker — Implementation Plan

> **Spec reference:** Issue #1 — Project: Personal Expense Tracker  
> **Architecture style:** Modular Monolith  
> **Governance:** L0 Enterprise · L1 Product · L3 Domain (Global Tax)  
> **Status:** READY FOR IMPLEMENTATION

---

## Table of Contents

1. [Overview](#overview)
2. [Final Tech Stack](#final-tech-stack)
3. [System Architecture](#system-architecture)
4. [Project Structure](#project-structure)
5. [Database Design](#database-design)
6. [API Contract](#api-contract)
7. [Frontend Design](#frontend-design)
8. [Security Implementation](#security-implementation)
9. [Accessibility Requirements](#accessibility-requirements)
10. [Testing Strategy](#testing-strategy)
11. [Infrastructure & CI/CD](#infrastructure--cicd)
12. [Implementation Task List](#implementation-task-list)

---

## Overview

The Personal Expense Tracker is a single-user web application for recording, viewing, and managing daily expenses. No authentication is required. The application exposes a REST API (documented with OpenAPI/Swagger) backed by SQL Server and serves a React frontend from the same ASP.NET Core host.

**Key constraints:**
- Single-user, no login/auth required
- SQL Server as the primary data store
- Azure Services for hosting
- WCAG 2.2 Level AA accessibility
- OWASP Top 10 security compliance
- 80% minimum test coverage
- OpenAPI 3.1 with Swagger UI (disabled in production)

---

## Final Tech Stack

| Layer | Technology | Version | Notes |
|---|---|---|---|
| Backend language | C# | 13 | |
| Backend framework | ASP.NET Core | .NET 9 | |
| ORM | Entity Framework Core | 9.x | Code-first migrations |
| Database | SQL Server / Azure SQL | Latest | `LocalDB` for dev, Azure SQL for prod |
| Frontend language | TypeScript | 5.x | |
| Frontend framework | React | 18.x | |
| Frontend build tool | Vite | 5.x | Proxies API in dev, outputs to `wwwroot` |
| API documentation | `Microsoft.AspNetCore.OpenApi` + Swagger UI | Built-in .NET 9 | Disabled in production |
| Backend testing | xUnit + Moq + `WebApplicationFactory` | Latest | |
| Frontend testing | Vitest + React Testing Library | Latest | |
| Linting (BE) | `dotnet format` | .NET 9 CLI | |
| Linting (FE) | ESLint + Prettier | Latest | |
| Infrastructure | Azure App Service + Azure SQL Database | — | |
| Secrets | Azure Key Vault / App Service env vars | — | Never in source |
| CI/CD | GitHub Actions | — | |

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Client Browser                                                  │
│  React SPA (TypeScript + Vite)                                   │
│  Served from ASP.NET Core /wwwroot                               │
└───────────────────────────┬──────────────────────────────────────┘
                            │ HTTP/REST  (JSON)
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  ASP.NET Core 9 Web API  (Modular Monolith)                      │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Expenses Module                                        │    │
│  │  ExpensesController → IExpenseService → IExpenseRepo    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Common                                                 │    │
│  │  GlobalExceptionHandler · SecurityHeadersMiddleware     │    │
│  │  ErrorResponse · HealthChecks · OpenAPI config          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Infrastructure                                         │    │
│  │  ExpenseDbContext (EF Core) · ExpenseRepository         │    │
│  └─────────────────────────────────────────────────────────┘    │
└───────────────────────────┬──────────────────────────────────────┘
                            │ EF Core
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│  Azure SQL Database                                              │
│  Table: Expenses                                                 │
└──────────────────────────────────────────────────────────────────┘
```

**Architectural decisions:**

| ADR | Decision | Rationale |
|---|---|---|
| ADR-001 | Modular Monolith, single deployable | Avoids over-engineering for a personal tool; preserves clear module boundaries |
| ADR-002 | React SPA served from ASP.NET Core `wwwroot` | One deployment unit; no CORS concerns; simpler Azure App Service setup |
| ADR-003 | EF Core code-first migrations | Keeps schema in source control; enables Azure SQL without manual DDL |
| ADR-004 | Repository pattern | Decouples business logic from EF Core; enables unit testing via mocks |
| ADR-005 | Built-in .NET 9 OpenAPI | No third-party dependency; Swagger UI added only in dev/staging |

---

## Project Structure

```
personal-expense-tracker/
├── ExpenseTracker.sln
│
├── src/
│   └── ExpenseTracker.Api/                   ← ASP.NET Core 9 host (single project)
│       │
│       ├── Modules/
│       │   └── Expenses/                     ← Expenses module boundary
│       │       ├── ExpensesController.cs      ← REST controller
│       │       ├── IExpenseService.cs         ← Service interface
│       │       ├── ExpenseService.cs          ← Business logic
│       │       ├── IExpenseRepository.cs      ← Repository interface
│       │       └── Models/
│       │           ├── Expense.cs             ← Domain entity (EF Core model)
│       │           ├── ExpenseDto.cs          ← Response DTO
│       │           ├── CreateExpenseRequest.cs ← Validated input DTO
│       │           ├── UpdateExpenseRequest.cs ← Validated input DTO
│       │           └── ExpenseSummaryDto.cs   ← Summary response DTO
│       │
│       ├── Infrastructure/
│       │   ├── Data/
│       │   │   ├── ExpenseDbContext.cs        ← EF Core DbContext
│       │   │   └── Migrations/               ← EF Core migrations (auto-generated)
│       │   └── Repositories/
│       │       └── ExpenseRepository.cs       ← EF Core implementation
│       │
│       ├── Common/
│       │   ├── Models/
│       │   │   └── ErrorResponse.cs           ← Standard error envelope (all non-2xx)
│       │   └── Middleware/
│       │       ├── GlobalExceptionHandler.cs  ← Maps exceptions → ErrorResponse
│       │       └── SecurityHeadersMiddleware.cs ← OWASP security headers
│       │
│       ├── wwwroot/                           ← React build output (gitignored during dev)
│       ├── Program.cs                         ← DI registration, middleware pipeline
│       ├── appsettings.json
│       ├── appsettings.Development.json       ← LocalDB connection string
│       └── ExpenseTracker.Api.csproj
│
├── client/                                    ← React + TypeScript frontend
│   ├── src/
│   │   ├── components/
│   │   │   ├── ExpenseForm.tsx                ← Add / Edit form
│   │   │   ├── ExpenseList.tsx                ← Table of expenses
│   │   │   ├── ExpenseRow.tsx                 ← Single table row with actions
│   │   │   ├── SummaryCard.tsx                ← Total / by-category summary
│   │   │   ├── ConfirmDialog.tsx              ← Delete confirmation dialog
│   │   │   └── EmptyState.tsx                 ← No-expenses message
│   │   ├── services/
│   │   │   └── expenseApi.ts                  ← Typed fetch wrapper for REST API
│   │   ├── types/
│   │   │   └── expense.ts                     ← TypeScript interfaces
│   │   ├── hooks/
│   │   │   └── useExpenses.ts                 ← Data-fetching custom hook
│   │   ├── App.tsx                            ← Root component
│   │   ├── main.tsx                           ← React entry point
│   │   └── index.css                          ← Global styles (Motif tokens)
│   ├── index.html
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts                         ← Proxy /api → localhost:5000 in dev
│   └── .eslintrc.cjs
│
├── tests/
│   └── ExpenseTracker.Tests/                  ← Backend tests (xUnit)
│       ├── Unit/
│       │   ├── Services/
│       │   │   └── ExpenseServiceTests.cs
│       │   └── Controllers/
│       │       └── ExpensesControllerTests.cs
│       ├── Integration/
│       │   └── ExpensesEndpointTests.cs       ← WebApplicationFactory + in-memory DB
│       └── ExpenseTracker.Tests.csproj
│
├── .github/
│   └── workflows/
│       └── ci.yml                             ← Build, test, lint, coverage gate
│
└── .gitignore
```

---

## Database Design

### Schema

```sql
-- Table: Expenses
CREATE TABLE Expenses (
    Id          INT             IDENTITY(1,1) PRIMARY KEY,
    Amount      DECIMAL(18,2)   NOT NULL,
    Category    NVARCHAR(50)    NOT NULL,
    [Date]      DATE            NOT NULL,
    Description NVARCHAR(500)   NULL,
    CreatedAt   DATETIME2       NOT NULL DEFAULT GETUTCDATE(),
    UpdatedAt   DATETIME2       NOT NULL DEFAULT GETUTCDATE()
);

-- Constraint: amount must be > 0 (enforced in EF Core HasCheckConstraint)
-- Constraint: category must be one of the approved values (enforced in application layer)
```

### EF Core Entity

```csharp
// Modules/Expenses/Models/Expense.cs
public sealed class Expense
{
    public int Id { get; set; }
    public decimal Amount { get; set; }          // > 0
    public string Category { get; set; } = null!; // Food | Transport | Utilities | Entertainment | Other
    public DateOnly Date { get; set; }
    public string? Description { get; set; }     // max 500 chars
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

### EF Core DbContext Configuration

```csharp
// Infrastructure/Data/ExpenseDbContext.cs
public class ExpenseDbContext : DbContext
{
    public DbSet<Expense> Expenses => Set<Expense>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Expense>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.Amount).HasPrecision(18, 2);
            e.HasCheckConstraint("CK_Expenses_Amount", "[Amount] > 0");
            e.Property(x => x.Category).HasMaxLength(50).IsRequired();
            e.Property(x => x.Description).HasMaxLength(500);
            e.Property(x => x.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
            e.Property(x => x.UpdatedAt).HasDefaultValueSql("GETUTCDATE()");
        });
    }
}
```

### Approved Categories (Enum / Constant)

```
Food
Transport
Utilities
Entertainment
Other
```

---

## API Contract

### Base URL

- Development: `http://localhost:5000/api/v1`
- Production: `https://<azure-app-service>.azurewebsites.net/api/v1`

### OpenAPI Info

```yaml
openapi: "3.1.0"
info:
  title: "Personal Expense Tracker API"
  version: "1.0.0"
  description: "REST API for managing personal daily expenses."
servers:
  - url: "/api/v1"
    description: "Current environment"
```

### Endpoints

| Method | Path | `operationId` | Auth | Description |
|---|---|---|---|---|
| `GET` | `/health` | `getHealth` | None | Liveness check |
| `GET` | `/health/ready` | `getHealthReady` | None | Readiness check (DB ping) |
| `GET` | `/api/v1/expenses` | `listExpenses` | None | List all expenses, sorted by date descending |
| `POST` | `/api/v1/expenses` | `createExpense` | None | Create a new expense |
| `GET` | `/api/v1/expenses/{id}` | `getExpense` | None | Get a single expense |
| `PUT` | `/api/v1/expenses/{id}` | `updateExpense` | None | Replace an expense |
| `DELETE` | `/api/v1/expenses/{id}` | `deleteExpense` | None | Delete an expense |
| `GET` | `/api/v1/expenses/summary` | `getExpenseSummary` | None | Aggregated totals |

### Request / Response Schemas

#### `CreateExpenseRequest`
```json
{
  "amount": 42.50,
  "category": "Food",
  "date": "2026-04-02",
  "description": "Lunch at the deli"
}
```
**Validation rules:**
- `amount`: required, `number`, `> 0`, max 2 decimal places
- `category`: required, must be one of `["Food","Transport","Utilities","Entertainment","Other"]`
- `date`: required, `string` ISO 8601 date (`yyyy-MM-dd`), not in the future by more than 1 day
- `description`: optional, max 500 characters

#### `UpdateExpenseRequest`
Same fields as `CreateExpenseRequest` — full replacement (PUT).

#### `ExpenseDto` (response)
```json
{
  "id": 1,
  "amount": 42.50,
  "category": "Food",
  "date": "2026-04-02",
  "description": "Lunch at the deli",
  "createdAt": "2026-04-02T12:00:00Z",
  "updatedAt": "2026-04-02T12:00:00Z"
}
```

#### `ExpenseSummaryDto` (response)
```json
{
  "totalAllTime": 1234.50,
  "totalCurrentMonth": 300.00,
  "byCategory": [
    { "category": "Food", "total": 150.00 },
    { "category": "Transport", "total": 100.00 },
    { "category": "Utilities", "total": 50.00 }
  ]
}
```

#### `ErrorResponse` (all non-2xx)
```json
{
  "code": 400,
  "message": "Validation failed.",
  "details": "Amount must be greater than 0."
}
```

### Response Codes

| Endpoint | 200 | 201 | 204 | 400 | 404 | 500 |
|---|---|---|---|---|---|---|
| `listExpenses` | ✓ | | | | | ✓ |
| `createExpense` | | ✓ | | ✓ | | ✓ |
| `getExpense` | ✓ | | | | ✓ | ✓ |
| `updateExpense` | ✓ | | | ✓ | ✓ | ✓ |
| `deleteExpense` | | | ✓ | | ✓ | ✓ |
| `getExpenseSummary` | ✓ | | | | | ✓ |

---

## Frontend Design

### Pages / Views

The SPA has a single page with the following regions:

```
┌──────────────────────────────────────────────────────┐
│  Header: "Personal Expense Tracker"                 │
├──────────────────────────────────────────────────────┤
│  SummaryCard                                        │
│  Total all time | Total this month | By category   │
├──────────────────────────────────────────────────────┤
│  [+ Add Expense]  button                            │
├──────────────────────────────────────────────────────┤
│  ExpenseList (table)                                │
│  Date | Category | Amount | Description | Actions   │
│  ─────────────────────────────────────────────────  │
│  2026-04-02 | Food | $42.50 | Lunch | [Edit][Del]  │
│  (empty state message if no rows)                   │
└──────────────────────────────────────────────────────┘
```

### Components

| Component | Props | Responsibilities |
|---|---|---|
| `App` | — | Fetches expenses; holds state; renders all sections |
| `SummaryCard` | `summary: ExpenseSummaryDto` | Displays totals and category breakdown |
| `ExpenseList` | `expenses[]`, `onEdit`, `onDelete` | Renders the `<table>`, handles empty state |
| `ExpenseRow` | `expense`, `onEdit`, `onDelete` | Single `<tr>` with edit/delete buttons |
| `ExpenseForm` | `expense?`, `onSave`, `onCancel` | Add/Edit form (modal). Today's date pre-filled |
| `ConfirmDialog` | `message`, `onConfirm`, `onCancel` | Delete confirmation dialog |
| `EmptyState` | — | "No expenses yet" message |

### TypeScript Types

```typescript
// types/expense.ts
export type Category = 'Food' | 'Transport' | 'Utilities' | 'Entertainment' | 'Other';

export interface Expense {
  id: number;
  amount: number;
  category: Category;
  date: string;          // ISO 8601 date (yyyy-MM-dd)
  description?: string;
  createdAt: string;     // ISO 8601 datetime
  updatedAt: string;
}

export interface ExpenseSummary {
  totalAllTime: number;
  totalCurrentMonth: number;
  byCategory: { category: Category; total: number }[];
}

export interface CreateExpenseRequest {
  amount: number;
  category: Category;
  date: string;
  description?: string;
}

export type UpdateExpenseRequest = CreateExpenseRequest;
```

### State Management

Use React's built-in `useState` / `useEffect` — no Redux or Zustand needed.

```
App state:
  - expenses: Expense[]
  - summary: ExpenseSummary | null
  - loading: boolean
  - error: string | null
  - editingExpense: Expense | null   (null = adding new)
  - showForm: boolean
  - deleteTargetId: number | null
```

### Styling

- Use **Motif design tokens** (CSS variables) for all colors and spacing — no hard-coded hex values.
- Minimal custom CSS; prefer semantic HTML and Motif CSS classes where available.
- Responsive layout: single column on narrow screens, two-column (summary | list) on wide screens.
- Dark/light theme support via `data-motif-theme` attribute.

### API Service

```typescript
// services/expenseApi.ts
const BASE = '/api/v1';

export const expenseApi = {
  list:    (): Promise<Expense[]>         => fetchJson(`${BASE}/expenses`),
  get:     (id: number)                  => fetchJson<Expense>(`${BASE}/expenses/${id}`),
  create:  (req: CreateExpenseRequest)   => fetchJson<Expense>(`${BASE}/expenses`, 'POST', req),
  update:  (id: number, req: UpdateExpenseRequest) =>
             fetchJson<Expense>(`${BASE}/expenses/${id}`, 'PUT', req),
  remove:  (id: number)                  => fetchVoid(`${BASE}/expenses/${id}`, 'DELETE'),
  summary: (): Promise<ExpenseSummary>   => fetchJson(`${BASE}/expenses/summary`),
};
```

---

## Security Implementation

> All OWASP Top 10 risks must be addressed.

| OWASP Risk | Mitigation |
|---|---|
| A01 Broken Access Control | No access control needed (single-user); no sensitive data exposed |
| A02 Cryptographic Failures | HTTPS enforced in production; no sensitive data stored |
| A03 Injection | Parameterized queries via EF Core (no raw SQL); model binding prevents injection |
| A04 Insecure Design | Positive validation (allow-list categories); amount > 0 DB constraint |
| A05 Security Misconfiguration | `SecurityHeadersMiddleware` adds CSP, HSTS, X-Content-Type-Options, X-Frame-Options |
| A06 Vulnerable Components | `dotnet list package --vulnerable` in CI; `npm audit` in CI |
| A07 Auth Failures | N/A (no auth) |
| A08 Software Integrity | Pinned NuGet and npm packages; GitHub Actions uses pinned action versions |
| A09 Logging Failures | Structured logging (Microsoft.Extensions.Logging); no sensitive data in logs |
| A10 SSRF | No outbound HTTP calls from the API |

### Security Headers Middleware

```csharp
// Common/Middleware/SecurityHeadersMiddleware.cs
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Content-Type-Options"]  = "nosniff";
    ctx.Response.Headers["X-Frame-Options"]          = "DENY";
    ctx.Response.Headers["Referrer-Policy"]          = "strict-origin-when-cross-origin";
    ctx.Response.Headers["Content-Security-Policy"]  =
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';";
    ctx.Response.Headers["Permissions-Policy"]       = "geolocation=(), microphone=()";
    if (!app.Environment.IsDevelopment())
        ctx.Response.Headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains";
    await next();
});
```

### Connection String Security

```json
// appsettings.json — placeholder only
{
  "ConnectionStrings": {
    "DefaultConnection": ""
  }
}
```
- Development: set in `appsettings.Development.json` (gitignored) or `secrets.json` (`dotnet user-secrets`)
- Production: set via Azure App Service environment variables / Key Vault reference

---

## Accessibility Requirements

> Target: WCAG 2.2 Level AA

| Requirement | Implementation |
|---|---|
| Landmarks | `<header>`, `<main>`, `<footer>` with correct roles |
| Single `<h1>` | "Personal Expense Tracker" page title |
| Skip link | First focusable element: `<a href="#main">Skip to main content</a>` |
| Keyboard navigation | All buttons, form controls, table actions reachable by Tab; no keyboard traps |
| Visible focus | CSS `:focus-visible` with 3:1 contrast on focus ring |
| Form labels | Every `<input>` / `<select>` has an associated `<label>` |
| Error messages | `aria-invalid="true"` + `aria-describedby` linking to error text |
| Required fields | `aria-required="true"` + visible asterisk `*` |
| Table semantics | `<table>` with `<th scope="col">` headers; `<caption>` |
| Icon-only buttons | `aria-label="Edit expense"` / `aria-label="Delete expense"` |
| Dialog | `role="dialog"`, `aria-modal="true"`, `aria-labelledby`, focus trapped, Escape closes |
| Color contrast | Text ≥ 4.5:1; large text ≥ 3:1; UI boundaries ≥ 3:1 |
| Color not sole cue | Error states use icon + text + color, never color alone |
| Reflow at 320px | Single-column layout; no horizontal scroll for content |

---

## Testing Strategy

### Backend (xUnit)

**Unit tests — `ExpenseService`**

```
should return all expenses ordered by date descending
should throw NotFoundException when expense id does not exist (get)
should create expense and return dto with generated id
should throw ValidationException when amount is zero or negative
should throw ValidationException when category is invalid
should update expense and refresh UpdatedAt timestamp
should throw NotFoundException when expense id does not exist (update)
should delete expense when id exists
should throw NotFoundException when expense id does not exist (delete)
should return summary with correct totalAllTime
should return summary with correct totalCurrentMonth
should return summary grouped by category
```

**Unit tests — `ExpensesController`**

```
should return 200 with expense list on GET /expenses
should return 201 with created expense on POST /expenses
should return 400 with ErrorResponse on POST with invalid amount
should return 404 with ErrorResponse on GET /expenses/{id} when not found
should return 200 with updated expense on PUT /expenses/{id}
should return 204 on DELETE /expenses/{id}
```

**Integration tests — `WebApplicationFactory` with in-memory SQLite**

```
GET /health returns 200
GET /health/ready returns 200
full CRUD cycle: create → list → get → update → delete
POST /expenses returns 400 for missing required fields
PUT /expenses/{id} returns 404 for unknown id
DELETE /expenses/{id} returns 404 for unknown id
GET /api/v1/expenses/summary returns aggregated data
```

### Frontend (Vitest + React Testing Library)

```
ExpenseForm: renders empty form with today's date pre-filled
ExpenseForm: shows validation error when amount is blank
ExpenseForm: shows validation error when amount is 0 or negative
ExpenseForm: calls onSave with correct payload on valid submit
ExpenseForm: populates fields when editing an existing expense
ExpenseList: renders table with correct headers
ExpenseList: renders one row per expense
ExpenseList: shows EmptyState when expenses array is empty
ExpenseList: calls onDelete when delete button is clicked
SummaryCard: displays totalAllTime formatted as currency
```

### Coverage Gate

- Backend: ≥ 80% line coverage (`coverlet`)
- Frontend: ≥ 80% line coverage (`@vitest/coverage-v8`)
- CI fails if either threshold is not met

---

## Infrastructure & CI/CD

### Azure Resources

| Resource | Purpose | Configuration |
|---|---|---|
| Azure App Service (B1 or higher) | Hosts the ASP.NET Core app (including React static files) | .NET 9 runtime stack |
| Azure SQL Database | SQL Server database | Basic tier for personal use |
| Azure Key Vault *(optional)* | Store connection string secret | Managed Identity from App Service |

### GitHub Actions CI Pipeline (`.github/workflows/ci.yml`)

```yaml
# Triggers: push to main, all PRs
jobs:
  backend:
    steps:
      - Checkout
      - Setup .NET 9
      - dotnet restore
      - dotnet format --verify-no-changes
      - dotnet build --no-restore
      - dotnet test --collect "XPlat Code Coverage" --results-directory coverage
      - dotnet list package --vulnerable (fail on critical/high)
      - Upload coverage artifact

  frontend:
    steps:
      - Checkout
      - Setup Node 20
      - npm ci (in client/)
      - npm run lint
      - npm run build
      - npm run test:coverage (vitest)
      - Upload coverage artifact

  coverage-gate:
    needs: [backend, frontend]
    steps:
      - Validate backend coverage >= 80%
      - Validate frontend coverage >= 80%
```

### Local Development Setup

```bash
# Prerequisites: .NET 9 SDK, Node 20, SQL Server LocalDB (or Docker SQL Server)

# 1. Clone the repo
git clone https://github.com/KyleLJohnson/personal-expense-tracker.git
cd personal-expense-tracker

# 2. Set up backend connection string (user secrets)
cd src/ExpenseTracker.Api
dotnet user-secrets set "ConnectionStrings:DefaultConnection" \
  "Server=(localdb)\\mssqllocaldb;Database=ExpenseTrackerDb;Trusted_Connection=True;"

# 3. Run EF Core migrations
dotnet ef database update

# 4. Start the backend (port 5000)
dotnet run

# 5. In a second terminal, start the frontend dev server (port 5173)
cd ../../client
npm install
npm run dev
# Open http://localhost:5173  (Vite proxies /api → http://localhost:5000)

# 6. Run all tests
dotnet test ../../tests/ExpenseTracker.Tests
npm run test --prefix ../../client
```

---

## Implementation Task List

> Tasks are ordered for sequential implementation. Use this list as the tracking checklist.

### Phase 1 — Solution Scaffold

- [ ] **T001** Create `ExpenseTracker.sln` at the repository root
- [ ] **T002** Create `src/ExpenseTracker.Api` ASP.NET Core 9 Web API project; add to solution
- [ ] **T003** Create `tests/ExpenseTracker.Tests` xUnit project; add to solution; reference API project
- [ ] **T004** Configure `.gitignore` (dotnet, node, build artifacts, user secrets)
- [ ] **T005** Add `Directory.Build.props` for shared MSBuild settings (nullable, implicit usings, warnings as errors)

### Phase 2 — Domain & Infrastructure

- [ ] **T006** Create `Expense` entity (`Modules/Expenses/Models/Expense.cs`) with all fields
- [ ] **T007** Create `ExpenseDbContext` with entity configuration (constraints, precision, default values)
- [ ] **T008** Register EF Core with SQL Server in `Program.cs` (read connection string from config)
- [ ] **T009** Create initial EF Core migration: `dotnet ef migrations add InitialCreate`
- [ ] **T010** Create `IExpenseRepository` interface with: `GetAllAsync`, `GetByIdAsync`, `AddAsync`, `UpdateAsync`, `DeleteAsync`, `GetSummaryAsync`
- [ ] **T011** Implement `ExpenseRepository` using EF Core (parameterized, no raw SQL)

### Phase 3 — Application Layer

- [ ] **T012** Create `ErrorResponse` model (code, message, details)
- [ ] **T013** Create `ExpenseDto`, `CreateExpenseRequest`, `UpdateExpenseRequest`, `ExpenseSummaryDto`
- [ ] **T014** Add Data Annotations validation to request DTOs (Required, Range, MaxLength, allowed values)
- [ ] **T015** Create `IExpenseService` interface
- [ ] **T016** Implement `ExpenseService` (map DTOs ↔ entities; enforce business rules; delegate to repository)
- [ ] **T017** Create `GlobalExceptionHandler` (IExceptionHandler .NET 9) — maps `NotFoundException` → 404, `ValidationException` → 400, all others → 500; all responses use `ErrorResponse`

### Phase 4 — API Layer

- [ ] **T018** Create `ExpensesController` with all 6 endpoints; add OpenAPI annotations (summary, operationId, response codes, parameters)
- [ ] **T019** Register `/health` and `/health/ready` endpoints (Microsoft.Extensions.Diagnostics.HealthChecks + EF Core health check)
- [ ] **T020** Configure OpenAPI in `Program.cs`: title, version, contact; enable Swagger UI only in Development/Staging
- [ ] **T021** Add `SecurityHeadersMiddleware` (CSP, X-Frame-Options, HSTS in prod, etc.)
- [ ] **T022** Add `app.UseExceptionHandler()` wired to `GlobalExceptionHandler`
- [ ] **T023** Configure model validation to return `ErrorResponse` (suppress default `ValidationProblemDetails`)
- [ ] **T024** Add HTTPS redirection and HSTS in production; configure static file serving from `wwwroot`

### Phase 5 — Backend Tests

- [ ] **T025** Write `ExpenseServiceTests` — all 12 unit test cases listed in [Testing Strategy](#testing-strategy)
- [ ] **T026** Write `ExpensesControllerTests` — all 6 unit test cases
- [ ] **T027** Write `ExpensesEndpointTests` (integration) — all 8 integration test cases; use `WebApplicationFactory` with SQLite in-memory DB
- [ ] **T028** Configure `coverlet` in test project; verify ≥ 80% coverage with `dotnet test --collect "XPlat Code Coverage"`

### Phase 6 — Frontend Scaffold

- [ ] **T029** Scaffold React + TypeScript + Vite project in `client/` (`npm create vite@latest client -- --template react-ts`)
- [ ] **T030** Configure ESLint and Prettier; add `.eslintrc.cjs` and `.prettierrc`
- [ ] **T031** Configure Vite proxy: `'/api' → 'http://localhost:5000'` in `vite.config.ts`
- [ ] **T032** Configure Vite build output to `../src/ExpenseTracker.Api/wwwroot` for production builds
- [ ] **T033** Add Motif design tokens CSS import; configure base styles
- [ ] **T034** Define TypeScript types in `types/expense.ts`
- [ ] **T035** Implement `expenseApi.ts` service (typed fetch wrappers for all 7 endpoints)
- [ ] **T036** Implement `useExpenses` custom hook (load, create, update, delete, refresh summary)

### Phase 7 — Frontend Components

- [ ] **T037** Implement `EmptyState` component with accessible message
- [ ] **T038** Implement `SummaryCard` component (totalAllTime, totalCurrentMonth, byCategory table)
- [ ] **T039** Implement `ExpenseRow` component: accessible edit/delete buttons with `aria-label`
- [ ] **T040** Implement `ExpenseList` component: accessible `<table>` with `<caption>` and `<th scope="col">` headers; renders `ExpenseRow` per expense; shows `EmptyState` when empty
- [ ] **T041** Implement `ConfirmDialog` component: `role="dialog"`, `aria-modal`, focus trap, Escape closes, accessible heading
- [ ] **T042** Implement `ExpenseForm` component: all fields with labels, validation, `aria-invalid`, `aria-describedby` for errors; today's date pre-filled; category `<select>`; accessible submit/cancel
- [ ] **T043** Implement `App` component: wire all components together; manage global state
- [ ] **T044** Add skip link `<a href="#main">Skip to main content</a>` as first focusable element
- [ ] **T045** Add page `<title>` "Personal Expense Tracker" to `index.html`

### Phase 8 — Frontend Tests

- [ ] **T046** Set up Vitest and React Testing Library in `client/`
- [ ] **T047** Write `ExpenseForm.test.tsx` — all 5 test cases
- [ ] **T048** Write `ExpenseList.test.tsx` — all 5 test cases
- [ ] **T049** Write `SummaryCard.test.tsx` — display tests
- [ ] **T050** Configure `@vitest/coverage-v8`; verify ≥ 80% coverage

### Phase 9 — CI/CD & Accessibility Verification

- [ ] **T051** Create `.github/workflows/ci.yml` with backend build/test/coverage and frontend lint/build/test/coverage jobs
- [ ] **T052** Add `dotnet list package --vulnerable` step to CI; fail on critical/high CVEs
- [ ] **T053** Add `npm audit --audit-level=high` step to CI
- [ ] **T054** Verify WCAG 2.2 AA compliance with browser accessibility tool (e.g., axe DevTools or Accessibility Insights)
- [ ] **T055** Verify keyboard navigation: Tab through all interactive elements; no traps; Escape closes dialog
- [ ] **T056** Verify color contrast ratios meet 4.5:1 (text) and 3:1 (UI boundaries) thresholds
- [ ] **T057** Verify reflow at 320px viewport width: no horizontal scroll; all content visible

### Phase 10 — Documentation & Final Review

- [ ] **T058** Update `context/architecture.md` with final system design, components, and ADRs
- [ ] **T059** Update `context/tech-stack.md` with all approved packages and versions
- [ ] **T060** Update `context/project.md` with user persona, success criteria, and key constraints
- [ ] **T061** Update root `README.md` with: project description, local dev setup instructions, CI badge, deploy instructions
- [ ] **T062** Final OWASP review: verify all 10 risks mitigated; document in a comment in `Program.cs`
- [ ] **T063** Run full test suite; confirm all tests pass and coverage thresholds are met
- [ ] **T064** Verify Swagger UI is accessible at `/docs` in Development and returns 404 in Production

---

## Definition of Done

A task is complete when:
1. Code is implemented and passes `dotnet build` / `npm run build` with zero errors
2. All relevant unit and integration tests pass
3. Code coverage ≥ 80% maintained
4. No new OWASP findings introduced
5. No `dotnet format` or ESLint violations
6. Accessibility requirements for that component verified (keyboard, ARIA, contrast)
7. Changes committed with a descriptive commit message referencing the task ID (e.g., `feat(T018): implement ExpensesController`)

---

## Confidence Score

**HIGH (92%)** — All acceptance criteria are well-defined, the tech stack is constrained by governance, and the architecture is straightforward for a single-module CRUD application. The only minor ambiguity is the exact Motif component availability for React; fall back to accessible HTML + Motif CSS tokens if Motif React wrappers are unavailable.
