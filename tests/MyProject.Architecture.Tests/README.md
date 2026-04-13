# MyProject.Architecture.Tests

## Why This Project Exists

Architecture tests are **automated enforcement of the dependency rules** that define Clean Architecture. Without them, the rules are just documentation — eventually someone adds a reference "just this once" and the architecture silently degrades.

With architecture tests running in CI on every pull request, dependency rule violations become build failures. The architecture is self-defending.

---

## What to Test Here

- The dependency rule: Domain does not reference Application, Infrastructure, or Api
- Application does not reference Infrastructure or Api
- Infrastructure does not reference Api
- Domain entities do not use ORM annotations
- Controllers do not reference Domain types directly
- No layer skips the one above it (Api should not call repositories directly)

---

## What NOT to Test Here

- Business behavior (Domain.Tests)
- HTTP contracts (Api.Tests)
- Any test that requires a running application

---

## Recommended Project Structure

```plaintext
MyProject.Architecture.Tests/
├─ LayerDependencyTests.cs     // the most important tests
├─ NamingConventionTests.cs    // optional: validate naming rules
└─ AnnotationTests.cs          // optional: validate no ORM annotations on domain
```

---

## .NET Implementation with NetArchTest

```csharp
public class LayerDependencyTests
{
    private static readonly Assembly DomainAssembly =
        typeof(Order).Assembly;

    private static readonly Assembly ApplicationAssembly =
        typeof(PlaceOrderCommand).Assembly;

    private static readonly Assembly InfrastructureAssembly =
        typeof(AppDbContext).Assembly;

    private static readonly Assembly ApiAssembly =
        typeof(OrdersController).Assembly;

    [Fact]
    public void Domain_ShouldNot_ReferenceTo_Application()
    {
        var result = Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn(ApplicationAssembly.GetName().Name)
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Domain must not depend on Application. " +
                     "Violations: " + string.Join(", ", result.FailingTypeNames ?? []));
    }

    [Fact]
    public void Domain_ShouldNot_ReferenceTo_Infrastructure()
    {
        var result = Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn(InfrastructureAssembly.GetName().Name)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_ShouldNot_ReferenceTo_Api()
    {
        var result = Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn(ApiAssembly.GetName().Name)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNot_ReferenceTo_Infrastructure()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .Should().NotHaveDependencyOn(InfrastructureAssembly.GetName().Name)
            .GetResult();

        result.IsSuccessful.Should().BeTrue(
            because: "Application must not depend on Infrastructure. " +
                     "Use abstractions defined in Application/Abstractions instead.");
    }

    [Fact]
    public void Application_ShouldNot_ReferenceTo_Api()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .Should().NotHaveDependencyOn(ApiAssembly.GetName().Name)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Infrastructure_ShouldNot_ReferenceTo_Api()
    {
        var result = Types.InAssembly(InfrastructureAssembly)
            .Should().NotHaveDependencyOn(ApiAssembly.GetName().Name)
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }
}
```

---

## Optional: Naming Convention Tests

```csharp
public class NamingConventionTests
{
    [Fact]
    public void CommandHandlers_Should_BeNamedWithHandlerSuffix()
    {
        var result = Types.InAssembly(ApplicationAssembly)
            .That().ImplementInterface(typeof(IRequestHandler<,>))
            .Should().HaveNameEndingWith("Handler")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void DomainEvents_Should_BeNamedWithEventSuffix()
    {
        var result = Types.InAssembly(DomainAssembly)
            .That().ImplementInterface(typeof(IDomainEvent))
            .Should().HaveNameEndingWith("Event")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Repositories_In_Infrastructure_Should_BeNamedCorrectly()
    {
        var result = Types.InAssembly(InfrastructureAssembly)
            .That().ResideInNamespace("MyProject.Infrastructure.Persistence.Repositories")
            .Should().HaveNameEndingWith("Repository")
            .GetResult();

        result.IsSuccessful.Should().BeTrue();
    }
}
```

---

## Java Implementation with ArchUnit

```java
@AnalyzeClasses(packages = "com.myproject")
public class LayerDependencyTests {

    @ArchTest
    static final ArchRule domain_should_not_depend_on_application =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..application..");

    @ArchTest
    static final ArchRule domain_should_not_depend_on_infrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");

    @ArchTest
    static final ArchRule application_should_not_depend_on_infrastructure =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");

    @ArchTest
    static final ArchRule domain_entities_should_not_use_jpa_annotations =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().beAnnotatedWith(javax.persistence.Entity.class)
            .orShould().beAnnotatedWith(javax.persistence.Table.class);
}
```

---

## CI Integration

Add architecture tests to the CI pipeline as a separate step that runs on every pull request:

```yaml
# .github/workflows/ci.yml
- name: Run Architecture Tests
  run: dotnet test tests/MyProject.Architecture.Tests --no-build
```

Fail the build if any architecture test fails. This is non-negotiable — the tests exist precisely to prevent "just this once" violations from accumulating.

---

## Important Notes

- Architecture tests are **static** — they analyze assemblies, not runtime behavior. They run in milliseconds.
- The failure message should be descriptive enough to tell the developer exactly which type violated which rule and what they should do instead. Add `because:` clauses to your assertions.
- This project references all production assemblies but only for type resolution — it never calls their code.
- Consider adding tests for: no `public` constructors on value objects (use factory methods), no `static` dependencies in handlers, and all domain exceptions extending `DomainException`.
