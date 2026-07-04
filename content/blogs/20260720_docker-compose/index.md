---
title: "Docker Compose: A Complete Guide"
date: 2026-07-20
description: "A complete guide to Docker Compose: file syntax, networking, healthchecks, scaling, security hardening, CI/CD integration, and advanced patterns."
image: "/blogs/20260720_docker-compose/images/docker_compose.png"
tags:
  - docker
  - docker-compose
  - devops
---
## Introduction

Docker Compose is the standard tool for defining and running multi-container applications on a single host. Whether you are building a web application backed by a database, an event-driven system with message queues, or a microservices architecture that you want to run locally, Compose lets you describe the entire stack in one YAML file and bring it up with a single command. This guide covers everything you need to go from zero to a production-ready Compose setup: file syntax, networking, healthchecks, scaling, security hardening, CI/CD integration, and advanced patterns like profiles, configuration reuse, and multi-stage builds.

This guide is written for developers and DevOps engineers who have a basic understanding of Docker (images, containers, and the `docker run` command) and want to manage multi-container workloads without hand-rolling shell scripts or memorizing long command-line invocations. You do not need prior experience with Docker Compose, though experienced users will find the later sections on profiles, include/extends, and multi-stage dev workflows valuable.

Every section includes inline code examples you can run immediately. The full, working examples for each section are available in the companion repository at [github.com/example/docker-compose-guide](https://github.com/example/docker-compose-guide) under the `examples/` directory. Clone the repo, `cd` into any example folder, and run `docker compose up` to see it in action.

## 1. What is Docker Compose and Why Use It?

You have three containers to run. So you open a terminal and start typing.

```bash
docker network create my-app
docker run -d --name redis --network my-app redis:alpine
docker run -d --name web --network my-app -p 8080:80 nginx:alpine
docker run -d --name db --network my-app -e POSTGRES_PASSWORD=secret postgres:16-alpine
```

Four commands just to get a basic stack running. Now add a background worker and a metrics exporter. Suddenly you have a growing shell script that breaks every time someone joins the team and forgets to run the commands in the right order, or forgets to create the network first, or uses the wrong port. And when you want to tear everything down, you need to remember every container name you created.

Docker Compose replaces all of this with a single declarative YAML file. You describe your services, networks, and volumes in one place, and Compose handles the rest. Starting, stopping, networking, naming, and cleanup all become single commands.

### Installing Docker Compose

Docker Compose ships with Docker Desktop on macOS and Windows. If you are using Docker Desktop, you already have it. On Linux, install the Compose plugin:

```bash
# Install the Docker Compose plugin (recommended)
sudo apt install docker-compose-plugin

# Verify the installation
docker compose version
```

The modern invocation is `docker compose` (with a space), which is a CLI plugin for the Docker command. The older standalone `docker-compose` (with a hyphen) is a separate Python binary that is no longer actively developed. All examples in this guide use the plugin syntax.

### From `docker run` to `docker-compose.yml`

Here is the simplest possible Compose file. It defines two services: a web server and a cache:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"

  cache:
    image: redis:alpine
```

Save this as `docker-compose.yml` (or `compose.yml` -- both names are recognized) and run:

```bash
docker compose up
```

That single command pulls both images (if they are not already cached), creates a shared network, starts both containers, and streams their logs to your terminal. The web server is accessible at `http://localhost:8080`. The cache is reachable from the `web` container at the hostname `cache` on its default port.

To stop everything and remove the containers:

```bash
docker compose down
```

To run the stack in the background (detached mode):

```bash
docker compose up -d
```

### What Compose gives you for free

The benefits go beyond saving keystrokes. When you run the two-service example above, Compose automatically:

1. **Creates a shared network.** Both services are placed on the same Docker network. The `web` container can reach the `cache` container by its service name -- no `--link` flags, no manual network creation, no hardcoded IP addresses.

2. **Registers DNS entries.** Docker's internal DNS resolves service names to container IP addresses. If you `exec` into the `web` container and run `ping cache`, it resolves. If a container restarts and gets a new IP, the DNS record updates automatically.

3. **Manages the lifecycle.** `docker compose down` removes all containers and the project network in one step. No orphaned containers, no forgotten `docker rm` commands.

4. **Provides consistent naming.** Containers are named using a predictable `<project>-<service>-<index>` pattern. The project name defaults to the directory name, so different projects never collide.

### The Compose file is infrastructure as code

The most important property of the Compose file is that it lives in version control alongside your application code. Your local development environment is no longer tribal knowledge -- it is code. It evolves with the project, gets reviewed in pull requests, and is consistent across every developer's machine.

When a new team member clones the repository, they do not need a setup guide. They run `docker compose up` and have a working environment in seconds.

This declarative model also maps directly to production orchestration tools. Docker Swarm uses the same Compose file format with minor additions. Kubernetes uses a different manifest format, but the mental model is the same: define your desired state, and the platform makes it so.

### Project isolation

Each Compose project is isolated by default. The project name -- which defaults to the name of the directory containing the Compose file -- is used as a prefix for all containers, networks, and volumes. This means you can run multiple Compose projects on the same machine without name collisions. If you have two projects that both define a `db` service, they get separate containers, separate networks, and separate volumes.

You can override the project name with the `-p` flag or the `COMPOSE_PROJECT_NAME` environment variable:

```bash
docker compose -p my-project up
```

This is useful when you want to run multiple instances of the same project simultaneously, for example to test different branches.

### Essential commands

Here are the commands you will use most often:

```bash
docker compose up          # start all services, stream logs
docker compose up -d       # start all services in the background
docker compose down        # stop and remove containers and networks
docker compose ps          # show status of all services
docker compose logs        # stream logs from all services
docker compose logs web    # stream logs from one service
docker compose exec web sh # open a shell inside a running container
docker compose stop        # stop containers without removing them
docker compose start       # start previously stopped containers
```

The full example for this section is at `examples/01-intro/` in the companion repository.

## 2. Docker Compose File Structure and Syntax

A Docker Compose file is a YAML document organized around a handful of top-level keys. Once you understand what each key does and the conventions that govern how values are interpreted, you can read and write Compose files fluently. This section walks through every major key with annotated examples.

### The top-level structure

Every Compose file has up to four top-level keys:

```yaml
services:    # Container definitions (required)
networks:    # Custom network definitions (optional)
volumes:     # Named volume definitions (optional)
configs:     # Configuration file definitions (optional, less commonly used)
```

The `services` key is where nearly all of your configuration lives. The others are supporting declarations that services reference. Here is a complete example that uses all three of the commonly used top-level keys:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - frontend

  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
    networks:
      - frontend

networks:
  frontend:
    driver: bridge

volumes:
  db-data:
```

### `services`

Each key under `services` becomes a container. The key name serves double duty: it is used as the container's hostname on the internal network, and it is part of the container's name. In the example above, the `web` service can reach the database at the hostname `db`.

### `image` vs `build`

The `image` key tells Compose to pull a pre-built image from a registry (Docker Hub by default):

```yaml
services:
  db:
    image: postgres:16-alpine
```

The `build` key tells Compose to build an image from a Dockerfile before starting the container:

```yaml
services:
  app:
    build: .
```

You can combine both. When you specify `build` and `image` together, Compose builds the image and tags it with the name you provide:

```yaml
services:
  app:
    build: .
    image: myregistry/myapp:latest
```

This is useful when you want to push the locally built image to a registry.

### `ports` -- the HOST:CONTAINER convention

Port mappings follow the format `HOST:CONTAINER`:

```yaml
ports:
  - "8080:80"
```

The left number (8080) is the port on your host machine. The right number (80) is the port inside the container. When you visit `localhost:8080` in your browser, Docker forwards the traffic to port 80 inside the container. The process inside the container always listens on the right-side port regardless of what you set on the left.

A common mistake is changing the left number and expecting the container's process to respond on the new port. It will not. The left number only controls where your host machine listens. The right number must match what the process inside the container is actually bound to.

You can also bind to a specific host interface:

```yaml
ports:
  - "127.0.0.1:8080:80"   # only accessible from localhost
  - "0.0.0.0:8080:80"     # accessible from any interface (default)
```

### `volumes`

Compose supports two kinds of volume mounts:

**Named volumes** are managed by Docker and persist across container restarts and removals:

```yaml
volumes:
  - db-data:/var/lib/postgresql/data
```

The named volume `db-data` must also be declared at the top-level `volumes:` key. Named volumes survive `docker compose down` (but not `docker compose down -v`, which explicitly removes them).

**Bind mounts** map a path on your host filesystem directly into the container:

```yaml
volumes:
  - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

The `:ro` suffix makes the mount read-only inside the container. Bind mounts are useful for configuration files and for mounting source code during local development so that changes on the host are reflected inside the container immediately.

### `networks`

By default, Compose creates a single network for the entire project and connects all services to it. Every service can reach every other service by name. When you need more control -- isolating frontend services from backend services, for example -- you define explicit networks:

```yaml
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

services:
  web:
    networks:
      - frontend
  api:
    networks:
      - frontend
      - backend
  db:
    networks:
      - backend
```

In this configuration, `web` can reach `api` but cannot reach `db` directly. The `api` service bridges both networks.

### `environment`

Pass configuration into containers as environment variables. The inline map format is the clearest:

```yaml
environment:
  POSTGRES_USER: demo
  POSTGRES_PASSWORD: demo
  POSTGRES_DB: demo
```

For secrets and sensitive values, use variable interpolation with an `.env` file:

```yaml
environment:
  DATABASE_URL: postgres://${DB_USER}:${DB_PASS}@db:5432/${DB_NAME}
```

Compose automatically loads variables from a file named `.env` in the project directory. You can also use `env_file:` to load from a specific file:

```yaml
env_file: .env.production
```

### `depends_on`

Controls startup order. In its simplest form, it ensures one service's container exists before another starts:

```yaml
depends_on:
  - db
```

Important caveat: this only waits for the container to be created, not for the process inside to be ready. For robust dependency management, pair `depends_on` with a healthcheck (covered in section 4).

### `restart`

Sets the container's restart policy:

```yaml
restart: unless-stopped
```

The available policies are:
- `no` -- never restart (default)
- `always` -- restart unconditionally
- `on-failure` -- restart only if the container exits with a non-zero exit code
- `unless-stopped` -- restart unless the container was explicitly stopped by the user

For most services, `unless-stopped` is the right choice. The container restarts after crashes and host reboots, but stays stopped if you deliberately stop it.

### A note on `version:`

Older Compose files include a `version:` key at the top:

```yaml
version: "3.8"
```

This field is no longer needed. Modern versions of Docker Compose infer the specification version automatically. If you see it in existing files, you can safely remove it. If you leave it, Compose ignores it.

The full example for this section is at `examples/02-syntax/` in the companion repository.

## 3. Running a Multi-Container Application

Docker Compose puts all services on a shared internal network and registers each service name as a DNS hostname. This means your containers can reach each other by name -- no IP addresses, no service discovery configuration, no environment variables with hardcoded hosts. Understanding how this works is fundamental to building multi-container applications with Compose.

### How Docker's internal DNS works

When you define services in a Compose file, Docker creates an internal network for the project and registers each service name as a hostname on that network. If your Compose file has a `backend` service, any other container in the same project can reach it at `http://backend`. Docker's built-in DNS server resolves the service name to the container's IP address automatically. If a container restarts and gets a new IP, the DNS record updates without any intervention.

This means your application code uses service names as hostnames. A Node.js app connecting to a database would use `postgres://db:5432/mydb`. An nginx proxy forwarding to an API would use `proxy_pass http://api:3000`. The service name is the hostname. Docker handles the rest.

### A practical example: nginx proxying to a backend

The following example demonstrates two services: a reverse proxy that handles public traffic and a backend that is only reachable internally. The proxy forwards requests to the backend using its service name as a hostname.

The `docker-compose.yml`:

```yaml
services:
  proxy:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend

  backend:
    image: nginx:alpine
    expose:
      - "8080"
    command: >
      sh -c "echo 'server { listen 8080; location / { return 200 \"Hello from backend\n\";
      add_header Content-Type text/plain; } }' > /etc/nginx/conf.d/default.conf
      && nginx -g 'daemon off;'"
```

The `nginx.conf` for the proxy:

```nginx
events {}

http {
  server {
    listen 80;

    location / {
      proxy_pass http://backend:8080;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }
  }
}
```

The critical line is `proxy_pass http://backend:8080;`. The hostname `backend` is the service name from the Compose file. Docker resolves it at runtime to the container's IP address. No configuration beyond naming the service is required.

Run `docker compose up` and visit `http://localhost:8080`. The request hits the proxy on port 80, the proxy forwards it to the backend on port 8080, and you see the response "Hello from backend". The backend is never directly accessible from your host machine.

### `expose` vs `ports` -- understanding the difference

This distinction causes significant confusion, so it is worth being precise.

**`ports`** maps a container port to your host machine. It makes the service accessible from outside Docker:

```yaml
ports:
  - "8080:80"   # HOST:CONTAINER
```

With this mapping, you can open `http://localhost:8080` in your browser and reach the container's port 80.

**`expose`** does not publish any port to your host. It is purely documentation -- a signal that the container listens on a particular port. It does not create any network mapping, and it does not restrict container-to-container communication:

```yaml
expose:
  - "8080"
```

Here is the key insight: containers on the same Compose network can reach each other on any port regardless of whether `expose` or `ports` is set. If you removed the `expose` block from the backend service entirely, the proxy would still be able to connect to it on port 8080. The `expose` directive is useful for human readers and for tooling that inspects Compose files, but it has no effect on actual network connectivity between containers.

### When to use `ports` vs `expose`

The rule is straightforward:

- **Use `ports`** for services that need to be accessible from your host machine: web servers, API gateways, database admin tools you want to access in your browser.
- **Use `expose` (or nothing)** for internal services: backends, workers, internal APIs that only other containers need to reach.

This gives you a clean way to enforce architectural boundaries in your local development environment. The proxy faces the outside world. The backend stays internal. Nothing unintended leaks to your host network.

### Building on this pattern

This proxy-backend pattern is the foundation for more complex architectures. In a typical web application, you might have:

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api
      - frontend

  frontend:
    build: ./frontend
    expose:
      - "3000"

  api:
    build: ./api
    expose:
      - "8080"
    depends_on:
      - db
      - cache

  db:
    image: postgres:16-alpine
    volumes:
      - db-data:/var/lib/postgresql/data

  cache:
    image: redis:alpine

volumes:
  db-data:
```

Nginx is the only service with a `ports` entry. Everything else is internal. The `api` service connects to `db` and `cache` by name. The nginx config proxies `/api` to `http://api:8080` and `/` to `http://frontend:3000`. Every service uses the others' names as hostnames.

### Debugging connectivity between services

If a service cannot reach another by name, these commands help diagnose the issue:

```bash
# Check that both services are on the same network
docker compose ps

# Exec into the source container and test connectivity
docker compose exec proxy sh
# Inside the container:
ping backend
wget -qO- http://backend:8080

# Inspect the network directly
docker network ls
docker network inspect <project_name>_default
```

The most common cause of failed name resolution is that services are on different networks (when explicit networks are defined) or that the target service has not started yet.

The full example for this section is at `examples/03-multi-container/` in the companion repository.

## 4. Healthchecks and Service Dependencies

Your application starts before the database is ready and crashes. This is one of the most common Docker Compose frustrations, and it has a clean solution built right into Compose. The combination of healthchecks and conditional dependencies eliminates startup race conditions without shell sleep loops or retry scripts.

### The problem with `depends_on` alone

When you write `depends_on: - db`, Compose ensures that the `db` container is created and running before starting the dependent service. But "running" and "ready" are two very different things.

A Postgres container, for example, takes several seconds after starting before it actually accepts connections. During that window, your application can attempt to connect, receive a "connection refused" error, and exit. From Compose's perspective, the `db` container is running -- the fact that Postgres has not finished its initialization sequence is invisible to `depends_on`.

This is not a bug. It is a design choice: Compose manages containers, not the processes inside them. To bridge that gap, you need healthchecks.

### Defining a healthcheck

A healthcheck is a command that Compose runs periodically inside the container to determine whether the service is healthy. Here is a complete example:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U demo -d demo"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
```

Each parameter controls a specific aspect of the health checking process:

- **`test`** -- The command to execute inside the container. An exit code of 0 means the check passed; any other exit code counts as a failure. After `retries` consecutive failures, Docker marks the container `unhealthy`. The `CMD-SHELL` form runs the command through the container's shell, which allows pipes and shell operators. The `CMD` form (e.g., `["CMD", "pg_isready", "-U", "demo"]`) runs the command directly without a shell.

- **`interval`** -- How often to run the healthcheck. Five seconds is a reasonable default for most services. Shorter intervals detect failures faster but add more overhead.

- **`timeout`** -- How long to wait for the healthcheck command to complete before treating it as a failure. If `pg_isready` hangs for more than 5 seconds, something is wrong, and the check should fail.

- **`retries`** -- How many consecutive failures are required before Docker marks the container as unhealthy. With 5 retries at 5-second intervals, a service has 25 seconds of sustained failure before being marked unhealthy.

- **`start_period`** -- A grace period at startup during which healthcheck failures do not count toward the retry limit. This is critical for services with slow initialization. Postgres needs time to initialize its data directory, run recovery, and start accepting connections. Setting `start_period: 10s` means that failures during the first 10 seconds are ignored -- you are not penalizing a service that simply has not finished starting yet.

### Common healthcheck commands for popular services

Different services require different healthcheck commands. Here are the most common ones:

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]

# MySQL
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]

# HTTP services (any service with an HTTP endpoint)
healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]

# MongoDB
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]

# Elasticsearch
healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
```

The key principle is the same in every case: the healthcheck command tests whether the service is actually ready to handle requests, not just whether the process is running.

### Waiting for healthy with `condition: service_healthy`

With a healthcheck defined on the dependency, you can upgrade `depends_on` to wait for the service to be genuinely ready:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U demo -d demo"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  app:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      db:
        condition: service_healthy
```

The `condition: service_healthy` directive tells Compose to wait until the `db` service's healthcheck has passed before starting the `app` service. The application does not start, does not attempt database connections, and does not receive traffic until Postgres has confirmed it is ready.

### The three dependency conditions

Compose supports three conditions for `depends_on`:

| Condition | Behavior |
|---|---|
| `service_started` | Wait for the container to be running (default) |
| `service_healthy` | Wait for the healthcheck to pass |
| `service_completed_successfully` | Wait for the container to exit with code 0 |

The third condition, `service_completed_successfully`, is designed for one-shot setup containers. A common pattern is to run database migrations as a separate service that exits after completion:

```yaml
services:
  migrate:
    build: .
    command: python manage.py migrate
    depends_on:
      db:
        condition: service_healthy

  app:
    build: .
    command: python manage.py runserver
    depends_on:
      db:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
```

The `app` service waits for both the database to be healthy and the migration to complete successfully before starting. This guarantees the database schema is up to date before the application serves any requests.

### Healthchecks beyond local development

Healthchecks defined in Compose files are also respected by Docker Swarm. If you later deploy with Swarm, Docker Swarm also respects healthchecks when determining rollback conditions during rolling updates. Writing correct healthchecks in Compose pays dividends beyond local development.

For Kubernetes, you would configure readiness and liveness probes separately in pod specifications, but the mental model is identical: define what "ready" means, and let the platform enforce it.

The full example for this section is at `examples/04-healthcheck-depends-on/` in the companion repository.

## 5. Using Docker Compose with Dockerfiles

Pre-built images from Docker Hub work perfectly for infrastructure services like databases and caches. But when your application has custom code, specific dependencies, or configuration that needs to be baked into the image, you need to build your own. Docker Compose integrates with Dockerfiles so that building custom images is part of the same workflow as starting your stack.

### `image:` vs `build:` in Compose

These two keys represent fundamentally different approaches:

- **`image: nginx:alpine`** pulls a pre-built image from a registry. No building happens.
- **`build: .`** builds an image from a Dockerfile in the specified directory before starting the container. The `.` is the build context -- the directory whose contents are sent to the Docker daemon for the build.

You can also combine them:

```yaml
services:
  app:
    build: .
    image: myregistry/myapp:latest
```

When both are specified, Compose builds the image and tags it with the name from `image:`. This is useful when you want to push the locally built image to a registry later with `docker push`.

### A minimal example

The simplest case is a custom HTML page served by nginx. Two files are needed: a Compose file and a Dockerfile.

The `docker-compose.yml`:

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
```

The `Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

And the `index.html`:

```html
<!DOCTYPE html>
<html>
  <head><title>Docker Compose + Dockerfile</title></head>
  <body>
    <h1>Built with Docker Compose!</h1>
    <p>This page is served from a custom image built by docker compose.</p>
  </body>
</html>
```

Run it with:

```bash
docker compose up --build
```

Visit `http://localhost:8080` and you see the custom page. That is the full loop: write a Dockerfile, reference it in Compose with `build:`, and run the stack.

### The `--build` flag and when you need it

By default, `docker compose up` reuses a previously built image if one exists. If you change your Dockerfile, your source files, or anything in the build context, you need to tell Compose to rebuild:

```bash
docker compose up --build
```

This triggers a rebuild for every service that has a `build:` key. Docker evaluates each layer in the Dockerfile and serves unchanged layers from cache, so rebuilds are fast when only your application code has changed.

If you want to ignore the cache entirely and rebuild from scratch:

```bash
docker compose build --no-cache
docker compose up
```

A common workflow during development is to always use `--build` so you never accidentally run stale code. The performance impact is negligible because of layer caching.

### Advanced build options

When you need more control than `build: .` provides, use the expanded form:

```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production
        APP_VERSION: "1.2.3"
      target: builder
      cache_from:
        - myregistry/myapp:latest
```

Each option serves a specific purpose:

- **`context`** -- The directory sent to the Docker daemon as the build context. All paths in `COPY` and `ADD` instructions are relative to this directory. In a monorepo, you might set `context: .` at the repository root so that the Dockerfile can copy files from multiple subdirectories.

- **`dockerfile`** -- The name of the Dockerfile to use, if it is not the default `Dockerfile`. This lets you maintain separate `Dockerfile.dev` and `Dockerfile.prod` files and switch between them.

- **`args`** -- Build-time variables injected via `ARG` instructions in the Dockerfile. These are available only during the build, not at runtime. For runtime configuration, use `environment:` in the service definition.

- **`target`** -- The build stage to stop at, when using multi-stage Dockerfiles (covered in detail in section 12).

- **`cache_from`** -- Images to use as cache sources. Useful in CI where the local build cache is empty but a previously pushed image can provide cached layers.

### Build arguments vs environment variables

A common source of confusion is the difference between `args` (build-time) and `environment` (runtime):

```yaml
services:
  app:
    build:
      context: .
      args:
        NODE_ENV: production      # Available during 'docker build' via ARG
    environment:
      DATABASE_URL: postgres://db:5432/mydb  # Available when the container runs
```

In the Dockerfile, build arguments are consumed with `ARG`:

```dockerfile
FROM node:20-alpine
ARG NODE_ENV
RUN echo "Building for $NODE_ENV"
# NODE_ENV is available here during build

# But NOT available at runtime unless you explicitly set it:
ENV NODE_ENV=$NODE_ENV
```

If you need a build argument to also be available at runtime, you must explicitly convert it with `ENV` in the Dockerfile.

### Structuring your Dockerfile for fast rebuilds

The order of instructions in your Dockerfile directly affects rebuild speed. Docker caches each layer, and when a layer changes, all subsequent layers are invalidated. Structure your Dockerfile so that infrequently changing layers come first:

```dockerfile
FROM node:20-alpine
WORKDIR /app

# 1. Copy dependency manifests first (changes rarely)
COPY package.json package-lock.json ./

# 2. Install dependencies (cached unless package.json changes)
RUN npm ci

# 3. Copy application code last (changes frequently)
COPY . .

# 4. Build the application
RUN npm run build

CMD ["node", "dist/server.js"]
```

If you copy the application code before installing dependencies, every code change invalidates the dependency installation layer, and `npm ci` runs on every build. By copying `package.json` first, dependencies are only reinstalled when the manifest changes.

This optimization applies regardless of whether you are using Compose or building images directly. But since Compose development typically involves many rebuild cycles, the time savings compound quickly.

The full example for this section is at `examples/05-dockerfiles/` in the companion repository.

## 6. Managing and Scaling Services

Docker Compose can run multiple replicas of a service, giving you a way to test concurrent workloads, experiment with round-robin load balancing, and understand what "stateless" means for your services -- all on your local machine. While Compose scaling is not a replacement for production orchestrators like Kubernetes or Docker Swarm, it is an excellent tool for development and testing.

### Scaling with `--scale` at runtime

The simplest way to run multiple instances of a service is the `--scale` flag:

```bash
docker compose up --scale worker=5
```

This starts 5 replicas of the `worker` service. Compose names them `worker-1` through `worker-5`, places them all on the same internal network, and Docker's DNS automatically round-robins requests to the `worker` hostname across all replicas.

You do not need to configure a load balancer or change application code. Any service that connects to `http://worker/` has its requests distributed across all running replicas automatically.

### Setting a default replica count with `deploy.replicas`

If you always want more than one replica, set the default in the Compose file:

```yaml
services:
  proxy:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - worker

  worker:
    image: nginx:alpine
    expose:
      - "80"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "0.5"
          memory: 64M
```

Running `docker compose up` starts 3 worker replicas by default. The `--scale` flag overrides this at runtime:

```bash
docker compose up --scale worker=5   # override to 5
docker compose up --scale worker=1   # override to 1
```

### Why scaled services cannot bind fixed host ports

Notice that the `worker` service uses `expose` instead of `ports`. This is not a style choice -- it is a hard constraint.

If you gave `worker` a host port binding like `ports: - "8081:80"`, scaling to 3 replicas would fail immediately. Only one process can bind a given port on your host machine at a time. Three containers cannot all map to host port 8081.

The solution is to not bind a host port on the replicated service. Let Docker handle routing internally via DNS. Place a proxy or gateway in front of the replicated services, and give only the proxy a `ports:` entry:

```yaml
services:
  proxy:
    image: nginx:alpine
    ports:
      - "8080:80"    # Only the proxy binds a host port
    depends_on:
      - worker

  worker:
    image: nginx:alpine
    expose:
      - "80"         # Internal only -- no host port
    deploy:
      replicas: 3
```

This mirrors production architecture: a load balancer or API gateway faces the public network, while backend services stay on a private network with no direct port exposure.

### Testing round-robin DNS

You can verify that Docker distributes requests across replicas by execing into the proxy and making repeated requests:

```bash
# Start the stack with 3 workers
docker compose up -d

# Exec into the proxy
docker compose exec proxy sh

# Inside the proxy, make repeated requests
for i in $(seq 1 10); do curl -s worker; done
```

Each request may go to a different replica. Docker's DNS round-robin distributes the connections across all healthy replicas of the `worker` service.

### Resource limits with `deploy.resources`

The `deploy.resources.limits` block caps what each replica can consume:

```yaml
deploy:
  replicas: 3
  resources:
    limits:
      cpus: "0.5"
      memory: 64M
    reservations:
      cpus: "0.25"
      memory: 32M
```

- **`limits`** set the maximum resources a container can use. If a container exceeds its memory limit, Docker kills it (OOM kill). If it exceeds its CPU limit, Docker throttles it.
- **`reservations`** set the minimum resources Docker guarantees to the container. These are useful for ensuring that critical services always have enough resources, even when the host is under load.

The `cpus` value accepts decimals: `"0.5"` means half a core, `"2.0"` means two full cores. The `memory` value accepts suffixes: `M` for megabytes, `G` for gigabytes.

Without resource limits, a single misbehaving replica can consume all available memory and crash every other container on the host. Setting limits surfaces problems early: if your service fails under its memory limit locally, it will fail in production too, and it is better to know now.

### Monitoring resource usage

Use `docker stats` to see real-time resource consumption:

```bash
docker stats
```

This shows CPU, memory, network I/O, and disk I/O for every running container. Compare actual usage against your limits to determine if they are set appropriately. If a container is consistently at 95% of its memory limit, it is too tight. If it is at 10%, you can reduce the limit and free resources for other services.

### The limits of Compose scaling

Compose scaling is local and manual. It does not:

- Auto-scale based on load or metrics
- Distribute replicas across multiple machines
- Automatically restart failed replicas (though `restart: unless-stopped` handles individual container crashes)
- Perform rolling updates

For these capabilities, you need Docker Swarm (which uses the same Compose file format with minor additions to the `deploy` section) or Kubernetes. But for testing concurrent behavior, verifying that your application handles multiple instances correctly, and understanding the constraints of horizontal scaling, Compose is the right tool.

The full example for this section is at `examples/06-scaling/` in the companion repository.

## 7. Best Practices

A Compose file that starts your application is a good start. A Compose file that handles secrets safely, survives host reboots, constrains resource usage, and minimizes the attack surface is production-ready. This section covers five practices that take a Compose setup from "it runs" to "it is ready."

### Use `env_file` to keep secrets out of version control

Hardcoding credentials in your Compose file means they end up in version control:

```yaml
# Don't do this
environment:
  POSTGRES_PASSWORD: my-secret-password
```

Instead, use `env_file` to load environment variables from a file that is excluded from git:

```yaml
services:
  db:
    image: postgres:16-alpine
    env_file: .env
    volumes:
      - db-data:/var/lib/postgresql/data
```

The `.env` file:

```
POSTGRES_USER=demo
POSTGRES_PASSWORD=demo
POSTGRES_DB=demo
```

Add `.env` to your `.gitignore`. Provide a `.env.example` with placeholder values so that new team members know which variables to set. This pattern keeps your Compose file safely committable while ensuring credentials never enter version control.

For more complex setups, you can use multiple env files:

```yaml
env_file:
  - .env
  - .env.local
```

Earlier files take precedence — if the same variable appears in multiple files, the first value wins. This gives you a layered configuration approach where your base `.env` sets defaults that `.env.local` can selectively override by introducing new variables, but cannot clobber existing ones.

### Choose named volumes over bind mounts for persistent data

Compose supports two volume types, and the choice matters:

**Named volumes** are managed by Docker. They are portable, automatically created, and their lifecycle is tied to the Docker volume system rather than to a specific path on your host:

```yaml
volumes:
  - db-data:/var/lib/postgresql/data
```

**Bind mounts** map a host directory directly into the container:

```yaml
volumes:
  - ./data:/var/lib/postgresql/data
```

For persistent application data (databases, search indices, message queues), use named volumes. They are more portable across machines, do not depend on the host's directory structure, and avoid permission issues that arise when the container's UID does not match the host's directory ownership.

Use bind mounts for development workflows where you need changes on the host to be reflected inside the container immediately (source code, configuration files). For everything else, named volumes are the safer choice.

### Set a restart policy

Without a restart policy, a crashed container stays stopped until you manually restart it. For any service that should be continuously available, set a restart policy:

```yaml
restart: unless-stopped
```

The four available policies and their use cases:

| Policy | When to use |
|---|---|
| `no` | One-shot tasks, migration scripts |
| `on-failure` | Services where a clean exit (code 0) means "done" |
| `always` | Services that must always be running |
| `unless-stopped` | Most long-running services (sensible default) |

`unless-stopped` is the right choice for most services. The container restarts automatically after crashes and host reboots, but stays stopped if you explicitly stopped it with `docker compose stop`. This gives you automatic recovery without fighting you when you intentionally take a service down.

### Set resource limits

Without resource limits, a container can consume unlimited CPU and memory. A memory leak in one service can bring down every other container on the host. Set explicit limits:

```yaml
services:
  db:
    image: postgres:16-alpine
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 256M

  web:
    image: nginx:alpine
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 64M
```

Start with generous limits, then use `docker stats` to observe actual usage and tighten them. The goal is to prevent runaway containers from affecting the rest of the system while giving each service enough headroom for normal operation.

A container that exceeds its memory limit is killed by the OOM (Out of Memory) killer. If this happens consistently, increase the limit or investigate the memory leak. A container that exceeds its CPU limit is throttled, not killed -- it runs slower but stays alive.

### Use `read_only: true` with targeted `tmpfs` mounts

Locking the container's root filesystem prevents compromised processes from writing malicious files to disk:

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    read_only: true
    tmpfs:
      - /var/cache/nginx
      - /var/run
      - /tmp
```

With `read_only: true`, any attempt to write to the container's filesystem fails. But most applications need to write to certain paths -- cache directories, PID files, temporary files. The `tmpfs` directive mounts these specific paths as in-memory filesystems. They are fast, writable, and their contents disappear when the container restarts.

This is a defense-in-depth measure. Even if an attacker exploits a vulnerability in your application, they cannot write persistent files to the container's filesystem. Combined with running as a non-root user (via `user:` in the service definition or `USER` in the Dockerfile), `read_only` significantly reduces the container's attack surface.

### Putting it all together

Here is a complete example applying all five practices:

```yaml
services:
  db:
    image: postgres:16-alpine
    env_file: .env
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 256M

  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /var/cache/nginx
      - /var/run
      - /tmp
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 64M

volumes:
  db-data:
```

Every secret is externalized. Every service restarts on failure. Every container has resource limits. The web server's filesystem is locked down. The database uses a named volume for persistence. This is a solid foundation for any Compose-based deployment.

The full example for this section is at `examples/07-best-practices/` in the companion repository.

## 8. Troubleshooting Common Issues

Debugging a broken Docker Compose setup is a skill in itself. Most problems fall into a handful of patterns, and once you can recognize them, you can go from "why won't this start" to "fixed" in minutes. This section is a field guide to the most common issues and the commands that actually help.

### The debugging commands you need

Get these into muscle memory:

```bash
docker compose logs              # stream logs from all services
docker compose logs app          # logs from one service
docker compose logs --tail 50    # last 50 lines
docker compose logs -f           # follow (stream in real time)
docker compose exec app sh       # shell inside a running container
docker compose ps                # show status of all services
docker compose config            # print the fully resolved configuration
docker compose top               # show running processes in containers
docker compose down -v           # tear everything down, including volumes
```

**`docker compose logs`** is your first stop for any issue. Container exit codes, error messages, and stack traces all appear here. Use `--tail` to limit output and `-f` to follow in real time.

**`docker compose exec`** gets you inside a running container when logs are not enough. From inside, you can test network connectivity, inspect files, check environment variables, and run diagnostic commands.

**`docker compose config`** is underused but powerful. It resolves all variable substitutions, merges override files, and prints the fully resolved configuration. If a variable is not being substituted correctly, or an override file is not being picked up, `docker compose config` shows you exactly what Compose sees.

**`docker compose ps`** shows the state of every service -- running, exited, or restarting. The exit code column is particularly useful: a service that exits with code 1 usually has an application error; code 137 means it was OOM-killed; code 0 means it exited cleanly (which might be unexpected for a long-running service).

### Startup race conditions

The most common issue: your application starts before a dependency is ready, fails to connect, and exits.

**Symptom:** The `app` service exits immediately after startup with connection errors in the logs. Restarting it manually (or waiting for the restart policy to kick in) often works because the database has had time to initialize.

**Root cause:** `depends_on` without a health condition only waits for the container to exist, not for the process inside to be ready.

**Fix:** Add a healthcheck to the dependency and use `condition: service_healthy`:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U demo"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    image: nginx:alpine
    depends_on:
      db:
        condition: service_healthy
```

### Port conflicts

**Symptom:** `Ports are not available: listen tcp 0.0.0.0:5432: bind: address already in use`

**Root cause:** Another process on your host is already using that port. Common culprits include a locally installed Postgres, another Compose project, or a previously stopped container that did not release the port.

**Diagnosis:**

```bash
# Find what is using the port (macOS/Linux)
lsof -i :5432

# On Linux, alternatively:
ss -tlnp | grep 5432
```

**Fix:** Either stop the conflicting process, or change the host-side port in your Compose file:

```yaml
ports:
  - "5433:5432"    # map to host port 5433 instead
```

Remember: changing the left number does not affect the container. Postgres still listens on 5432 inside the container. Your client connection string just needs to use port 5433.

### Volume permission errors

**Symptom:** A service fails to start with "Permission denied" errors when trying to read or write files in a mounted volume.

**Root cause:** The process inside the container runs as a specific user (often non-root), but the mounted directory or volume has different ownership. UID mismatches between the host and container are the usual culprit.

**Diagnosis:**

```bash
# Check what user the process runs as inside the container
docker compose exec db id

# Check file ownership in the mounted path
docker compose exec db ls -la /var/lib/postgresql/data
```

**Fix options:**

1. **Switch to named volumes** (Docker manages permissions automatically):
   ```yaml
   volumes:
     - db-data:/var/lib/postgresql/data
   ```

2. **Set the user in the service definition** to match the host UID:
   ```yaml
   user: "1000:1000"
   ```

3. **Use an init script** that corrects ownership on startup (some official images support this via environment variables).

### Environment variable issues

**Symptom:** The application behaves unexpectedly or fails to connect to services, even though the Compose file looks correct.

**Root cause:** A typo in a variable name, a missing `.env` file, or a variable that is defined but empty.

**Diagnosis:**

```bash
# See what the container actually received
docker compose exec app env | sort

# See the fully resolved Compose configuration
docker compose config

# Check if the .env file is being loaded
docker compose config | grep -i database
```

Common traps:
- The application reads `DATABASE_URI` but you set `DATABASE_URL`.
- The `.env` file has a syntax error (spaces around `=`, trailing whitespace, or missing quotes for values with special characters).
- A variable is defined in `environment:` but references `${VAR}`, and `VAR` is not set anywhere.

### Network connectivity failures

**Symptom:** One service cannot reach another by name. `curl http://api:3000` fails with "could not resolve host."

**Diagnosis:**

```bash
# Check that both services are running
docker compose ps

# Check that they are on the same network
docker network inspect $(docker compose ps -q | head -1 | xargs docker inspect --format='{{range $k, $v := .NetworkSettings.Networks}}{{$k}}{{end}}')

# Test from inside a container
docker compose exec web ping api
docker compose exec web nslookup api
```

**Common causes:**
- The services are on different explicit networks and cannot see each other.
- The target service has not started yet or has crashed.
- A typo in the service name (service names are case-sensitive).

### The nuclear option: clean slate

When you have changed volume contents, renamed services, updated images, or suspect cached state is causing issues:

```bash
# Remove everything: containers, networks, AND volumes
docker compose down -v

# Rebuild images from scratch (ignore cache)
docker compose build --no-cache

# Start fresh
docker compose up
```

This is the debugging equivalent of "turn it off and back on again." It removes all containers, all named volumes (and their data), and forces a complete image rebuild. It works surprisingly often, especially when the problem is stale data in a volume or a cached image layer that does not reflect recent changes.

**Warning:** `docker compose down -v` deletes all named volume data. If you have important data in a database volume, back it up first with `docker compose exec db pg_dump`.

The full example for this section is at `examples/08-troubleshooting/` in the companion repository.

## 9. Using Profiles

Your Compose file includes services for development tools, database admin UIs, and network debuggers. But you do not want all of them starting every time you run `docker compose up`. Profiles let you define optional services that only start when you explicitly ask for them, keeping your default stack lean while having specialized tools available on demand.

### How profiles work

The rule is simple:

- **Services without a `profiles` key always start.** They are part of the default stack.
- **Services with a `profiles` key only start when you activate that profile.**

```bash
docker compose up                        # default services only
docker compose --profile debug up        # default + debug services
docker compose --profile tools up        # default + tools services
docker compose --profile debug --profile tools up  # all three
```

Here is a Compose file that demonstrates the pattern:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    volumes:
      - db-data:/var/lib/postgresql/data
    # No profile = always started

  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
    # No profile = always started

  debugger:
    profiles: [debug]
    image: nicolaka/netshoot
    command: sleep infinity
    network_mode: service:web
    # Only started when --profile debug is passed

  adminer:
    profiles: [tools]
    image: adminer
    ports:
      - "8081:8080"
    depends_on:
      - db
    # Only started when --profile tools is passed

volumes:
  db-data:
```

The `db` and `web` services have no `profiles` key, so they always start. The `debugger` service only starts when you pass `--profile debug`. The `adminer` service (a web-based database browser) only starts when you pass `--profile tools`. This keeps the default `docker compose up` fast and resource-efficient.

### A service can belong to multiple profiles

A service can be listed under more than one profile:

```yaml
services:
  monitoring:
    profiles: [debug, observability]
    image: prom/prometheus
    ports:
      - "9090:9090"
```

This service starts when either `--profile debug` or `--profile observability` is activated. It does not need both.

### The netshoot pattern for network debugging

The `debugger` service in the example above uses `nicolaka/netshoot`, a container packed with network troubleshooting tools: `curl`, `dig`, `nmap`, `tcpdump`, `traceroute`, `iperf`, and many more. Combined with `network_mode: service:web`, it shares the `web` container's network namespace, letting you diagnose connectivity from exactly the right vantage point.

```bash
# Start the stack with the debug profile
docker compose --profile debug up -d

# Use netshoot to test connectivity from inside the web service's network
docker compose exec debugger curl http://db:5432
docker compose exec debugger dig db
docker compose exec debugger ping db
docker compose exec debugger traceroute db

# Check what ports are actually listening
docker compose exec debugger ss -tlnp
```

When you are done debugging, `docker compose down` stops everything including the debug container. Nothing lingers.

The `sleep infinity` command keeps the container running without doing anything, so you can exec into it whenever you need to run diagnostic commands. Without it, the container would start, have nothing to do, and exit immediately.

### Common use cases for profiles

**Admin and database UIs:**

```yaml
services:
  pgadmin:
    profiles: [tools]
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"

  redis-insight:
    profiles: [tools]
    image: redislabs/redisinsight
    ports:
      - "8001:8001"
```

These are invaluable for developers inspecting data but are wasteful in CI or when running the default stack.

**Test runners:**

```yaml
services:
  test-runner:
    profiles: [test]
    build: .
    command: pytest tests/ -v
    depends_on:
      db:
        condition: service_healthy
```

Run integration tests explicitly with `docker compose --profile test up`, without the test runner starting on every `docker compose up`.

**Documentation and API explorers:**

```yaml
services:
  swagger-ui:
    profiles: [docs]
    image: swaggerapi/swagger-ui
    environment:
      SWAGGER_JSON: /api/openapi.json
    volumes:
      - ./openapi.json:/api/openapi.json:ro
    ports:
      - "8082:8080"
```

Developers who want to browse the API documentation activate the `docs` profile. Everyone else is unaffected.

### Activating profiles via environment variable

Instead of passing `--profile` on every command, you can set the `COMPOSE_PROFILES` environment variable:

```bash
# In your shell profile or .env file
export COMPOSE_PROFILES=debug,tools

# Now 'docker compose up' activates both profiles automatically
docker compose up
```

This is convenient for developers who always want certain tools available. It can also be set in a `.env` file:

```
COMPOSE_PROFILES=debug,tools
```

### Profiles in CI

Profiles are useful in CI pipelines for activating test-specific services:

```bash
# CI pipeline
docker compose --profile test up -d
docker compose run --rm test-runner pytest tests/
docker compose down
```

The CI pipeline explicitly opts into the `test` profile, starting only the services needed for the test run. The default stack (database, application) starts as normal, and the test runner is added on top.

### Stopping profiled services

When you run `docker compose down`, it stops all running containers regardless of profile. But if you want to stop only the services from a specific profile while keeping the rest running:

```bash
# Stop only the debug services
docker compose --profile debug stop

# Or remove them entirely
docker compose --profile debug down
```

The full example for this section is at `examples/09-profiles/` in the companion repository.

## 10. Using Include and Extends

Copy-pasting Compose configuration across projects leads to drift. Four projects that all need the same Postgres service end up with four slightly different versions -- different healthcheck timings, different resource limits, different volume names. When you need to update the Postgres version or tighten a resource limit, you have to remember to update every copy. Compose provides two tools to eliminate this duplication: `include` for file-level reuse and `extends` for service-level inheritance.

### The base file pattern

Start with a shared file that defines your common infrastructure. This file is a building block that other Compose files reference:

```yaml
# base.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  db-data:
```

This file captures your team's standard Postgres configuration: the image version, the restart policy, the volume setup. It is the single source of truth.

### `include`: file-level merge

The `include` directive pulls an entire Compose file into your current project. All services, volumes, and networks from the included file are merged as if they were defined locally:

```yaml
# docker-compose.yml
include:
  - base.yml

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db
```

When you run `docker compose up`, Compose first loads `base.yml`, then merges it with your project's Compose file. The `db` service and `db-data` volume from `base.yml` are available as if you had defined them inline. Your `web` service can reference `db` in `depends_on` without needing to know where it comes from.

This is the right tool when you want to share an entire set of services across projects. Place `base.yml` in a shared location (a central repository, a git submodule, a symlinked path), and every project that `include`s it picks up updates from one place.

You can include multiple files:

```yaml
include:
  - infrastructure/postgres.yml
  - infrastructure/redis.yml
  - infrastructure/monitoring.yml
```

Each file's services, volumes, and networks are merged into the final configuration. Name collisions between included files and your project's own definitions are resolved by the project file taking precedence.

### `extends`: service-level inheritance

While `include` operates at the file level, `extends` operates at the service level. You inherit a single service definition from another file and override specific fields:

```yaml
# docker-compose.yml
services:
  db-replica:
    extends:
      file: base.yml
      service: db
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo_replica
    volumes:
      - db-replica-data:/var/lib/postgresql/data

volumes:
  db-replica-data:
```

The `db-replica` service inherits everything from the `db` service in `base.yml` -- the image, the restart policy, the base environment variables -- and overrides only what needs to differ. In this case, it uses a different database name and a separate volume, simulating a read replica.

Fields you specify in the extending service replace the inherited values. Fields you omit are inherited as-is. This lets you define a service once and create variations without duplicating the entire definition.

### Combining `include` and `extends`

You can use both in the same file. `include` brings in the shared infrastructure, and `extends` creates variations of individual services:

```yaml
include:
  - base.yml

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - db

  db-replica:
    extends:
      file: base.yml
      service: db
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo_replica
    volumes:
      - db-replica-data:/var/lib/postgresql/data

volumes:
  db-replica-data:
```

The `include` gives you the standard `db` service and its volume. The `extends` creates a `db-replica` variation. The `web` service depends on the included `db`. Everything composes cleanly.

### `include` vs `extends`: choosing the right tool

| Scenario | Tool |
|---|---|
| Share an entire infrastructure stack (DB + cache + queue) across projects | `include` |
| Create a variation of one service (test DB, replica, different config) | `extends` |
| Mix shared infra with project-specific services | `include` for infra, `extends` for variations |
| Multiple teams using a standardized base configuration | `include` pointing to a shared repo |

### How `extends` handles different field types

Understanding how field merging works is important for avoiding surprises:

- **Scalar values** (image, restart, command) are replaced entirely by the extending service.
- **Mapping values** (environment, labels) are merged: the extending service's keys override inherited keys, but inherited keys not mentioned in the extending service are preserved.
- **List values** (volumes, ports, expose) are replaced entirely, not appended. If you extend a service that has `volumes: - db-data:/var/lib/postgresql/data` and you specify your own `volumes:`, the inherited volume list is replaced, not extended.

This means you need to be explicit about lists. If the base service has a volume you want to keep and you are adding a second volume, you must include both in your extending service's `volumes:` list.

### Practical impact at scale

The real payoff comes when you have multiple projects referencing the same base configuration. If your team standardizes on a Postgres setup with specific healthcheck parameters, memory limits, and logging configuration, that standard lives in one file. When the team decides to update to Postgres 17, tighten resource limits, or adjust healthcheck timings, one file changes and every project picks up the update on the next pull.

No more archaeology to figure out which project has the most up-to-date configuration. No more drift between projects that should be identical. The base file is the standard, and `include` or `extends` is how projects adopt it.

### Limitations to be aware of

- `extends` cannot be used with services that define `links`, `volumes_from`, or reference other services by name in `network_mode`. These create hard dependencies that cannot be inherited cleanly.
- `include` files are resolved relative to the file that includes them, not relative to the working directory. Keep this in mind when paths in the included file reference other files.
- Deeply nested inheritance (service A extends B which extends C) is technically possible but quickly becomes difficult to reason about. One level of inheritance is usually sufficient.

The full example for this section is at `examples/10-include-extends/` in the companion repository.

## 11. Integrating with CI/CD Pipelines

Your Compose setup works on your laptop. Making it work reliably on an ephemeral CI runner -- no persistent state, different credentials, potentially different resource constraints -- requires a few adjustments. The good news is that Compose has a clean pattern for this, and it does not require maintaining a separate Compose file from scratch.

### The override file pattern

Compose can layer multiple files on top of each other. Later files override earlier ones, field by field:

```bash
# Local development (uses docker-compose.yml only)
docker compose up

# CI (layers CI overrides on top of the base)
docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d
```

Your base `docker-compose.yml` defines the full stack as it runs in local development. Your `docker-compose.ci.yml` contains only the CI-specific differences -- it is a diff, not a rewrite. Any field you do not mention in the CI file is inherited from the base.

### The base Compose file

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: demo
      POSTGRES_PASSWORD: demo
      POSTGRES_DB: demo
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U demo"]
      interval: 5s
      timeout: 5s
      retries: 5

  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      db:
        condition: service_healthy

volumes:
  db-data:
```

This file works perfectly for local development. The database persists data in a named volume, uses development credentials, and the web service waits for the database to be healthy.

### The CI override file

The CI override targets two concerns: eliminate persistent state and inject CI-specific configuration.

```yaml
# docker-compose.ci.yml
services:
  db:
    volumes:
      - type: tmpfs
        target: /var/lib/postgresql/data
    environment:
      POSTGRES_USER: ci
      POSTGRES_PASSWORD: ci
      POSTGRES_DB: ci_test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ci"]

  web:
    environment:
      APP_ENV: ci

volumes:
  db-data: null
```

Each override serves a specific purpose:

**`tmpfs` volume:** The database switches from a named volume to an in-memory `tmpfs` mount. Data starts empty on every run and disappears when the container stops. No cleanup scripts are needed, and no leftover state from a previous test run can affect your current tests. This is faster than disk-backed storage and perfectly suited for CI, where data persistence is not just unnecessary but actively harmful.

**`volumes: db-data: null`:** This removes the named volume declaration from the base file entirely. Without this, Compose would try to create the `db-data` volume even though no service references it in the CI configuration.

**CI credentials:** The database uses different credentials in CI. This is both a security practice (CI credentials should not match development credentials) and a correctness measure (it verifies that your application reads credentials from configuration, not from hardcoded values).

**`APP_ENV: ci`:** The web service receives an environment variable indicating it is running in CI. Application code can use this to adjust behavior: disable rate limiting, use test-specific configuration, enable verbose logging, or skip optional external service integrations.

### Running tests with `docker compose run`

For executing test suites in CI, `docker compose run` is more appropriate than `docker compose up`:

```bash
# Start the stack in the background
docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d

# Run a one-off test command
docker compose run --rm web sh -c "pytest tests/"

# Tear everything down
docker compose down
```

`docker compose run --rm` starts a one-off container for a single command and removes it when done. The container inherits the network, environment variables, and volume mounts from its service definition, so it can reach all other services by hostname.

The `--rm` flag ensures the container is removed after the command completes. Without it, exited test containers accumulate and consume disk space.

### A complete GitHub Actions example

Here is a GitHub Actions workflow that runs integration tests using Docker Compose:

```yaml
name: Integration Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start services
        run: |
          docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d
          docker compose ps

      - name: Wait for services to be healthy
        run: |
          docker compose -f docker-compose.yml -f docker-compose.ci.yml \
            exec -T db pg_isready -U ci -d ci_test --timeout=30

      - name: Run integration tests
        run: |
          docker compose run --rm web sh -c "pytest tests/ -v --tb=short"

      - name: Collect logs on failure
        if: failure()
        run: docker compose logs

      - name: Tear down
        if: always()
        run: docker compose down -v
```

Key details:

- The `if: failure()` step collects logs only when tests fail, giving you diagnostic output without cluttering successful runs.
- The `if: always()` step ensures cleanup happens even if the test step fails. CI runners have limited disk space, and orphaned containers and volumes can cause subsequent runs to fail.
- The `-T` flag on `docker compose exec` disables pseudo-TTY allocation, which is necessary in non-interactive CI environments.

### GitLab CI example

```yaml
integration-tests:
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker compose -f docker-compose.yml -f docker-compose.ci.yml up -d
    - docker compose run --rm web sh -c "pytest tests/"
  after_script:
    - docker compose down -v
```

### Best practices for Compose in CI

1. **Always use `-d` (detached mode)** when starting the stack. You want the stack running in the background while your test commands execute in the foreground.

2. **Always tear down with `docker compose down -v`** in an `always` or `after_script` block. Even on ephemeral runners, cleaning up prevents issues with shared runner pools or cached Docker state.

3. **Use `--rm` with `docker compose run`** to avoid accumulating exited containers.

4. **Keep the CI override file minimal.** Only override what needs to be different. The more your CI environment resembles your development environment, the more confidence your tests provide.

5. **Use healthchecks.** CI runners vary in performance. A database that initializes in 2 seconds on your laptop might take 10 seconds on a loaded CI runner. Healthchecks with appropriate `start_period` and `retries` values handle this variance gracefully.

The full example for this section is at `examples/11-cicd/` in the companion repository.

## 12. Multi-stage Dockerfiles and Dev Workflows

Shipping debug tools in a production image is a security risk. But stripping them out means your development workflow suffers. Multi-stage Dockerfiles solve this tension: you define multiple build stages in a single Dockerfile, each producing a different image. One stage includes everything you need for debugging, another produces a minimal production binary. Docker Compose wires them together so you pick the right one with a single command.

### Understanding multi-stage builds

A multi-stage Dockerfile has multiple `FROM` instructions. Each one starts a new stage with its own filesystem. You name stages with `AS <name>` and can copy files between stages:

```dockerfile
# ---- dev stage ----
FROM golang:1.24-alpine AS dev

WORKDIR /app

# Install Delve debugger
RUN go install github.com/go-delve/delve/cmd/dlv@latest

COPY . .

# Disable optimizations and inlining so dlv can set breakpoints accurately
RUN go build -gcflags="all=-N -l" -o app .

# dlv exec runs the binary under the debugger
CMD ["dlv", "exec", "./app", "--headless", "--listen=:2345", \
     "--api-version=2", "--accept-multiclient", "--continue"]

# ---- builder stage (intermediate) ----
FROM golang:1.24-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o app .

# ---- prod stage ----
FROM alpine:3.19 AS prod
WORKDIR /app
COPY --from=builder /app/app .
EXPOSE 8080
CMD ["./app"]
```

This single Dockerfile defines three stages:

1. **`dev`** -- Includes the Go toolchain and the Delve debugger. The binary is built with `-gcflags="all=-N -l"`, which disables compiler optimizations and function inlining. This is necessary for the debugger to set breakpoints accurately and inspect variables. The default command starts Delve in headless mode, listening on port 2345 for debugger connections.

2. **`builder`** -- An intermediate stage that compiles an optimized production binary. It uses the full Go toolchain but is not the final image.

3. **`prod`** -- Copies only the compiled binary from the `builder` stage into a minimal Alpine base image. No Go toolchain, no debugger, no source code, no build artifacts. The final image is as small as possible.

### Why multi-stage builds matter

Without multi-stage builds, you have two unappealing options:

1. **One Dockerfile with everything:** The production image includes the Go toolchain, Delve, source code, and intermediate build artifacts. The image is large (hundreds of megabytes), has a massive attack surface, and ships tools that have no business being in production.

2. **Two separate Dockerfiles:** `Dockerfile.dev` and `Dockerfile.prod` with duplicated instructions. When you change a dependency, update the Go version, or modify the build process, you have to update both files and keep them in sync.

Multi-stage builds give you one file that produces different images depending on which stage you target. The dev and prod builds share the same base instructions, so changes propagate to both automatically.

### Targeting stages in Docker Compose

The `target` field in a service's `build` configuration tells Compose which stage to build:

```yaml
services:
  app-dev:
    build:
      context: .
      target: dev
    ports:
      - "8080:8080"
      - "2345:2345"

  app-prod:
    build:
      context: .
      target: prod
    ports:
      - "8081:8080"
```

Two services, one Dockerfile. Each service builds a different stage:

- `docker compose up app-dev` builds the `dev` stage and starts the application with Delve attached.
- `docker compose up app-prod` builds the `prod` stage and starts the minimal production image.

Port 2345 is exposed on the `app-dev` service for debugger connections. The `app-prod` service maps to a different host port (8081) so both can run simultaneously if needed.

### Attaching a debugger

With the `app-dev` service running, Delve listens on port 2345 in headless mode. Connect from any Delve-compatible client:

**From the command line:**

```bash
dlv connect localhost:2345
```

**From VS Code** (add to `.vscode/launch.json`):

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to Docker",
      "type": "go",
      "request": "attach",
      "mode": "remote",
      "port": 2345,
      "host": "localhost"
    }
  ]
}
```

**From GoLand:** Create a "Go Remote" run configuration pointing at `localhost:2345`.

Once connected, you have full debugging capabilities: breakpoints, variable inspection, step-through execution, goroutine inspection -- all running inside the container, not on your host machine. The application behaves exactly as it would in Docker, with the same environment, networking, and filesystem.

### Handling code changes

The Compose file intentionally does not mount the host source directory into the container:

```yaml
services:
  app-dev:
    build:
      context: .
      target: dev
    ports:
      - "8080:8080"
      - "2345:2345"
    # No volumes: mounting would shadow the compiled binary
