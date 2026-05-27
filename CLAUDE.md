# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Resume and vacancy management system (Variant 5) built on .NET with a multi-layer architecture.

### Domain

Three subsystems:

1. Add/edit resumes and vacancies
2. Display with sorting, filtering, and search
3. Apply resume to vacancy or propose vacancy for resume; view linked vacancies/resumes

### User Roles

- **Administrator** — full access
- **Employer/Recruiter** — can add vacancies, search resumes by vacancy
- **Employee** — can add resumes, search vacancies by resume
- **Unauthorized user** — no access to vacancies or resumes

---

## Solution Structure

Four projects in one solution:

    Solution/
    ├── DAL/           # Data Access Layer (class library)
    ├── BLL/           # Business Logic Layer (class library)
    ├── PL/            # Presentation Layer (ASP.NET WebAPI project)
    ├── Tests/         # Unit tests (separate project, interacts with BLL only)
    └── UI/            # (optional) Web/Desktop/Mobile UI

---

## DAL (Data Access Layer)

- Class library project.
- Database: MS SQL, managed via ADO.NET Entity Framework (Code First).
- Entities reflect domain objects: Resume, Vacancy, User, Application (resume-to-vacancy link), Role.
- Access organized through repositories combined into a Unit of Work.
- **Repository pattern**: each repository provides access to a set of entities of a specific type.
- **Unit of Work (UoW)**: single access point to all repositories and the EF context.
- DAL exposes only interfaces to upper layers — no concrete classes leak upward.

### Key interfaces

- `IRepository<T>` — generic CRUD operations
- `IUnitOfWork` — aggregates all repositories, exposes `SaveChanges()`
- Concrete: `ResumeRepository`, `VacancyRepository`, `UserRepository`, `ApplicationRepository`

---

## BLL (Business Logic Layer)

- Class library project.
- Implements all application functionality described in the variant.
- Operates through `IUnitOfWork` only — never accesses DAL concrete classes directly.
- Uses mapping (manual or AutoMapper) to convert between DAL entities and BLL DTOs.
- All business operations validate inputs and throw exceptions where appropriate; custom exception classes preferred.
- Apply known design principles and patterns where relevant (e.g. Single Responsibility, DI, Strategy).

### Services

- **ResumeService** — add, edit, delete, get resumes; search vacancies matching a resume
- **VacancyService** — add, edit, delete, get vacancies; search resumes matching a vacancy
- **ApplicationService** — apply resume to vacancy; propose vacancy to resume; view linked vacancies/resumes
- **UserService** — registration, authentication, role management

### DTOs (BLL/DTOs/)

- Plain classes only — no EF or DAL types exposed.
- `OperationResult<T>` used as standard return type for mutations.

### Business rules

- Unauthorized users have no access to resumes or vacancies.
- Employer/Recruiter: can add vacancies, search resumes by vacancy.
- Employee: can add resumes, search vacancies by resume.
- Administrator: full access.
- Sorting and filtering must be supported on resume and vacancy listings.

---

## PL (Presentation Layer — ASP.NET WebAPI)

- ASP.NET WebAPI project.
- Controllers handle HTTP only: receive requests, delegate to BLL services, return responses.
- No business logic or data access in PL.
- All data in PL uses its own ViewModel/request models — never exposes BLL DTOs or DAL entities directly.
- Services injected via constructor DI (Ninject or Autofac recommended).
- Returns 404 JSON (not HTML) for unsupported routes.

### DI Container

Ninject or Autofac wires:

- `IUnitOfWork → UnitOfWork`
- `IResumeService → ResumeService`
- `IVacancyService → VacancyService`
- `IApplicationService → ApplicationService`
- `IUserService → UserService`

### Controllers and Endpoints

**ResumesController** — `/api/resumes`

- `GET    /api/resumes` — list resumes (with sorting/filtering/search query params)
- `GET    /api/resumes/{id}` — get resume by id
- `POST   /api/resumes` — create resume (Employee only)
- `PUT    /api/resumes/{id}` — update resume (Employee only)
- `DELETE /api/resumes/{id}` — delete resume

**VacanciesController** — `/api/vacancies`

- `GET    /api/vacancies` — list vacancies (with sorting/filtering/search query params)
- `GET    /api/vacancies/{id}` — get vacancy by id
- `POST   /api/vacancies` — create vacancy (Employer/Recruiter only)
- `PUT    /api/vacancies/{id}` — update vacancy (Employer/Recruiter only)
- `DELETE /api/vacancies/{id}` — delete vacancy

**ApplicationsController** — `/api/applications`

- `POST   /api/applications/apply` — apply resume to vacancy
- `POST   /api/applications/propose` — propose vacancy to resume
- `GET    /api/applications/resume/{id}` — view vacancies linked to a resume
- `GET    /api/applications/vacancy/{id}` — view resumes linked to a vacancy

**AuthController** — `/api/auth`

- `POST   /api/auth/register` — register user
- `POST   /api/auth/login` — login, returns token

### PL Models (PL/Models/)

Separate ViewModels and request models per entity:

- `ResumeViewModel`, `CreateResumeRequest`, `UpdateResumeRequest`
- `VacancyViewModel`, `CreateVacancyRequest`, `UpdateVacancyRequest`
- `ApplicationViewModel`, `ApplyRequest`, `ProposeRequest`
- `LoginRequest`, `RegisterRequest`, `UserViewModel`

### PL Mappers (PL/Mappers/)

Static or AutoMapper-based conversion between PL models and BLL DTOs.

---

## Tests (Unit Test Project)

