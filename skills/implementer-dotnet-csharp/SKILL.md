---
name: implementer-dotnet-csharp
description: Use when implementing .NET or C# backend tasks that require production-safe contracts, reliability, and maintainable service code.
---

# Implementer .NET/C#

Apply this skill for backend implementation tasks in .NET/C#.

## Required Rules
- Keep API and domain contracts explicit and backward-compatible.
- Validate inputs and return consistent error contracts.
- Use idempotency, timeout, and retry patterns where applicable.
- Prefer simple, readable code over clever abstractions.
- Keep business logic out of controllers and transport adapters.

## Minimum Quality Gates
- Add or update unit/integration tests for changed behavior.
- Run verification commands and capture real results.
- Include operational risk notes (performance, security, migration impact).

## Output Expectations
- Changed files with rationale
- Test evidence (what was run, result)
- Risk and follow-up notes

## Architecture
Prefer clear boundaries between:
- Domain logic
- Application/use-case orchestration
- Infrastructure integrations
- Transport/API adapters

Clean Architecture is a valid default, but do not force full structural rewrites when the existing project uses another coherent pattern (for example vertical slices or feature folders).

## Core Rules
- Follow SOLID principles.
- Use Dependency Injection; avoid hard-coded service construction in business paths.
- Use Repository pattern only when it adds value (not required for simple EF Core usage).
- Avoid fat controllers - keep controllers thin, delegate to application layer.
- Use `async/await` consistently - never block with `.Result` or `.Wait()`.
- Pass `CancellationToken` through async call chains for request-scoped work.

## Validation
- Use FluentValidation or similar for input validation.
- Validate at the application boundary and enforce domain invariants in the domain model.

## Error Handling
- Use global exception middleware - don't catch and swallow exceptions in controllers.
- Return appropriate HTTP status codes.
- Normalize error payloads so clients can rely on stable fields/codes.

## Logging
- Use built-in `ILogger<T>` - don't use static loggers.
- Log at appropriate levels: Debug, Info, Warning, Error.
- Prefer structured logs with key identifiers (request id, entity id, operation).

## Anti-Patterns to Avoid
- Returning EF entities directly from API endpoints.
- Mapping transport DTOs deep inside domain methods.
- Fire-and-forget tasks inside request handlers.
- Catching broad exceptions and returning success/fallback silently.

## Verification Matrix
- Build: compile changed projects with warnings reviewed.
- Tests: run relevant unit/integration tests for changed behavior.
- Contracts: verify API response/error shape compatibility.
- Data: validate migration impact and rollback safety when schema changes exist.
- Operations: confirm timeout/retry/cancellation paths for new external calls.

