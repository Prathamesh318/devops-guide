# 🎼 Chapter 9: Docker Compose

> **"One command to rule them all — define, build, and run your entire application stack."**

---

## 🎯 What is Docker Compose?

```
┌──────────────────────────────────────────────────────────────────┐
│                 DOCKER COMPOSE                                    │
│                                                                   │
│  WITHOUT Compose (painful):                                      │
│  ────────────────────────────────────────────                     │
│  docker network create myapp-net                                │
│  docker volume create pg-data                                   │
│  docker volume create redis-data                                │
│  docker run -d --name db --network myapp-net -v pg-data:...     │
│  docker run -d --name cache --network myapp-net -v redis-data:..│
│  docker run -d --name api --network myapp-net -p 3000:3000 ...  │
│  docker run -d --name web --network myapp-net -p 80:80 ...      │
│  # 😰 And remember ALL these commands for every environment!    │
│                                                                   │
│  WITH Compose (simple):                                          │
│  ────────────────────────────────────────────                     │
│  docker compose up -d                                            │
│  # 😊 One command. Done.                                        │
│                                                                   │
│  Compose lets you define multi-container apps in a YAML file.    │
│  Networks, volumes, env vars, ports — everything in one place.   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📋 Compose File Structure

```yaml
# docker-compose.yml (or compose.yml — both work)

# Version is optional in Compose V2 (Docker Compose plugin)
# version: "3.8"  ← No longer needed

services:
  # ─── Service 1: Web Application ───
  web:
    build: .                     # Build from Dockerfile in current dir
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    depends_on:
      - db
      - redis
    networks:
      - frontend
      - backend

  # ─── Service 2: Database ───
  db:
    image: postgres:15-alpine
    volumes:
      - pg-data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    networks:
      - backend

  # ─── Service 3: Cache ───
  redis:
    image: redis:alpine
    volumes:
      - redis-data:/data
    networks:
      - backend

# Named volumes
volumes:
  pg-data:
  redis-data:

# Networks
networks:
  frontend:
  backend:
```

---

## 🔧 Essential Compose Commands

```bash
# Start all services (in background)
docker compose up -d

# Start with build (rebuild images)
docker compose up -d --build

# Stop all services
docker compose down

# Stop and remove volumes too (⚠️ deletes data!)
docker compose down -v

# View running services
docker compose ps

# View logs
docker compose logs
docker compose logs -f               # Follow (real-time)
docker compose logs -f web           # Specific service only

# Scale a service
docker compose up -d --scale web=3   # Run 3 instances of 'web'

# Restart specific service
docker compose restart web

# Execute command in running service
docker compose exec web bash
docker compose exec db psql -U admin -d myapp

# Run one-off command
docker compose run --rm web npm test

# View resource usage
docker compose top
```

---

## 🏗️ Complete Real-World Examples

### Example 1: Full-Stack Web Application

```yaml
# compose.yml — Complete web app stack
services:
  # ─── Frontend (React) ───
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:5000
    depends_on:
      - api
    networks:
      - frontend-net

  # ─── Backend API (Node.js) ───
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://admin:secret@db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=my-super-secret
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - frontend-net
      - backend-net

  # ─── Database (PostgreSQL) ───
  db:
    image: postgres:15-alpine
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Auto-run on first start
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend-net

  # ─── Cache (Redis) ───
  redis:
    image: redis:alpine
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redis-data:/data
    networks:
      - backend-net

  # ─── Reverse Proxy (Nginx) ───
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - frontend
      - api
    networks:
      - frontend-net

volumes:
  pg-data:
  redis-data:

networks:
  frontend-net:
  backend-net:
```

### Example 2: Development Environment with Hot Reload

```yaml
# compose.dev.yml — Development with live reload
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "5000:5000"
      - "9229:9229"     # Node.js debugger port
    volumes:
      - .:/app           # Mount source code (live reload!)
      - /app/node_modules # Protect node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=app:*
    command: npm run dev  # Override CMD for dev (e.g., nodemon)
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    ports:
      - "5432:5432"      # Expose for local DB tools
    volumes:
      - pg-dev-data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev

  adminer:
    image: adminer:latest
    ports:
      - "8080:8080"       # Database admin UI
    depends_on:
      - db

volumes:
  pg-dev-data:
```

```bash
# Run with specific compose file
docker compose -f compose.dev.yml up -d

