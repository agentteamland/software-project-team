# Known Issues & Lessons Learned

Issues discovered during real project scaffolding and development. These must be considered when creating new projects or features.

## EF Core Relational Package Must Be Explicit

**Discovered:** 2026-04-15 during reference-project scaffolding

**Problem:** `Npgsql.EntityFrameworkCore.PostgreSQL` should transitively bring `Microsoft.EntityFrameworkCore.Relational`, but in .NET 9 a version mismatch can occur causing `FileNotFoundException: Could not load file or assembly 'Microsoft.EntityFrameworkCore.Relational'` at startup during migration.

**Fix:** Always explicitly add `Microsoft.EntityFrameworkCore.Relational` to the Infrastructure project:

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.Relational" Version="9.*" />
```

**Apply when:** Every new project scaffolding. Never rely on transitive dependency for EF Core Relational.

---

## MinIO Init Container Shows as "Exited"

**Discovered:** 2026-04-15 during reference-project scaffolding

**Problem:** `wfm-minio-init` container appears as "exited" in Docker Desktop, which can cause concern. This is normal behavior — it's a one-shot init container that creates the default bucket and then exits.

**Fix:** Document in CLAUDE.md and README that `minio-init` is expected to exit after first run. Not a bug.

**Apply when:** Every project using MinIO/S3 file storage.
