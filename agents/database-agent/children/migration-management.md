# Migration Management

## Golden Rule

**Migrations are ALWAYS created inside the Docker container.** Never use local dotnet tooling.

```bash
# Create a new migration
docker compose exec api dotnet ef migrations add {MigrationName} \
  --project ../WalkingForMe.Infrastructure \
  --startup-project .

# Apply pending migrations
docker compose exec api dotnet ef database update \
  --project ../WalkingForMe.Infrastructure \
  --startup-project .

# Revert to a specific migration
docker compose exec api dotnet ef database update {TargetMigrationName} \
  --project ../WalkingForMe.Infrastructure \
  --startup-project .

# Remove the last migration (if not yet applied)
docker compose exec api dotnet ef migrations remove \
  --project ../WalkingForMe.Infrastructure \
  --startup-project .

# Generate SQL script (for production review)
docker compose exec api dotnet ef migrations script \
  --project ../WalkingForMe.Infrastructure \
  --startup-project . \
  --idempotent
```

## Naming Convention

Format: `{Timestamp}_{Description}`

EF Core auto-generates the timestamp. The description should be clear and concise:

| Pattern | Example |
|---------|---------|
| Add entity | `AddOrders` |
| Add column | `AddStatusToOrders` |
| Add index | `AddIndexOnOrdersCreatedAt` |
| Add relationship | `AddCustomerOrderRelationship` |
| Remove column | `RemoveTemporaryFieldFromUsers` |
| Rename column | `RenameUserNameToUsernameInUsers` |
| Add constraint | `AddUniqueConstraintOnOrderNumber` |
| Seed data | `SeedDefaultRoles` |

**Rules:**
- PascalCase, no spaces or special characters
- Verb + Noun pattern
- Be specific about what changed and where
- Never use generic names like `UpdateDatabase` or `FixStuff`

## Development Workflow

### Auto-Migrate in Development

In `Program.cs`, Development environment auto-applies migrations on startup:

```csharp
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
    await db.Database.MigrateAsync();
}
```

This means: create the migration, restart the container, schema is updated.

### Step-by-Step: New Migration

1. Make changes to entity / configuration
2. Rebuild the container: `docker compose up --build api`
3. Create migration:
   ```bash
   docker compose exec api dotnet ef migrations add AddStatusToOrders \
     --project ../WalkingForMe.Infrastructure \
     --startup-project .
   ```
4. Review the generated migration file — check for unintended changes
5. Restart to auto-apply: `docker compose restart api`
6. Verify: check the database via Adminer (http://localhost:8083) or psql

### Step-by-Step: Fix a Bad Migration (Not Yet Applied in Prod)

1. Revert the migration:
   ```bash
   docker compose exec api dotnet ef database update {PreviousMigration} \
     --project ../WalkingForMe.Infrastructure \
     --startup-project .
   ```
2. Remove the migration:
   ```bash
   docker compose exec api dotnet ef migrations remove \
     --project ../WalkingForMe.Infrastructure \
     --startup-project .
   ```
3. Fix the entity/configuration
4. Create a new migration

## Production Workflow

### Never Auto-Migrate in Production

Production uses explicit migration scripts:

```bash
# Generate idempotent SQL script
docker compose exec api dotnet ef migrations script \
  --project ../WalkingForMe.Infrastructure \
  --startup-project . \
  --idempotent \
  -o /tmp/migration.sql
```

The script is reviewed, approved, and applied manually or via CI/CD pipeline.

### Production Checklist

- [ ] Migration script generated and reviewed
- [ ] No data-destructive operations (column drops, type changes)
- [ ] Backward-compatible changes (add nullable column, add index)
- [ ] Estimated execution time for large tables
- [ ] Rollback script prepared
- [ ] Database backup taken before applying

## Handling Merge Conflicts

### ModelSnapshot Conflicts

The `ApplicationDbContextModelSnapshot.cs` file is the most common source of merge conflicts.

**Resolution strategy:**

1. Accept the incoming snapshot entirely (theirs)
2. Remove the conflicting migration
3. Recreate your migration on top of the latest snapshot

```bash
# Step 1: Accept their snapshot
git checkout --theirs src/WalkingForMe.Infrastructure/Persistence/Migrations/ApplicationDbContextModelSnapshot.cs

# Step 2: Remove your migration files (the .cs and .Designer.cs)
rm src/.../Migrations/{YourMigration}.cs
rm src/.../Migrations/{YourMigration}.Designer.cs

# Step 3: Recreate your migration
docker compose exec api dotnet ef migrations add {YourMigrationName} \
  --project ../WalkingForMe.Infrastructure \
  --startup-project .
```

### Conflicting Migrations with Same Timestamp

If two developers create migrations simultaneously, one will need to regenerate. The developer whose migration is not yet in `main` recreates theirs.

## Empty Migration for Data Seeding

Use empty migrations for seed data that must run in a specific order:

```bash
docker compose exec api dotnet ef migrations add SeedDefaultRoles \
  --project ../WalkingForMe.Infrastructure \
  --startup-project .
```

Then edit the generated migration:

```csharp
public partial class SeedDefaultRoles : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("""
            INSERT INTO roles (id, name, created_at)
            VALUES
                (gen_random_uuid(), 'Admin', now()),
                (gen_random_uuid(), 'User', now())
            ON CONFLICT (name) DO NOTHING;
        """);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("""
            DELETE FROM roles WHERE name IN ('Admin', 'User');
        """);
    }
}
```

**Rules for seed migrations:**
- Always include `ON CONFLICT DO NOTHING` for idempotency
- Always implement `Down()` for reversibility
- Use raw SQL for seed data — EF's `HasData()` generates noisy diffs

## Migration File Structure

```
src/{ProjectName}.Infrastructure/
  Persistence/
    Migrations/
      {Timestamp}_InitialCreate.cs
      {Timestamp}_InitialCreate.Designer.cs
      {Timestamp}_AddOrders.cs
      {Timestamp}_AddOrders.Designer.cs
      ApplicationDbContextModelSnapshot.cs    ← auto-generated, never edit manually
```

**Never manually edit `*.Designer.cs` or `ApplicationDbContextModelSnapshot.cs`.** These are auto-generated. Only edit the main migration file (`Up` / `Down` methods) when needed.