# Or use multiple files (merge override into base)
docker compose -f compose.yml -f compose.dev.yml up -d
```

---

## 🔑 Key Compose Features

### Environment Variables

```yaml
services:
  app:
    image: myapp
    environment:
      # Direct values
      - DB_HOST=postgres
      - DB_PORT=5432
      
    # OR from file
    env_file:
      - .env
      - .env.local
      
    # OR object syntax
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
```

```bash
# .env file (auto-loaded by Compose)
POSTGRES_PASSWORD=supersecret
APP_VERSION=2.1.0

# Use in compose.yml
services:
  db:
    image: postgres:${POSTGRES_VERSION:-15}  # Default: 15
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

### Depends On (Service Ordering)

```yaml
services:
  api:
    depends_on:
      # Simple (just wait for container to start)
      - redis
      
      # With health check (wait for service to be READY)
      db:
        condition: service_healthy
      
      # Wait for one-time setup to complete
      migrate:
        condition: service_completed_successfully

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      timeout: 5s
      retries: 5

  migrate:
    image: myapp
    command: python manage.py migrate
    depends_on:
      db:
        condition: service_healthy
```

### Build Options

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      args:
        - VERSION=2.0
        - BUILD_DATE=2024-01-15
      target: production          # Multi-stage: build specific stage
      cache_from:
        - myapp:latest
      platforms:
        - linux/amd64
        - linux/arm64
    image: myapp:latest           # Tag the built image
```

### Resource Limits

```yaml
services:
  api:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
      replicas: 3
      restart_policy:
        condition: on-failure
        max_attempts: 5
```

### Healthchecks

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### Profiles (Run Subset of Services)

```yaml
services:
  web:
    image: myapp        # Always starts
    
  db:
    image: postgres     # Always starts
    
  adminer:
    image: adminer
    profiles: ["debug"]  # Only starts with --profile debug
    
  prometheus:
    image: prom/prometheus
    profiles: ["monitoring"]

# Usage:
# docker compose up -d                          → web + db only
# docker compose --profile debug up -d          → web + db + adminer
# docker compose --profile monitoring up -d     → web + db + prometheus
# docker compose --profile debug --profile monitoring up -d  → all
```

---

## 📁 Multiple Compose Files Strategy

```
project/
├── compose.yml           # Base configuration
├── compose.dev.yml       # Development overrides
├── compose.prod.yml      # Production overrides
├── compose.test.yml      # Testing overrides
├── .env                  # Default env variables
├── .env.production       # Production env variables
└── docker-compose.yml    # Legacy name (still works)
```

```bash
# Development
docker compose -f compose.yml -f compose.dev.yml up -d

# Production
docker compose -f compose.yml -f compose.prod.yml up -d

# Testing
docker compose -f compose.yml -f compose.test.yml up -d

# Or set COMPOSE_FILE env variable
export COMPOSE_FILE=compose.yml:compose.dev.yml
docker compose up -d
```

---

## 🧠 Memory Shortcuts for This Chapter

### Compose commands: **"UDLS"**
```
U = Up (start all services)
D = Down (stop all services)
L = Logs (view output)
S = Scale (increase replicas)
```

### Compose file sections: **"SVNE"**
```
S = Services (containers to run)
V = Volumes (persistent storage)
N = Networks (communication)
E = Environment (variables, env_file)
```

---

## ❓ Quick Quiz

1. What is Docker Compose and why is it needed?
2. What does `depends_on` with `condition: service_healthy` do?
3. How do you run development and production with different settings?
4. What is the difference between `docker compose up` and `docker compose run`?
5. How do you use profiles in Compose?

<details>
<summary>Click for Answers</summary>

1. Docker Compose defines multi-container applications in a YAML file. Instead of running many `docker run` commands, one `docker compose up` starts everything.
2. It waits for the dependency service's healthcheck to pass before starting the dependent service. Without the health condition, it only waits for the container to start (not be ready).
3. Use multiple compose files: a base `compose.yml` and override files like `compose.dev.yml` and `compose.prod.yml`. Merge them with `-f` flags.
4. `up` starts all services defined in the compose file. `run` creates and starts a single one-off container from a service definition (like ad-hoc commands).
5. Add `profiles: ["name"]` to services. They only start when you use `--profile name` flag. Great for optional services like debug tools.

</details>

---

**← Previous: [08 - Storage & Volumes](./08-storage-volumes.md)** | **Next: [10 - Registry & Image Management](./10-registry-image-management.md)** ➡️
