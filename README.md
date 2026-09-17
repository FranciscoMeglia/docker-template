# Docker Compose Deployment Template

Template for deploying a **Django (API + Celery) + Frontend + PostgreSQL + Redis** project behind **Nginx**, using images from a private registry.

Services are split into several `docker-compose-*.yml` files so you can:

- **Run everything on a single VM**, or
- **Split services across different VMs** and connect them over the network.

---

## Structure

```
.
├── env.example.txt              # Global variables (copy to .env)
├── docker-compose-db.yml        # PostgreSQL + Redis
├── docker-compose-api.yml       # API (gunicorn) + Celery worker + Celery beat
├── docker-compose-frontend.yml  # Frontend
├── docker-compose-nginx.yml     # Reverse proxy
├── api/env.example.txt          # API variables (copy to api/.env)
├── db/env.example.txt           # Postgres variables (copy to db/.env)
├── frontend/env.example.txt     # Frontend variables (copy to frontend/.env)
└── nginx/
    ├── nginx.conf.template        # Subdomains + HTTPS
    └── nginx.conf.template.local  # Local IP + HTTP
```

Folders created at runtime (do not commit):

| Folder          | Contents                         |
| --------------- | -------------------------------- |
| `data/postgres` | PostgreSQL data                  |
| `data/redis`    | Redis data (AOF)                 |
| `certs/`        | SSL certificates (HTTPS mode)    |

Recommended `.gitignore`:

```gitignore
.env
*/.env
data/
certs/
```

---

## Services

| Compose                       | Service    | Container                  | Description                                 |
| ----------------------------- | ---------- | -------------------------- | ------------------------------------------- |
| `docker-compose-db.yml`       | `db`       | `${PROJECT_NAME}_db`       | PostgreSQL with healthcheck                 |
|                               | `redis`    | `${PROJECT_NAME}_redis`    | Redis with persistence (`appendonly`)       |
| `docker-compose-api.yml`      | `api`      | `${PROJECT_NAME}_api`      | Django with gunicorn on `${PORT_API}`       |
|                               | `worker`   | `${PROJECT_NAME}_worker`   | Celery worker (concurrency 4)               |
|                               | `beat`     | `${PROJECT_NAME}_beat`     | Celery beat                                 |
| `docker-compose-frontend.yml` | `frontend` | `${PROJECT_NAME}_frontend` | Frontend served on `${PORT_FRONTEND}`       |
| `docker-compose-nginx.yml`    | `nginx`    | `${PROJECT_NAME}_nginx`    | Reverse proxy, publishes `80` and `443`     |

All services join an **external** Docker network named `${DOCKER_NETWORK}`, which must be created before starting anything.

Only Nginx publishes ports to the host. Everything else uses `expose`, so it is only reachable from inside the Docker network.

---

## Configuration

### 1. Global variables

```bash
cp env.example.txt .env
```

| Variable                | Description                                                  | Example                                   |
| ----------------------- | ------------------------------------------------------------ | ----------------------------------------- |
| `PROJECT_NAME`          | Prefix for containers and images                             | `miproyecto`                              |
| `DOCKER_NETWORK`        | Name of the external Docker network                          | `miproyecto-network`                      |
| `REGISTRY`              | Registry for the app images                                  | `registry.franciscomeglia.com/desarrollo` |
| `API_VERSION`           | Tag of the `${PROJECT_NAME}-api` image                       | `1.0.0`                                   |
| `FRONTEND_VERSION`      | Tag of the `${PROJECT_NAME}-frontend` image                  | `1.0.0`                                   |
| `NGINX_VERSION`         | Nginx image tag                                              | `1.28.2-alpine`                           |
| `POSTGRES_VERSION`      | Postgres image tag                                           | `14.19-alpine3.21`                        |
| `REDIS_VERSION`         | Redis image tag                                              | `7-alpine`                                |
| `PORT_API`              | Internal gunicorn port                                       | `8000`                                    |
| `PORT_FRONTEND`         | Internal frontend port                                       | `80`                                      |
| `NGINX_TEMPLATE`        | Nginx template to use (see below)                            | `nginx.conf.template.local`               |
| `NGINX_DOMAIN_FRONTEND` | Frontend domain. **In local mode: the VM's IP**              | `app.mydomain.com` / `192.168.1.50`       |
| `NGINX_DOMAIN_API`      | API domain (HTTPS mode only)                                 | `api.mydomain.com`                        |
| `NGINX_CERT_PATH`       | Host folder containing the certificates                      | `./certs`                                 |
| `NGINX_CERT_FILE`       | Certificate (full chain)                                     | `fullchain.pem`                           |
| `NGINX_CERT_KEY`        | Private key                                                  | `privkey.pem`                             |

