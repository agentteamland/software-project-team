# Compose Service Blueprint

This is the **primary production unit** of the Infra Agent. Every time a new service is added to docker-compose.yml, follow this blueprint step by step.

## .NET Host Service Template

Use this template for any new .NET service (API, consumer, worker, hub):

```yaml
  {service-name}:
    build:
      context: .
      dockerfile: src/{ProjectName}.{ServiceName}/Dockerfile.dev
    container_name: {prefix}-{service-name}
    restart: unless-stopped
    ports:                                          # Only if service exposes HTTP
      - "${SERVICE_PORT:-XXXX}:XXXX"
    environment:
      ASPNETCORE_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT:-Development}
      ASPNETCORE_URLS: http://0.0.0.0:XXXX          # Only for web hosts
      DOTNET_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT:-Development}  # For non-web hosts
      ConnectionStrings__DefaultConnection: "Host=db;Port=5432;Database=${POSTGRES_DB:-dbname};Username=${POSTGRES_USER:-postgres};Password=${POSTGRES_PASSWORD:-postgres}"
      ConnectionStrings__Redis: "redis:6379"
      ConnectionStrings__RabbitMq: "amqp://${RABBITMQ_DEFAULT_USER:-guest}:${RABBITMQ_DEFAULT_PASS:-guest}@rabbitmq:5672"
      # Add only the connection strings this service actually needs
    volumes:
      - ./src:/src/src                              # Source mount (hot reload)
      - ./{SolutionName}.sln:/src/{SolutionName}.sln
      # Shadow volumes for this service's bin/obj
      - {service}_bin:/src/src/{ProjectName}.{ServiceName}/bin
      - {service}_obj:/src/src/{ProjectName}.{ServiceName}/obj
      # Shadow volumes for shared projects this service references
      - domain_bin:/src/src/{ProjectName}.Domain/bin
      - domain_obj:/src/src/{ProjectName}.Domain/obj
      - app_bin:/src/src/{ProjectName}.Application/bin
      - app_obj:/src/src/{ProjectName}.Application/obj
      - infra_bin:/src/src/{ProjectName}.Infrastructure/bin
      - infra_obj:/src/src/{ProjectName}.Infrastructure/obj
      - logging_bin:/src/src/{ProjectName}.Logging/bin
      - logging_obj:/src/src/{ProjectName}.Logging/obj
      # NuGet cache (per service)
      - nuget_{service}:/root/.nuget/packages
    depends_on:
      db:
        condition: service_healthy                  # Only if service uses PostgreSQL
      rabbitmq:
        condition: service_healthy                  # Only if service uses RabbitMQ
      redis:
        condition: service_healthy                  # Only if service uses Redis
      elasticsearch:
        condition: service_healthy                  # Only if service uses Elasticsearch
```

**Key decisions:**
- `context: .` is always the repo root (where the .sln lives)
- `dockerfile:` points to the service's own Dockerfile.dev
- Use `ASPNETCORE_URLS` for web hosts (API, Socket), `DOTNET_ENVIRONMENT` for non-web hosts (Worker, Consumer)
- Include only the connection strings and depends_on entries that the service actually uses
- Every .NET host needs shadow volumes for its own bin/obj AND for every shared project it references

## Infrastructure Service Template

Use this template for third-party services (databases, queues, caches):

```yaml
  {service-name}:
    image: {image}:{tag}
    container_name: {prefix}-{service-name}
    restart: unless-stopped
    environment:
      # Service-specific env vars
    ports:
      - "${SERVICE_PORT:-XXXX}:XXXX"
    volumes:
      - {service}_data:/path/to/data              # Persistent data volume
    healthcheck:
      test: ["CMD-SHELL", "{health check command}"]
      interval: 5s
      timeout: 5s
      retries: 5
```

**Key decisions:**
- Always pin a major version in the image tag (e.g., `postgres:17-alpine`, `redis:7-alpine`)
- Always use `-alpine` variants when available (smaller image, faster pull)
- Every infrastructure service MUST have a healthcheck
- Persistent data volumes are named `{service}_data` (e.g., `postgres_data`, `redis_data`)
- Ports are always parameterized with `${VAR:-default}` for override flexibility

### Spatial/GIS override — PostgreSQL

If the project needs maps, location, polygons, or other spatial queries, the `db` service image changes:

```yaml
  db:
    image: imresamu/postgis:17-3.5-alpine   # NOT postgres:17-alpine, NOT postgis/postgis
    # rest of the service stays the same (env, ports, volumes, healthcheck)
```

**Why this specific image:**
- Official `postgis/postgis` does NOT publish an ARM64 manifest (as of 2026) — it fails on Apple Silicon.
- `imresamu/postgis` is maintained by PostGIS maintainer Imre Samu and ships multiarch builds (amd64 + arm64) — reliable cross-platform.
- Tag convention: `{postgres-major}-{postgis-version}-alpine` (e.g., `17-3.5-alpine`). Bump both parts when upgrading.

**Companion change:** The first migration (via database-agent) must run `CREATE EXTENSION IF NOT EXISTS postgis;`. Without the extension, PostGIS types and functions aren't available even though the image ships with them.

**Don't pull this image for non-GIS projects.** It's ~5× larger than vanilla `postgres:17-alpine` — wasted bandwidth and disk for projects that don't need spatial features.

## Frontend Service Template

Use this template for React/Vite or other frontend dev servers:

