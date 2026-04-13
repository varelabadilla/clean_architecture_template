# MyProject.Architecture.Tests (v0.3)

## What Changed From v0.2

The v0.2 architecture tests all still apply. v0.3 adds tests for the new messaging boundaries introduced in the microservice context:

- Consumers must not contain business logic (only deserialize + forward)
- Integration event publishers must not be referenced from the Application layer directly
- Consumed event contracts must live in the correct folder
- The Application layer must not reference MassTransit, Kafka, or any broker-specific types

---

## Full Test Suite (v0.2 tests + v0.3 additions)

```csharp
public class LayerDependencyTests
{
    // ── v0.2 tests (unchanged) ────────────────────────────────────────────

    [Fact]
    public void Domain_ShouldNot_Reference_Application()
    {
        Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn(ApplicationAssembly.GetName().Name)
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_ShouldNot_Reference_Infrastructure()
    {
        Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn(InfrastructureAssembly.GetName().Name)
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Application_ShouldNot_Reference_Infrastructure()
    {
        Types.InAssembly(ApplicationAssembly)
            .Should().NotHaveDependencyOn(InfrastructureAssembly.GetName().Name)
            .GetResult().IsSuccessful.Should().BeTrue(
                because: "Application must use abstractions. " +
                         "IIntegrationEventPublisher is in Application/Abstractions — " +
                         "MassTransitIntegrationEventPublisher is in Infrastructure/Messaging.");
    }

    // ── v0.3 additions ────────────────────────────────────────────────────

    [Fact]
    public void Application_ShouldNot_Reference_MassTransit()
    {
        Types.InAssembly(ApplicationAssembly)
            .Should().NotHaveDependencyOn("MassTransit")
            .GetResult().IsSuccessful.Should().BeTrue(
                because: "Application layer must not know about MassTransit. " +
                         "Use IIntegrationEventPublisher abstraction instead.");
    }

    [Fact]
    public void Application_ShouldNot_Reference_RabbitMQ()
    {
        Types.InAssembly(ApplicationAssembly)
            .Should().NotHaveDependencyOn("RabbitMQ")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void Domain_ShouldNot_Reference_MassTransit()
    {
        Types.InAssembly(DomainAssembly)
            .Should().NotHaveDependencyOn("MassTransit")
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void IntegrationEvents_Should_Inherit_IntegrationEventBase()
    {
        Types.InAssembly(ApplicationAssembly)
            .That().ResideInNamespace("MyProject.Application.IntegrationEvents")
            .And().AreNotAbstract()
            .Should().Inherit(typeof(IntegrationEvent))
            .GetResult().IsSuccessful.Should().BeTrue();
    }

    [Fact]
    public void IntegrationEvents_Should_BeNamedWithCorrectSuffix()
    {
        Types.InAssembly(ApplicationAssembly)
            .That().Inherit(typeof(IntegrationEvent))
            .Should().HaveNameEndingWith("IntegrationEvent")
            .GetResult().IsSuccessful.Should().BeTrue();
    }
}

public class MessagingBoundaryTests
{
    [Fact]
    public void Consumers_Should_ResideIn_MessagingConsumersNamespace()
    {
        // MassTransit IConsumer<T> implementations must all live in Infrastructure/Messaging/Consumers
        Types.InAssembly(InfrastructureAssembly)
            .That().ImplementInterface(typeof(IConsumer<>))
            .Should().ResideInNamespace("MyProject.Infrastructure.Messaging.Consumers")
            .GetResult().IsSuccessful.Should().BeTrue(
                because: "All broker consumers must live in Infrastructure/Messaging/Consumers — " +
                         "never in Application or Domain.");
    }

    [Fact]
    public void Consumers_ShouldNot_Reference_Domain_Repositories_Directly()
    {
        // Consumers forward to MediatR — they must not bypass the application layer
        Types.InAssembly(InfrastructureAssembly)
            .That().ImplementInterface(typeof(IConsumer<>))
            .Should().NotHaveDependencyOn("MyProject.Domain.Repositories")
            .GetResult().IsSuccessful.Should().BeTrue(
                because: "Consumers must forward to MediatR, not call repositories directly. " +
                         "Business reactions belong in Application/Features/EventHandlers.");
    }

    [Fact]
    public void OutboxProcessor_Should_ResideIn_OutboxNamespace()
    {
        Types.InAssembly(InfrastructureAssembly)
            .That().HaveNameEndingWith("OutboxProcessor")
            .Should().ResideInNamespace("MyProject.Infrastructure.Outbox")
            .GetResult().IsSuccessful.Should().BeTrue();
    }
}
```

---

## Java / ArchUnit Equivalents

```java
@AnalyzeClasses(packages = "com.myproject")
public class MessagingBoundaryTests {

    @ArchTest
    static final ArchRule application_should_not_depend_on_spring_cloud_stream =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.springframework.cloud.stream..");

    @ArchTest
    static final ArchRule application_should_not_depend_on_kafka =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.apache.kafka..");

    @ArchTest
    static final ArchRule consumers_should_live_in_messaging_package =
        classes()
            .that().areAnnotatedWith(StreamListener.class)
            .should().resideInAPackage("..infrastructure.messaging.consumers..")
            .because("All broker consumers must live in infrastructure.messaging.consumers");

    @ArchTest
    static final ArchRule integration_events_should_have_correct_suffix =
        classes()
            .that().areAssignableTo(IntegrationEvent.class)
            .and().areNotAbstract()
            .should().haveSimpleNameEndingWith("IntegrationEvent");
}
```

---

## Important Notes

- The v0.3 architecture tests are additive — all v0.2 tests continue to run. Do not remove any v0.2 test when adding v0.3 tests.
- The most important new test is `Application_ShouldNot_Reference_MassTransit`. This is the single line that prevents broker coupling from creeping into the application layer over time. It will fail the moment someone adds a MassTransit `using` statement in a handler.
- Add these tests to the CI pipeline as a required check on every pull request — same as v0.2 architecture tests.
