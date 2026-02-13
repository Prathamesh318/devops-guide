# 📋 Chapter 14: Docker Cheatsheet

> **"Keep this page bookmarked — it covers 95% of daily Docker commands."**

---

## 🐳 Container Commands

| Command | Description |
|---------|-------------|
| `docker run -d --name web -p 80:80 nginx` | Run container (detached, named, port-mapped) |
| `docker run -it ubuntu bash` | Run interactive container with shell |
| `docker run --rm alpine echo hello` | Run and auto-remove on exit |
| `docker run -d -e KEY=value myapp` | Run with environment variable |
| `docker run -d --restart=always nginx` | Run with restart policy |
| `docker run -d --memory=256m --cpus=1 myapp` | Run with resource limits |
| `docker ps` | List running containers |
| `docker ps -a` | List ALL containers (including stopped) |
| `docker stop <name>` | Graceful stop (SIGTERM → SIGKILL) |
| `docker kill <name>` | Force stop (SIGKILL immediately) |
| `docker start <name>` | Start stopped container |
| `docker restart <name>` | Restart container |
| `docker rm <name>` | Remove stopped container |
| `docker rm -f <name>` | Force remove (even running) |
| `docker rm -f $(docker ps -aq)` | Remove ALL containers |

---

## 🖼️ Image Commands

| Command | Description |
|---------|-------------|
| `docker pull nginx:1.25` | Pull image from registry |
| `docker images` | List local images |
| `docker build -t myapp:v1 .` | Build image from Dockerfile |
| `docker build -f Dockerfile.prod -t myapp .` | Build with specific Dockerfile |
| `docker tag myapp:v1 registry/myapp:v1` | Tag image for registry |
| `docker push registry/myapp:v1` | Push image to registry |
| `docker rmi nginx:latest` | Remove image |
| `docker image prune` | Remove dangling images |
| `docker image prune -a` | Remove ALL unused images |
| `docker image history nginx` | See image layer history |
| `docker image inspect nginx` | Detailed image metadata |
| `docker save nginx -o nginx.tar` | Save image to tar file |
| `docker load -i nginx.tar` | Load image from tar file |

---

## 🔍 Debugging & Inspection

| Command | Description |
|---------|-------------|
| `docker logs <name>` | View container logs |
| `docker logs -f <name>` | Follow logs (real-time) |
| `docker logs --tail 100 <name>` | Last 100 log lines |
| `docker logs --since 30m <name>` | Logs from last 30 minutes |
| `docker exec -it <name> bash` | Open shell in running container |
| `docker exec -it <name> sh` | Open shell (Alpine containers) |
| `docker exec <name> cat /etc/os-release` | Run single command |
| `docker inspect <name>` | Full container/image details (JSON) |
| `docker inspect --format '{{.NetworkSettings.IPAddress}}' <name>` | Get specific field |
| `docker stats` | Real-time resource usage |
| `docker top <name>` | Processes running in container |
| `docker diff <name>` | Filesystem changes vs image |
| `docker cp <name>:/path ./local` | Copy file from container |
| `docker cp ./local <name>:/path` | Copy file to container |

---

## 🌐 Network Commands

| Command | Description |
|---------|-------------|
| `docker network ls` | List networks |
| `docker network create my-net` | Create user-defined bridge |
| `docker network create --subnet 10.0.0.0/16 my-net` | Create with custom subnet |
| `docker network connect my-net <container>` | Add container to network |
| `docker network disconnect my-net <container>` | Remove from network |
| `docker network inspect my-net` | Network details |
| `docker network rm my-net` | Remove network |
| `docker network prune` | Remove unused networks |
| `docker run --network my-net myapp` | Run on specific network |
| `docker run --network host myapp` | Run with host networking |

---

## 💾 Volume Commands

| Command | Description |
|---------|-------------|
| `docker volume create my-vol` | Create named volume |
| `docker volume ls` | List volumes |
| `docker volume inspect my-vol` | Volume details |
| `docker volume rm my-vol` | Remove volume |
| `docker volume prune` | Remove unused volumes |
| `docker run -v my-vol:/data myapp` | Named volume mount |
| `docker run -v $(pwd)/src:/app/src myapp` | Bind mount |
| `docker run -v my-vol:/data:ro myapp` | Read-only mount |
| `docker run --tmpfs /tmp myapp` | tmpfs mount (RAM) |

---

## 🎼 Docker Compose Commands

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start all services (background) |
| `docker compose up -d --build` | Start with rebuild |
| `docker compose down` | Stop and remove all |
| `docker compose down -v` | Stop + remove volumes |
| `docker compose ps` | List running services |
| `docker compose logs -f` | Follow all logs |
| `docker compose logs -f web` | Follow specific service |
| `docker compose exec web bash` | Shell into service |
| `docker compose run --rm web npm test` | One-off command |
| `docker compose restart web` | Restart specific service |
| `docker compose up -d --scale web=3` | Scale service |
| `docker compose -f compose.yml -f compose.dev.yml up -d` | Multiple files |
| `docker compose --profile debug up -d` | Start with profile |
| `docker compose config` | Validate and view merged config |