```

Mounting the host directory would shadow the binary compiled during the build. The container would start with source code from the host but the binary built during `docker build`, and they might not match. Worse, the mounted directory might lack the compiled binary entirely.

Instead, rebuild when your code changes:

```bash
docker compose build app-dev
docker compose up app-dev
```

For a compiled language like Go, a rebuild takes seconds. Docker's layer cache means that only the layers affected by your code change are rebuilt -- the Go toolchain installation and Delve installation are cached.

If you want faster iteration, combine this with a file watcher:

```bash
# Using entr (install with: brew install entr / apt install entr)
find . -name '*.go' | entr -r docker compose up --build app-dev
```

This watches for changes to `.go` files and automatically rebuilds and restarts the container.

### Applying the pattern to other languages

The multi-stage pattern is not Go-specific. Here are examples for other ecosystems:

**Node.js with debugging:**

```dockerfile
FROM node:20-alpine AS dev
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["node", "--inspect=0.0.0.0:9229", "server.js"]

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

FROM node:20-alpine AS prod
WORKDIR /app
COPY --from=builder /app .
CMD ["node", "server.js"]
```

Note: this example uses the same `node:20-alpine` base for both stages. For a production scenario where image size matters most, you could use a distroless Node.js image (e.g. `gcr.io/distroless/nodejs20-debian12`) as the final stage, which strips the shell and package manager entirely from the runtime image.

**Python with development tools:**

```dockerfile
FROM python:3.12-slim AS dev
WORKDIR /app
COPY requirements.txt requirements-dev.txt ./
RUN pip install -r requirements.txt -r requirements-dev.txt
COPY . .
CMD ["python", "-m", "debugpy", "--listen", "0.0.0.0:5678", "-m", "flask", "run"]

