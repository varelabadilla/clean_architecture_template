# Tests

## Why This Folder Exists

Every production layer has a corresponding test project. Tests are not an afterthought — they are the mechanism that validates architectural decisions, catches regressions, and documents behavior.

This folder contains five test projects, each targeting a distinct layer and testing strategy.

---

## Test Project Overview

| Project | Type | What it tests |
| --- | --- | --- |
| `Domain.Tests` | Unit | Entity invariants, value objects, specifications, domain services |
| `Application.Tests` | Unit | Handlers, validators, behavior pipeline logic |
| `Infrastructure.Tests` | Integration | Repository queries, EF mappings, external service adapters |
| `Api.Tests` | Integration / E2E | HTTP contracts, middleware, authentication, full request pipeline |
| `Architecture.Tests` | Static analysis | Dependency rule enforcement — no layer references a layer it shouldn't |

---

## Testing Pyramid

```
        [Api.Tests]          ← Fewest, slowest, most realistic
       /             \
  [Infra.Tests]       \      ← Integration tests, real DB/services
     /                 \
[App.Tests]        [Arch.Tests]   ← Unit + static analysis
   /
[Domain.Tests]                    ← Most, fastest, pure logic
```

Invest most heavily in Domain and Application unit tests. They are fast, reliable, and test the code that matters most — business logic and use case orchestration.

---

## Shared Test Utilities

Consider a `MyProject.Tests.Common` project (not shown in the structure by default) for:
- Test data builders / Object Mother pattern
- Shared fakes and stubs (`FakeEmailService`, `FakeDateTimeProvider`)
- Base test class setups
- Database test fixtures

---

## Recommended Testing Libraries

### .NET
| Concern | Library |
| --- | --- |
| Test runner | xUnit |
| Assertions | FluentAssertions |
| Mocking | NSubstitute or Moq |
| Integration DB | Testcontainers, EF InMemory (with caveats) |
| Architecture | NetArchTest.Rules |
| API testing | `WebApplicationFactory<Program>` |

### Java
| Concern | Library |
| --- | --- |
| Test runner | JUnit 5 |
| Assertions | AssertJ |
| Mocking | Mockito |
| Integration DB | Testcontainers |
| Architecture | ArchUnit |
| API testing | MockMvc or RestAssured |

---

## Important Notes

- Never use EF InMemory provider for integration tests that involve complex queries or transactions — it does not behave like a real relational database. Use Testcontainers with the real database engine instead.
- Architecture tests should run in CI on every pull request. They are the enforcement mechanism for the dependency rules documented in the root `README.md`.
- Test projects reference production projects directly for their layer, plus `Tests.Common` if shared. They never reference layers they are not testing (e.g., `Domain.Tests` should not reference `Infrastructure`).
