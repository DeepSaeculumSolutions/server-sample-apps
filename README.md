# Server Sample Apps

Full-stack sample application with **Node.js API**, **React Frontend**, and backing services: **MongoDB**, **Redis**, **RabbitMQ**.

Each service is deployed separately using its own **Dockerfile**.

## Architecture

```
┌──────────────┐     ┌──────────────┐
│  React App   │────▶│  Node.js API │
│  (Frontend)  │     │  (Backend)   │
│  port: 8080  │     │  port: 3000  │
└──────────────┘     └──────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
       ┌───────────┐ ┌───────────┐ ┌───────────┐
       │  MongoDB  │ │   Redis   │ │ RabbitMQ  │
       │  :27017   │ │   :6379   │ │   :5672   │
       └───────────┘ └───────────┘ │  UI:15672 │
                                   └───────────┘
```

## Services Overview

| Service    | Purpose                              | Port(s)         |
|------------|--------------------------------------|-----------------|
| MongoDB    | Database - stores users              | 27017           |
| Redis      | Cache - counter & session store      | 6379            |
| RabbitMQ   | Message broker - event queue         | 5672, 15672(UI) |
| Node API   | Backend REST API                     | 3000            |
| React App  | Frontend dashboard                   | 8080            |

---

## Setup

### Step 1: Deploy Backing Services (Docker)

Run MongoDB, Redis, and RabbitMQ as separate containers:

```bash
# MongoDB
docker run -d --name sample-mongo \
  -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=mongopassword \
  -e MONGO_INITDB_DATABASE=sample_app \
  -v mongo_data:/data/db \
  mongo:7

# Redis
docker run -d --name sample-redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:7-alpine redis-server --requirepass redispassword --appendonly yes

# RabbitMQ (with Management UI)
docker run -d --name sample-rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=rabbitmqpassword \
  -v rabbitmq_data:/var/lib/rabbitmq \
  rabbitmq:3-management-alpine
```

### Step 2: Create Dockerfiles (Your Task!)

**`sample-node-api/Dockerfile`** — Node.js API:
- Base: `node:18-alpine`
- WORKDIR: `/app`
- Copy `package*.json`, run `npm install`
- Copy source, expose `3000`, CMD `node server.js`

**`sample-react-app/Dockerfile`** — Multi-stage build:
- Stage 1: `node:18-alpine` — install deps, build (`npm run build`)
- Stage 2: `nginx:alpine` — copy `/dist` to nginx, expose `80`

### Step 3: Set Up Environment Variables

```bash
cp sample-node-api/.env.example sample-node-api/.env
cp sample-react-app/.env.example sample-react-app/.env
```

Open each `.env` file and fill in your credentials. Comments inside each file explain every variable.

### Step 4: Build & Run Apps

```bash
# Build and run Node API
cd sample-node-api
docker build -t sample-api .
docker run -d --name sample-api -p 3000:3000 --env-file .env sample-api

# Build and run React Frontend
cd ../sample-react-app
docker build -t sample-frontend .
docker run -d --name sample-frontend -p 8080:80 sample-frontend
```

### Step 5: Verify

| Check                  | URL / Command                              |
|------------------------|--------------------------------------------|
| API Health             | http://localhost:3000/health                |
| API Root               | http://localhost:3000/                      |
| Frontend               | http://localhost:8080                       |
| RabbitMQ Management UI | http://localhost:15672                      |
| MongoDB (mongosh)      | `mongosh "mongodb://root:mongopassword@localhost:27017"` |
| Redis (redis-cli)      | `redis-cli -a redispassword`               |

---

## Environment Variables Reference

### Node API (`sample-node-api/.env`)

**Server:**

| Variable    | Example       | Description                    |
|-------------|---------------|--------------------------------|
| `PORT`      | `3000`        | API server port                |
| `NODE_ENV`  | `development` | development / staging / production |

**MongoDB:**

| Variable    | Example                                                                  | Description                                                        |
|-------------|--------------------------------------------------------------------------|--------------------------------------------------------------------|
| `MONGO_URL` | `mongodb://root:mongopassword@localhost:27017/sample_app?authSource=admin` | Full connection URL. Username/password must match the mongo container. |

**Redis:**

| Variable         | Example         | Description                                      |
|------------------|-----------------|--------------------------------------------------|
| `REDIS_URL`      | _(empty)_       | Optional full URL override, leave empty to use individual vars |
| `REDIS_HOST`     | `localhost`     | Redis host (`localhost` or container IP)          |
| `REDIS_PORT`     | `6379`          | Redis port                                       |
| `REDIS_USERNAME` | _(empty)_       | Only if Redis ACL is configured                  |
| `REDIS_PASSWORD` | `redispassword` | Must match `--requirepass` in redis container     |

**Message Queue (RabbitMQ):**

| Variable        | Example             | Description                                     |
|-----------------|---------------------|-------------------------------------------------|
| `MSMQ_ENABLE`   | `true`              | `true` to connect, `false` to skip RabbitMQ     |
| `MSMQ_PROTOCOL` | `amqp`              | `amqp` or `amqps` for TLS                       |
| `MSMQ_HOST`     | `localhost`          | RabbitMQ host (`localhost` or container IP)      |
| `MSMQ_PORT`     | `5672`              | AMQP port                                       |
| `MSMQ_USERNAME` | `admin`             | Must match `RABBITMQ_DEFAULT_USER`               |
| `MSMQ_PASSWORD` | `rabbitmqpassword`  | Must match `RABBITMQ_DEFAULT_PASS`               |
| `MSMQ_QUEUE`    | `task_queue`        | Queue name for publishing/consuming              |

### React App (`sample-react-app/.env`)

| Variable       | Example                | Description                        |
|----------------|------------------------|------------------------------------|
| `VITE_API_URL` | `http://localhost:3000` | Node API URL (must be reachable from browser) |

---

## API Endpoints

| Method | Endpoint             | Description                  | Service Used       |
|--------|----------------------|------------------------------|--------------------|
| GET    | `/`                  | Welcome + endpoint list      | -                  |
| GET    | `/health`            | Health check (all services)  | All                |
| GET    | `/info`              | Server info + service status | All                |
| GET    | `/users`             | List all users               | MongoDB            |
| POST   | `/users`             | Create a user                | MongoDB + RabbitMQ |
| GET    | `/users/:id`         | Get single user              | MongoDB            |
| GET    | `/counter`           | Get counter value            | Redis              |
| POST   | `/counter/increment` | Increment counter            | Redis              |
| POST   | `/queue/publish`     | Publish message to queue     | RabbitMQ           |
| GET    | `/queue/status`      | Queue info (message count)   | RabbitMQ           |
| GET    | `/logs`              | View recent app logs         | Filesystem         |

---

## Useful Commands

```bash
# Check running containers
docker ps

# View logs for a specific container
docker logs -f sample-api
docker logs -f sample-mongo
docker logs -f sample-redis
docker logs -f sample-rabbitmq

# Stop and remove a container
docker stop sample-api && docker rm sample-api

# Remove all sample containers
docker stop sample-api sample-frontend sample-mongo sample-redis sample-rabbitmq
docker rm sample-api sample-frontend sample-mongo sample-redis sample-rabbitmq

# Rebuild after code changes
docker build -t sample-api ./sample-node-api
docker build -t sample-frontend ./sample-react-app
```
