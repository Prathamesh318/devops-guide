# 🛠️ Chapter 13: Hands-On Projects

> **"You don't learn Docker by reading — you learn by building. Here are progressively complex projects to master everything."**

---

## 🟢 Project 1: Static Website with Nginx (Beginner)

### Goal
Containerize a simple HTML/CSS website and serve it with Nginx.

### Project Structure
```
project-1-static-site/
├── Dockerfile
├── .dockerignore
├── index.html
├── style.css
└── assets/
    └── logo.png
```

### index.html
```html
<!DOCTYPE html>
<html>
<head>
    <title>My Dockerized Site</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <h1>🐳 Hello from Docker!</h1>
    <p>This site is running inside a container.</p>
</body>
</html>
```

### Dockerfile
```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Commands
```bash
# Build
docker build -t my-static-site .

# Run
docker run -d -p 8080:80 --name website my-static-site

# Visit: http://localhost:8080

# Cleanup
docker rm -f website
```

### What You Learn
- Basic Dockerfile
- COPY instruction
- Port mapping
- Running containers in background

---

## 🟡 Project 2: Python Flask API with Database (Intermediate)

### Goal
Build a REST API with Flask, PostgreSQL, and Docker Compose.

### Project Structure
```
project-2-flask-api/
├── compose.yml
├── app/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app.py
│   └── models.py
└── init.sql
```

### app/requirements.txt
```
flask==3.0.0
flask-sqlalchemy==3.1.1
psycopg2-binary==2.9.9
gunicorn==21.2.0
```

### app/app.py
```python
import os
from flask import Flask, jsonify, request
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get(
    'DATABASE_URL', 'postgresql://admin:secret@db:5432/tododb'
)
db = SQLAlchemy(app)

class Todo(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    done = db.Column(db.Boolean, default=False)

with app.app_context():
    db.create_all()

@app.route('/health')
def health():
    return jsonify(status='healthy')

@app.route('/todos', methods=['GET'])
def get_todos():
    todos = Todo.query.all()
    return jsonify([{'id': t.id, 'title': t.title, 'done': t.done} for t in todos])

@app.route('/todos', methods=['POST'])
def create_todo():
    data = request.json
    todo = Todo(title=data['title'])
    db.session.add(todo)
    db.session.commit()
    return jsonify({'id': todo.id, 'title': todo.title, 'done': todo.done}), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### app/Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser
EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=5s \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

### compose.yml
```yaml
services:
  api:
    build: ./app
    ports:
      - "5000:5000"
    environment:
      - DATABASE_URL=postgresql://admin:secret@db:5432/tododb
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: tododb
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d tododb"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

volumes:
  pg-data:

networks:
  app-network:
```

### Commands
```bash
# Start everything
docker compose up -d --build

# Test API
curl http://localhost:5000/health
curl -X POST http://localhost:5000/todos \
    -H "Content-Type: application/json" \
    -d '{"title": "Learn Docker"}'
curl http://localhost:5000/todos

# Cleanup
docker compose down -v
```

### What You Learn
- Docker Compose multi-container setup
- Healthchecks and depends_on
- Named volumes for data persistence
- Environment variables
- Custom network for DNS resolution

---

## 🔴 Project 3: Full-Stack App with Nginx Reverse Proxy (Advanced)

### Goal
Build a complete full-stack application: React frontend, Node.js API, PostgreSQL, Redis, and Nginx reverse proxy.

### Project Structure
```
project-3-fullstack/
├── compose.yml
├── compose.dev.yml
├── nginx/
│   └── nginx.conf
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── index.js
└── .env
```

### backend/src/index.js
```javascript
const express = require('express');
const { Pool } = require('pg');
const Redis = require('ioredis');

const app = express();
app.use(express.json());

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const redis = new Redis(process.env.REDIS_URL);

app.get('/api/health', (req, res) => {
    res.json({ status: 'healthy', timestamp: new Date() });
});

app.get('/api/items', async (req, res) => {
    const cached = await redis.get('items');
    if (cached) return res.json(JSON.parse(cached));
    
    const { rows } = await pool.query('SELECT * FROM items ORDER BY id DESC');
    await redis.set('items', JSON.stringify(rows), 'EX', 60);
    res.json(rows);
});

app.listen(5000, () => console.log('API running on port 5000'));
```

### backend/Dockerfile
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

FROM node:20-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder --chown=app:app /app .
USER app
EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=5s \
    CMD wget -qO- http://localhost:5000/api/health || exit 1
CMD ["node", "src/index.js"]
```

### nginx/nginx.conf
```nginx
upstream frontend {
    server frontend:3000;
}

upstream api {
    server api:5000;
}

server {
    listen 80;

    location / {
        proxy_pass http://frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### compose.yml
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - frontend
      - api
    networks:
      - frontend-net

  frontend:
    build: ./frontend
    networks:
      - frontend-net

  api:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://admin:${DB_PASSWORD}@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - frontend-net
      - backend-net

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - backend-net

  redis:
    image: redis:alpine
    command: redis-server --save 60 1
    volumes:
      - redis-data:/data
    networks:
      - backend-net

volumes:
  pg-data:
  redis-data:

networks:
  frontend-net:
  backend-net:
```

### .env
```
DB_PASSWORD=supersecret123
```

### Commands
```bash
# Start full stack
docker compose up -d --build

# Visit http://localhost (Nginx routes to frontend)
# API at http://localhost/api/health

# View logs
docker compose logs -f api

# Scale API horizontally
docker compose up -d --scale api=3

# Cleanup
docker compose down -v
```

### What You Learn
- Multi-stage builds
- Nginx reverse proxy
- Multi-network isolation (frontend/backend)
- Redis caching layer
- Environment variables from .env files
- Service scaling
- Complete production-like architecture

---

## 🎯 Project Challenge Progression

```
┌──────────────────────────────────────────────────────────────────┐
│  YOUR DOCKER LEARNING PATH                                       │
│                                                                   │
│  Level 1: Static site with Nginx                                │
│  ✅ Dockerfile, build, run, port mapping                        │
│                                                                   │
│  Level 2: Python API + Database                                 │
│  ✅ Compose, volumes, healthchecks, env vars                    │
│                                                                   │
│  Level 3: Full-stack + Reverse Proxy                            │
│  ✅ Multi-stage, multi-network, scaling, caching                │
│                                                                   │
│  Level 4 (Challenge): Add CI/CD pipeline                        │
│  ☐ GitHub Actions builds and pushes to GHCR                    │
│  ☐ Automated vulnerability scanning                             │
│  ☐ Deploy to a cloud VM                                        │
│                                                                   │
│  Level 5 (Challenge): Add monitoring stack                      │
│  ☐ Prometheus + Grafana + cAdvisor                             │
│  ☐ Alert on high CPU/memory                                    │
│  ☐ Log aggregation with Loki                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

**← Previous: [12 - Production Optimization](./12-production-optimization.md)** | **Next: [14 - Cheatsheet](./14-cheatsheet.md)** ➡️
