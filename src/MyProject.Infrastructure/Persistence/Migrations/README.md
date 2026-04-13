# Infrastructure / Persistence / Migrations

## Why This Folder Exists

This folder holds the **auto-generated EF Core database migrations** that track and apply incremental schema changes over time. Migrations provide a version-controlled history of the database schema, enabling reproducible deployments across all environments.

---

## What Belongs Here

- EF Core generated migration files (`{Timestamp}_{MigrationName}.cs`)
- The `AppDbContextModelSnapshot.cs` file (EF's current schema snapshot)

---

## What Does NOT Belong Here

- Hand-written SQL scripts
- Seed data logic (use a separate initializer class)
- Rollback scripts (EF handles Down() methods)
- Any business logic

---

## Naming Conventions

Migration names should describe **what changes**, not when:

| Good names | Bad names |
| --- | --- |
| `AddOrdersTable` | `Migration1` |
| `AddCustomerEmailIndex` | `Update20241201` |
| `RenameOrderStatusColumn` | `Fix` |
| `AddSoftDeleteToCustomers` | `Changes` |

---

## Common Commands

**.NET CLI:**

```bash
# Create a new migration
dotnet ef migrations add AddOrdersTable \
  --project src/MyProject.Infrastructure \
  --startup-project src/MyProject.Api

# Apply pending migrations
dotnet ef database update \
  --project src/MyProject.Infrastructure \
  --startup-project src/MyProject.Api

# Roll back to a specific migration
dotnet ef database update AddCustomerEmailIndex \
  --project src/MyProject.Infrastructure \
  --startup-project src/MyProject.Api

# Generate SQL script for production deployment
dotnet ef migrations script \
  --project src/MyProject.Infrastructure \
  --startup-project src/MyProject.Api \
  --output migrations.sql \
  --idempotent
```

---

## Important Notes

- **Never edit generated migration files manually.** If a migration is wrong, add a new one or delete the pending one and regenerate it.
- The `--idempotent` flag generates a SQL script that is safe to run multiple times — it only applies migrations that haven't been applied yet. Use this for production deployments.
- In CI/CD pipelines, prefer running migrations programmatically at startup (`context.Database.MigrateAsync()`) in non-production environments, and using SQL scripts reviewed by a DBA in production.
- In Java/Spring Boot, the equivalent is Flyway or Liquibase. Migration files go in `src/main/resources/db/migration/` and follow Flyway's `V{version}__{description}.sql` naming convention.