### 2. Per-service variables

```bash
cp db/env.example.txt db/.env
cp api/env.example.txt api/.env
cp frontend/env.example.txt frontend/.env
```

Fill these in for your project. Example `db/.env`:

```env
POSTGRES_DB=miproyecto
POSTGRES_USER=miproyecto
POSTGRES_PASSWORD=change-me
```

`api/.env` holds the Django app's own settings (database connection, Celery broker, `SECRET_KEY`, `ALLOWED_HOSTS`, etc.). Inside the same Docker network, the database and Redis are reachable by container name, for example:

```env
# example variable names: they depend on how your project reads its config
DB_HOST=miproyecto_db
DB_PORT=5432
CELERY_BROKER_URL=redis://miproyecto_redis:6379/0
```

---

## Nginx modes

Selected with `NGINX_TEMPLATE`. On startup, the official Nginx image substitutes the template variables (`envsubst`) and writes `/etc/nginx/conf.d/nginx.conf`.

### Local mode — `nginx.conf.template.local`

For access by **IP** over **HTTP**, with no domain or certificates.

Everything is served under `http://<NGINX_DOMAIN_FRONTEND>`:

| Path       | Target   |
| ---------- | -------- |
| `/`        | frontend |
| `/api/`    | api      |
| `/admin/`  | api      |
| `/static/` | api      |

Requests with any other `Host` are closed without a response (`444`).

### Production mode — `nginx.conf.template`

For **subdomains** over **HTTPS**.

| Domain                  | Path       | Target   |
| ----------------------- | ---------- | -------- |
| `NGINX_DOMAIN_FRONTEND` | `/`        | frontend |
| `NGINX_DOMAIN_API`      | `/api/`    | api      |
| `NGINX_DOMAIN_API`      | `/admin/`  | api      |
| `NGINX_DOMAIN_API`      | `/static/` | api      |

- Port `80` redirects everything to HTTPS.
- Certificates are mounted from `NGINX_CERT_PATH`. `NGINX_CERT_FILE` and `NGINX_CERT_KEY` must exist in that folder.
- TLS 1.2 and 1.3.

The upstreams use `resolve` with Docker's internal DNS (`127.0.0.11`). This lets Nginx start even if the API or frontend are not up yet, and picks up the new IP when a container is recreated.

---

## Single-VM deployment

```bash
# 1. Log in to the registry
docker login registry.franciscomeglia.com

# 2. Create the network (once)
docker network create miproyecto-network

# 3. Start in order
docker compose -p miproyecto-db       -f docker-compose-db.yml       up -d
docker compose -p miproyecto-api      -f docker-compose-api.yml      up -d
docker compose -p miproyecto-frontend -f docker-compose-frontend.yml up -d
docker compose -p miproyecto-nginx    -f docker-compose-nginx.yml    up -d
```

> Using a different `-p` for each compose file avoids *orphan containers* warnings. Those show up because all the files live in the same folder and share a project name by default.

### Common operations