- Separate project interacting with BLL only — no DAL or PL dependencies.
- Framework: xUnit (preferred); NUnit acceptable.
- Follows **AAA pattern** (Arrange, Act, Assert) for every test.
- Uses Moq for mocks/stubs — repositories are mocked so tests never touch real data.
- Use FluentAssertions for readable assertions.
- Test naming convention: `MethodName_Scenario_ExpectedResult`.
- One assertion concept per test.
- Minimum coverage: all methods of the service with the most business operations + unique methods of other service classes.

---

## C# / .NET 8 Coding Conventions

Target: .NET 8 / C# 12. Priorities: Readability → Consistency → Simplicity → Correctness.

### General Principles

- Write clean, readable, maintainable, production-quality C# code.
- Prefer modern C# features; avoid outdated constructs.
- Follow SOLID principles; prefer composition over inheritance.
- Fail fast: validate inputs early and throw meaningful exceptions.
- If a method needs a comment to explain _what_ it does, refactor it.

### Naming

- **PascalCase** — classes, records, structs, interfaces, enums, public properties, methods, events.
- **camelCase** — local variables, method parameters.
- **\_camelCase** — private and internal fields.
- **PascalCase** — `const` and `static readonly` constants.
- Interfaces always prefixed with `I`: `IUserService`, `IRepository<T>`.
- Generic type parameters: `T` or descriptive `TPascalCase`.
- Async methods always suffixed with `Async`: `GetOrderAsync`, `SaveChangesAsync`.
- No magic numbers — use named constants.
- Use the minimum necessary access modifier — `public` must be justified.

### File & Project Structure

- One class (or record/enum) per file; filename matches type name.
- Use file-scoped namespaces (C# 10+): `namespace MyApp.Domain.Orders;`
- Organize by feature, not by type layer where possible.

### Formatting

- Indent with 4 spaces (no tabs).
- Opening brace `{` on the same line.
- One blank line between methods.
- Max line length: 120 characters (soft limit).
- Use `var` when type is obvious from the right-hand side.
- Prefer target-typed `new` (C# 9+): `Order order = new();`

### Classes & Records

- Use `record` for immutable DTOs and value objects.
- Use `class` for mutable entities and services.
- Use `sealed` by default if inheritance is not intended.
- Prefer primary constructors (C# 12) for simple types.

### Properties

- Use auto-properties whenever possible.
- Use `init` for immutable properties on classes.
- Avoid public setters on domain entities; expose behavior through methods.

### Methods

- Methods should do one thing; keep under 30 lines.
- Use expression-bodied members for simple one-liners.
- Return early to avoid deep nesting (Guard Clause pattern).

### Async / Await

- Always use `async/await`; never block with `.Result` or `.Wait()`.
- Always accept `CancellationToken` in public async methods and pass it down.
- Use `ValueTask` for hot paths where the result is often synchronous.

### Null Handling

- Enable Nullable Reference Types in every project: `<Nullable>enable</Nullable>`
- Use `ArgumentNullException.ThrowIfNull()` for guard checks.
- Use null-coalescing operators: `var name = user?.Name ?? "Anonymous";`
- Avoid `!` (null-forgiving operator) unless absolutely necessary.

### Exception Handling

- Catch specific exceptions, not `Exception` unless at a top-level boundary.
- Never swallow exceptions silently.
- Use custom exception types for domain errors.
- Log exceptions with context before re-throwing or wrapping.

### Dependency Injection

- Register services in `Program.cs` using extension methods grouped by feature.
- Prefer constructor injection (primary constructors in C# 12).
- Use correct lifetime: `Transient`, `Scoped`, `Singleton`.

### LINQ

- Use method syntax over query syntax.
- Keep chains readable across multiple lines.
- Avoid side effects in LINQ queries.

### Collections

- Use `IReadOnlyList<T>` or `IReadOnlyCollection<T>` for exposing collections from domain models.
- Use `List<T>` internally; expose via read-only interfaces.
- Prefer `IEnumerable<T>` for method parameters when only iteration is needed.

### String Handling

- Use string interpolation over concatenation or `string.Format`.
- Use raw string literals (C# 11) for multiline strings.
- Use `string.IsNullOrWhiteSpace()` for input validation.

### Logging

- Inject `ILogger<T>` via constructor.
- Use structured logging with named parameters — not string interpolation in log calls.
- Use appropriate log levels: `LogDebug`, `LogInformation`, `LogWarning`, `LogError`, `LogCritical`.

### Comments & Documentation

- Write self-documenting code — clear naming reduces the need for comments.
- Add XML doc comments on all public APIs.
- Use inline comments only to explain **why**, never **what**.

### Project Configuration

    <PropertyGroup>
      <TargetFramework>net8.0</TargetFramework>
      <Nullable>enable</Nullable>
      <ImplicitUsings>enable</ImplicitUsings>
      <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
      <AnalysisMode>All</AnalysisMode>
      <LangVersion>latest</LangVersion>
    </PropertyGroup>

Use `GlobalUsings.cs` for common namespaces across the project.

---

## Testing the API

Use Postman or curl. Examples:

    # Get all vacancies
    curl http://localhost:5000/api/vacancies

    # Create a vacancy (Employer)
    curl -X POST http://localhost:5000/api/vacancies \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer <token>" \
      -d '{"title": "Backend Developer", "description": "...", "salary": 3000}'

    # Apply resume to vacancy
    curl -X POST http://localhost:5000/api/applications/apply \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer <token>" \
      -d '{"resumeId": 1, "vacancyId": 2}'

Note: use curl instead of PowerShell Invoke-WebRequest to avoid UTF-8 encoding issues with Cyrillic characters.

---

## Task Tracking

Use Azure DevOps (dev.azure.com) Board and Repos, or GitHub / Trello as alternatives, for task decomposition and version control.