FROM python:3.12-slim AS prod
WORKDIR /app
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "app:create_app", "-b", "0.0.0.0:8080"]
```

The principle is the same regardless of language: the dev stage includes debugging tools and development dependencies; the prod stage includes only what is needed to run the application.

### Validating the production image locally

One of the key benefits of this pattern is that you can validate your production image locally before pushing it:

```bash
# Build and run the production image locally
docker compose up app-prod

# Test it
curl http://localhost:8081

# Check the image size
docker images | grep app-prod
```

If the production image works locally, you have high confidence it will work in production. The dev and prod stages share the same source code and build process, differing only in what tools are included and how the binary is compiled.

The full example for this section is at `examples/12-multistage-dev/` in the companion repository. It includes a complete Go application with the multi-stage Dockerfile and Compose configuration shown above.

## Conclusion

Docker Compose transforms the way you manage multi-container applications. Instead of juggling shell scripts, memorizing long `docker run` commands, and maintaining setup documentation that drifts out of date, you describe your entire stack in a single YAML file that lives in version control alongside your code. From basic service definitions and networking to healthchecks, scaling, profiles, configuration reuse, CI/CD integration, and multi-stage build workflows, Compose provides a consistent, declarative model that scales from a two-container demo to a complex development environment.

The patterns covered in this guide build on each other. Start with a simple Compose file (sections 1-3), add robustness with healthchecks and best practices (sections 4 and 7), debug effectively when things go wrong (section 8), and then adopt advanced patterns like profiles, include/extends, and multi-stage builds as your project grows (sections 9-12). Every section is designed to be independently useful -- you can adopt these practices incrementally.

All of the working examples from this guide are available in the companion repository at [github.com/example/docker-compose-guide](https://github.com/example/docker-compose-guide). Clone it, explore the `examples/` directory, and run `docker compose up` in any folder to see the concepts in action. If you find an issue or have a suggestion, open a pull request -- contributions are welcome.
