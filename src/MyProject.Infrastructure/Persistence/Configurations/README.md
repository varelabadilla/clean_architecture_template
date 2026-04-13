# Infrastructure / Persistence / Configurations

## Why This Folder Exists

This folder contains the **EF Core fluent API configurations** that map domain entities to database tables. By isolating all mapping here, domain entities remain free of ORM annotations — no `[Table]`, `[Column]`, `[Key]`, or `[ForeignKey]` attributes ever appear on a domain class.

Each entity gets its own configuration class implementing `IEntityTypeConfiguration<T>`, and all configurations are applied automatically via `ApplyConfigurationsFromAssembly()` in `AppDbContext`.

---

## What Belongs Here

- One `IEntityTypeConfiguration<T>` class per entity (aggregate roots and child entities)
- Value object owned entity configurations
- Index definitions and unique constraints
- Table and column name overrides
- Enum-to-string or enum-to-int conversion configurations

---

## What Does NOT Belong Here

- Business logic
- Migration files (those go in `Migrations/`)
- Seed data scripts (use a separate `DataSeeder` class or EF's `HasData`)
- Query filters as business rules (soft-delete global filters are acceptable here)

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Configuration class | EntityName + `Configuration` | `OrderConfiguration`, `OrderItemConfiguration` |
| Table name | snake_case plural | `orders`, `order_items` |
| Column name | snake_case | `customer_id`, `created_at` |
| Primary key | `id` | standard |

---

## Configuration Examples

### Aggregate root

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");

        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id)
            .HasConversion(id => id.Value, value => OrderId.From(value))
            .HasColumnName("id");

        builder.Property(o => o.CustomerId)
            .HasConversion(id => id.Value, value => CustomerId.From(value))
            .HasColumnName("customer_id")
            .IsRequired();

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasColumnName("status")
            .HasMaxLength(50)
            .IsRequired();

        // Value object: owned entity
        builder.OwnsOne(o => o.TotalAmount, money =>
        {
            money.Property(m => m.Amount)
                .HasColumnName("total_amount")
                .HasPrecision(18, 4);

            money.Property(m => m.Currency)
                .HasColumnName("total_currency")
                .HasMaxLength(3);
        });

        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("order_id")
            .OnDelete(DeleteBehavior.Cascade);

        builder.Property(o => o.CreatedAt)
            .HasColumnName("created_at");

        builder.HasIndex(o => o.CustomerId)
            .HasDatabaseName("ix_orders_customer_id");
    }
}
```

### Child entity

```csharp
public class OrderItemConfiguration : IEntityTypeConfiguration<OrderItem>
{
    public void Configure(EntityTypeBuilder<OrderItem> builder)
    {
        builder.ToTable("order_items");

        builder.HasKey(i => i.Id);
        builder.Property(i => i.Id)
            .HasConversion(id => id.Value, value => OrderItemId.From(value))
            .HasColumnName("id");

        builder.Property(i => i.ProductId)
            .HasConversion(id => id.Value, value => ProductId.From(value))
            .HasColumnName("product_id")
            .IsRequired();

        builder.Property(i => i.Quantity)
            .HasColumnName("quantity")
            .IsRequired();

        builder.OwnsOne(i => i.UnitPrice, money =>
        {
            money.Property(m => m.Amount)
                .HasColumnName("unit_price")
                .HasPrecision(18, 4);

            money.Property(m => m.Currency)
                .HasColumnName("unit_currency")
                .HasMaxLength(3);
        });
    }
}
```

---

## Important Notes

- Always use **snake_case** for table and column names if targeting PostgreSQL or MySQL. SQL Server tolerates PascalCase but snake_case is more portable.
- Configure **strongly-typed IDs** using `.HasConversion()`. Without this, EF will not know how to serialize `OrderId` to a `Guid` column.
- Value objects modeled as owned entities use `.OwnsOne()`. Their columns are stored inline in the parent table unless you use `.OwnsOne(..., b => b.ToTable("..."))` to split them.
- Apply all configurations via `modelBuilder.ApplyConfigurationsFromAssembly(...)` — never register them individually in `OnModelCreating`. This scales automatically as new entities are added.
- In Java/JPA, the equivalent configurations are `@Embeddable`/`@Embedded` for value objects, `@OneToMany` for collections, and `@AttributeConverter` for strongly-typed IDs. Keep these annotations on separate JPA entity classes in `Infrastructure/Persistence/`, not on the domain entities.