```bash
# Update the API to a new version (change API_VERSION in .env)
docker compose -p miproyecto-api -f docker-compose-api.yml pull
docker compose -p miproyecto-api -f docker-compose-api.yml up -d

# Migrations / collectstatic
docker exec -it miproyecto_api python manage.py migrate
docker exec -it miproyecto_api python manage.py collectstatic --noinput

# Logs
docker compose -p miproyecto-api -f docker-compose-api.yml logs -f

# Reload Nginx after renewing certificates
docker exec miproyecto_nginx nginx -s reload

# Stop a group of services
docker compose -p miproyecto-frontend -f docker-compose-frontend.yml down
```

---

## Multi-VM deployment

Each VM has its own copy of the repo, its own `.env` and its own local Docker network, and starts only the compose files it needs. For example:

```
                 Internet / LAN
                       │
               ┌───────▼────────┐
  VM 1         │ nginx          │  :80 / :443
  (proxy)      │ frontend       │
               └───────┬────────┘
                       │ private network
               ┌───────▼────────┐
  VM 2         │ api            │  :8000
  (app)        │ worker / beat  │
               └───────┬────────┘
                       │ private network
               ┌───────▼────────┐
  VM 3         │ db             │  :5432
  (data)       │ redis          │  :6379
               └────────────────┘
```

Docker networks are not shared between hosts, so **container names do not resolve across VMs**. To reach a service on another VM, publish its port and point to that VM's IP. These are the changes:

### VM 3 — DB + Redis

Publish the ports in `docker-compose-db.yml`, **bound to the VM's private IP** so they are not exposed to the internet:

```yaml
  db:
    ports:
      - "10.0.0.3:5432:5432"

  redis:
    ports:
      - "10.0.0.3:6379:6379"
```

```bash
docker network create miproyecto-network
docker compose -p miproyecto-db -f docker-compose-db.yml up -d
```

### VM 2 — API + Celery

Publish the API port in `docker-compose-api.yml`:

```yaml
  api:
    ports:
      - "10.0.0.2:${PORT_API}:${PORT_API}"
```

In `api/.env`, point to the data VM's IP:

```env
DB_HOST=10.0.0.3
CELERY_BROKER_URL=redis://10.0.0.3:6379/0
```

```bash
docker network create miproyecto-network
docker compose -p miproyecto-api -f docker-compose-api.yml up -d
```

### VM 1 — Nginx + Frontend

In the Nginx template you are using, change the API upstream to VM 2's IP:

```nginx
upstream api {
  server 10.0.0.2:${PORT_API};
}
```

The `frontend` upstream stays the same because it runs on the same VM. If the frontend were on another VM too, do the same: publish `PORT_FRONTEND` there and point the upstream to that IP.

```bash
docker network create miproyecto-network
docker compose -p miproyecto-frontend -f docker-compose-frontend.yml up -d
docker compose -p miproyecto-nginx    -f docker-compose-nginx.yml    up -d
```

### Security between VMs

- Publish ports only on the **private IP**, never on `0.0.0.0`.
- Use a firewall (`ufw`, security groups, etc.) to limit who can reach each port. For example, `5432` and `6379` only from VM 2, and `8000` only from VM 1.
- Redis has no password by default. If it is reachable over the network, set `requirepass` and use `redis://:password@host:6379/0`.
- Docker writes its own iptables rules and **can bypass `ufw`**. That is why binding ports to the private IP matters.

---

## New project checklist

- [ ] Copy the template and rename `PROJECT_NAME` and `DOCKER_NETWORK`
- [ ] Set `REGISTRY`, `API_VERSION` and `FRONTEND_VERSION`
- [ ] Create `.env`, `api/.env`, `db/.env` and `frontend/.env`
- [ ] Choose `NGINX_TEMPLATE` and fill in the domains or IP
- [ ] (HTTPS) Copy the certificates to `NGINX_CERT_PATH`
- [ ] Run `docker network create <DOCKER_NETWORK>` on each VM
- [ ] If distributed: publish ports, update hosts in `api/.env` and the Nginx upstreams
- [ ] Start db → api → frontend → nginx
- [ ] Run `migrate` and `collectstatic`