```yaml
  web:
    build:
      context: ./web
      dockerfile: Dockerfile.dev
    container_name: {prefix}-web
    restart: unless-stopped
    ports:
      - "${WEB_PORT:-3001}:3001"
    environment:
      VITE_API_URL: http://localhost:${API_PORT:-3000}
    volumes:
      - ./web:/app
      - web_node_modules:/app/node_modules         # Shadow volume for node_modules
    depends_on:
      api:
        condition: service_started                  # API doesn't need to be healthy, just running
```

**Key decisions:**
- `node_modules` is a shadow volume (never sync host's node_modules to container)
- Vite must bind to `0.0.0.0` inside the container (`--host 0.0.0.0`)
- `VITE_API_URL` uses `localhost` (browser makes the request, not the container)
- depends_on uses `service_started` (not `service_healthy`) since API may still be compiling

## Volume Registration

Every new volume used in a service definition MUST be declared in the top-level `volumes:` section at the bottom of docker-compose.yml:

```yaml
volumes:
  # Persistent data
  postgres_data:
  rabbitmq_data:
  redis_data:
  elasticsearch_data:
  # Shadow volumes for bin/obj
  {service}_bin:
  {service}_obj:
  # NuGet cache
  nuget_{service}:
  # Frontend
  web_node_modules:
```

Undeclared volumes cause `docker compose up` to fail silently or create anonymous volumes.

---

## Complete Example: Adding a New RMQ Consumer Service

Scenario: Adding a `NotificationSender` service that consumes from RabbitMQ and sends push notifications.

### Step 1: Create Dockerfile.dev

```dockerfile
# src/WalkingForMe.NotificationSender/Dockerfile.dev
FROM mcr.microsoft.com/dotnet/sdk:9.0

WORKDIR /src

# Copy all .csproj files for restore layer caching
COPY src/WalkingForMe.Domain/*.csproj src/WalkingForMe.Domain/
COPY src/WalkingForMe.Application/*.csproj src/WalkingForMe.Application/
COPY src/WalkingForMe.Infrastructure/*.csproj src/WalkingForMe.Infrastructure/
COPY src/WalkingForMe.Logging/*.csproj src/WalkingForMe.Logging/
COPY src/WalkingForMe.NotificationSender/*.csproj src/WalkingForMe.NotificationSender/
COPY WalkingForMe.sln .

RUN dotnet restore

ENTRYPOINT ["dotnet", "watch", "run", "--project", "src/WalkingForMe.NotificationSender/WalkingForMe.NotificationSender.csproj"]
```

### Step 2: Add Service to docker-compose.yml

```yaml
  notification-sender:
    build:
      context: .
      dockerfile: src/WalkingForMe.NotificationSender/Dockerfile.dev
    container_name: wfm-notification-sender
    restart: unless-stopped
    environment:
      DOTNET_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT:-Development}
      ConnectionStrings__RabbitMq: "amqp://${RABBITMQ_DEFAULT_USER:-guest}:${RABBITMQ_DEFAULT_PASS:-guest}@rabbitmq:5672"
      ConnectionStrings__Redis: "redis:6379"
      Firebase__ProjectId: ${FIREBASE_PROJECT_ID:-demo-project}
      Firebase__CredentialsPath: /run/secrets/firebase.json
    volumes:
      - ./src:/src/src
      - ./WalkingForMe.sln:/src/WalkingForMe.sln
      - notificationsender_bin:/src/src/WalkingForMe.NotificationSender/bin
      - notificationsender_obj:/src/src/WalkingForMe.NotificationSender/obj
      - domain_bin:/src/src/WalkingForMe.Domain/bin
      - domain_obj:/src/src/WalkingForMe.Domain/obj
      - app_bin:/src/src/WalkingForMe.Application/bin
      - app_obj:/src/src/WalkingForMe.Application/obj
      - infra_bin:/src/src/WalkingForMe.Infrastructure/bin
      - infra_obj:/src/src/WalkingForMe.Infrastructure/obj
      - logging_bin:/src/src/WalkingForMe.Logging/bin
      - logging_obj:/src/src/WalkingForMe.Logging/obj
      - nuget_notificationsender:/root/.nuget/packages
    depends_on:
      rabbitmq:
        condition: service_healthy
      redis:
        condition: service_healthy
```

### Step 3: Register Volumes

Add to the `volumes:` section:

```yaml
  notificationsender_bin:
  notificationsender_obj:
  nuget_notificationsender:
```

### Step 4: Update .env.example

```bash
# Notification Sender
FIREBASE_PROJECT_ID=demo-project
```

### Step 5: Run the Checklist

See below.

---

## Pre-Merge Checklist

Run through this checklist before every compose change is complete:

- [ ] **Container name prefixed?** `{prefix}-{service-name}` (e.g., `wfm-notification-sender`)
- [ ] **Health check defined?** Every infrastructure service has a healthcheck block
- [ ] **depends_on with condition?** Uses `service_healthy` (not bare `depends_on`)
- [ ] **Shadow volumes for bin/obj?** Service's own + all shared project bin/obj
- [ ] **NuGet cache volume?** `nuget_{service}` declared and mounted
- [ ] **Source mount?** `./src:/src/src` and `./{SolutionName}.sln:/src/{SolutionName}.sln`
- [ ] **Environment variables?** Only the connection strings this service needs
- [ ] **Port in .env?** If service exposes a port, it uses `${VAR:-default}` pattern
- [ ] **.env.example updated?** New variables added with safe defaults and comments
- [ ] **Volumes declared?** All new volumes listed in the top-level `volumes:` section
- [ ] **Dockerfile.dev created?** Follows the standard pattern (copy csproj, restore, entrypoint)
- [ ] **restart policy?** Set to `unless-stopped`
- [ ] **Build context correct?** `context: .` (repo root) with `dockerfile:` pointing to service path

---