---

## 🧹 Cleanup Commands

| Command | Description |
|---------|-------------|
| `docker system df` | Show disk usage breakdown |
| `docker system prune` | Remove stopped containers + dangling images + unused networks |
| `docker system prune -a` | ^ + ALL unused images |
| `docker system prune --volumes` | ^ + unused volumes |
| `docker container prune` | Remove stopped containers only |
| `docker image prune -a` | Remove unused images only |
| `docker volume prune` | Remove unused volumes only |
| `docker network prune` | Remove unused networks only |

---

## 📝 Dockerfile Quick Reference

```dockerfile
FROM image:tag              # Base image
WORKDIR /app                # Set working directory
COPY src dest               # Copy files (preferred)
ADD src dest                # Copy + auto-extract archives
RUN command                 # Execute during build
ENV KEY=value               # Environment variable (build + runtime)
ARG KEY=value               # Build-time only variable
EXPOSE port                 # Document port (doesn't publish)
USER username               # Switch to non-root user
HEALTHCHECK CMD command     # Container health check
ENTRYPOINT ["executable"]   # Main command (hard to override)
CMD ["args"]                # Default args (easy to override)
VOLUME ["/data"]            # Declare mount point
LABEL key=value             # Image metadata
```

### Dockerfile Best Practices Summary

```
1. Use specific tags      → FROM python:3.11-slim (not :latest)
2. Multi-stage builds     → Separate build and runtime
3. Layer caching          → COPY deps before code
4. Combine RUN            → RUN cmd1 && cmd2 && cleanup
5. Non-root user          → USER appuser
6. .dockerignore          → Exclude .git, node_modules, .env
7. HEALTHCHECK            → Define container health
8. Exec form              → CMD ["python", "app.py"]
```

---

## 🔑 Common Port Mappings

| Service | Container Port | Typical Host Port |
|---------|---------------|-------------------|
| Nginx | 80 | 80, 8080 |
| Nginx SSL | 443 | 443 |
| Node.js | 3000 | 3000 |
| Python Flask | 5000 | 5000 |
| Django | 8000 | 8000 |
| Spring Boot | 8080 | 8080 |
| PostgreSQL | 5432 | 5432 |
| MySQL | 3306 | 3306 |
| MongoDB | 27017 | 27017 |
| Redis | 6379 | 6379 |
| Elasticsearch | 9200 | 9200 |
| RabbitMQ | 5672/15672 | 5672/15672 |

---

## 🚨 Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `permission denied` on docker | User not in docker group | `sudo usermod -aG docker $USER` then logout/login |
| `port already in use` | Another process on that port | `docker ps` check, or use different host port |
| `image not found` | Wrong image name or registry | Check spelling, try `docker pull` first |
| `OOMKilled` | Container exceeded memory limit | Increase `--memory` or optimize app |
| `exec format error` | Wrong platform (ARM vs x86) | Use `--platform linux/amd64` in build |
| `COPY: file not found` | File not in build context | Check `.dockerignore`, check relative paths |
| `container exited (1)` | App crashed | Check `docker logs <name>` |
| `network not found` | Network doesn't exist | `docker network create <name>` first |
| `volume in use` | Container using the volume | Stop container first, then remove volume |

---

## 🧠 All Memory Shortcuts Summary

| Chapter | Mnemonic | Meaning |
|---------|----------|---------|
| Architecture | **CDRC** | Client, Daemon, Runtime, containerd |
| Install | **RAPI** | Remove, Add repo, Package install, Init |
| Images | **PLIRT** | Pull, List, Inspect, Remove, Tag |
| Containers | **CRSD** | Create, Run, Stop, Delete |
| Debugging | **LET** | Logs, Exec, Top/Stats |
| Run flags | **DPEN** | Detached, Port, Env, Name |
| Dockerfile | **FRoCoCE** | FROM, RUN, COPY, CMD, EXPOSE |
| Optimization | **SLIM** | Small base, Layer cache, Ignore, Multi-stage |
| Network | **BHO-N** | Bridge, Host, Overlay, None |
| Storage | **VBT** | Volumes, Bind mounts, tmpfs |
| Compose | **UDLS** | Up, Down, Logs, Scale |
| Security | **HIBRN** | Host, Image, Build, Runtime, Network |
| Production | **LMRSH** | Logging, Monitoring, Resources, Size, Health |

---

**← Previous: [13 - Hands-On Projects](./13-hands-on-projects.md)** | **Back to: [00 - Index](./00-docker-index.md)** 🏠
