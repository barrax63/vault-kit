# Vault Kit

Vault Kit is a production-ready Docker Compose setup that runs [Vaultwarden](https://github.com/dani-garcia/vaultwarden) behind a hardened Nginx reverse proxy with TLS, backed by a tuned PostgreSQL 16 database.

## Features

- **Vaultwarden** — lightweight, self-hosted Bitwarden-compatible server
- **Nginx** — TLS-terminating reverse proxy with HTTP/2, HSTS, rate limiting, and security headers
- **PostgreSQL 16** — persistent storage with PGTune-optimized settings and data checksums
- **Security hardened** — all containers drop capabilities, use `no-new-privileges`, and run with AppArmor
- **Resource limits** — CPU and memory constraints on every service
- **SMTP support** — optional email configuration for invitations and password hints
- **Health checks** — built-in readiness probes for Nginx, Vaultwarden, and PostgreSQL

## Requirements

- Docker and Docker Compose (v2+)

## Project structure

```
vault-kit/
├── .env.example                  # Sample environment variables
├── .gitignore                    # Ignores .env
├── docker-compose.yml            # Full service stack
├── nginx/
│   ├── certs/                    # Place TLS cert & key here (gitignored contents)
│   └── nginx.conf                # Reverse proxy configuration
└── postgres-init/
    └── 01-init-postgres.sql      # PGTune-optimized PostgreSQL settings
```

## Quick start

1. **Clone the repository:**

   ```bash
   git clone https://github.com/barrax63/vault-kit.git
   cd vault-kit
   ```

2. **Create your `.env` file** from the example and edit the values:

   ```bash
   cp .env.example .env
   ```

   At a minimum, update these variables:

   | Variable            | Description                                       | Default               |
   | ------------------- | ------------------------------------------------- | --------------------- |
   | `NGINX_PORT`        | Port Nginx exposes on the host                    | `4097`                |
   | `POSTGRES_PASSWORD` | Database password                                 | `password`            |
   | `HOST`              | Your domain or IP (used for the Vaultwarden URL)  | `vault.example.com`   |
   | `ADMIN_TOKEN`       | Vaultwarden admin panel token (argon2 hash)       | *(empty — disabled)*  |
   | `TZ`                | Timezone                                          | `Europe/Berlin`       |

3. **Place your TLS certificate and key** in `nginx/certs/` as `server.crt` and `server.key`.

   > For local testing you can generate a self-signed certificate:
   >
   > ```bash
   > openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   >   -keyout nginx/certs/server.key \
   >   -out nginx/certs/server.crt \
   >   -subj "/CN=localhost"
   > ```

4. **Start the stack:**

   ```bash
   docker compose up -d
   ```

5. **Open Vaultwarden** at `https://localhost:4097` (or your configured `NGINX_PORT` and `HOST`).

## Configuration

### Standard variables

| Variable            | Description                                              | Default             |
| ------------------- | -------------------------------------------------------- | ------------------- |
| `NGINX_PORT`        | Host-side HTTPS port                                     | `4097`              |
| `POSTGRES_PASSWORD` | PostgreSQL password                                      | `password`          |
| `HOST`              | Public domain or IP for Vaultwarden                      | `vault.example.com` |
| `ADMIN_TOKEN`       | Admin panel token (see [generating a token](#admin-token)) | *(empty)*         |
| `TZ`                | Timezone for all services                                | `Europe/Berlin`     |

### Advanced variables

| Variable                    | Description                                          | Default              |
| --------------------------- | ---------------------------------------------------- | -------------------- |
| `ENV`                       | Environment mode                                     | `prod`               |
| `POSTGRES_USER`             | Database user                                        | `root`               |
| `POSTGRES_DB`               | Database name                                        | `vaultwarden`        |
| `SMTP_HOST`                 | SMTP server hostname                                 | `smtp.domain.tld`    |
| `SMTP_FROM`                 | Sender email address                                 | `vaultwarden@domain.tld` |
| `SMTP_PORT`                 | SMTP port                                            | `587`                |
| `SMTP_SECURITY`             | SMTP security mode (`starttls`, `force_tls`, `off`)  | `starttls`           |
| `SMTP_USERNAME`             | SMTP login username                                  | `myusername`         |
| `SMTP_PASSWORD`             | SMTP login password                                  | `MyPassw0rd`         |
| `SIGNUPS_ALLOWED`           | Allow open user registration                         | `true`               |
| `SHOW_PASSWORD_HINT`        | Display password hints on the hint page              | `false`              |
| `SIGNUPS_DOMAINS_WHITELIST` | Restrict registration to these email domains (CSV)   | *(empty)*            |

### Admin token

Generate an admin token from inside the running container:

```bash
docker exec -it vault-kit-vaultwarden-1 /vaultwarden hash
```

> **Important:** Escape every `$` in the resulting argon2 PHC string with `$$` before pasting it into your `.env` file.

## Architecture

```
                  ┌──────────────────┐
  Client ──HTTPS──▶  Nginx (:8443)   │
                  │  TLS + HTTP/2    │
                  └───────┬──────────┘
                          │ HTTP
                  ┌───────▼──────────┐
                  │  Vaultwarden     │
                  │  (:80 internal)  │
                  └───────┬──────────┘
                          │ PostgreSQL protocol
                  ┌───────▼──────────┐
                  │  PostgreSQL 16   │
                  │  (:5432 internal)│
                  └──────────────────┘
```

All services communicate over an internal Docker bridge network (`main`). Only Nginx exposes a port to the host.

## Health checks

| Service      | Endpoint / Command                                                 | Interval |
| ------------ | ------------------------------------------------------------------ | -------- |
| **Nginx**    | `https://localhost:8443/health` (returns `200 OK`)                 | 30 s     |
| **Postgres** | `pg_isready -h 127.0.0.1 -U $POSTGRES_USER -d $POSTGRES_DB`      | 10 s     |
| **Vaultwarden** | Built-in health check (Nginx depends on `service_healthy`)     | —        |

## Security

This setup applies several hardening measures out of the box:

- **Capability dropping** — all Linux capabilities are dropped; only the minimum required are re-added per service.
- **`no-new-privileges`** — prevents privilege escalation inside containers.
- **AppArmor** — enforces the `docker-default` AppArmor profile.
- **Non-root users** — Vaultwarden runs as `1000:1000`; PostgreSQL runs as `postgres`.
- **Read-only mounts** — configuration files and certificates are mounted `:ro`.
- **TLS 1.2+ only** — modern cipher suites, HSTS, session tickets disabled.
- **Security headers** — `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`, `Referrer-Policy`, and `Strict-Transport-Security`.
- **Rate limiting** — Nginx limits requests to 50 req/s per IP with a burst of 100.
- **Resource constraints** — every service has CPU and memory limits and reservations.
- **Signup control** — disable open registration after creating your account by setting `SIGNUPS_ALLOWED=false`.

## Useful commands

```bash
# Start the stack
docker compose up -d

# View logs
docker compose logs -f

# Restart after configuration changes
docker compose restart

# Stop the stack
docker compose down

# Stop and remove volumes (⚠️ destroys all data)
docker compose down -v
```

## Updating

Pull the latest images and recreate the containers:

```bash
docker compose pull
docker compose up -d --force-recreate
```

> **Tip:** Always back up your PostgreSQL data before upgrading.

## License

MIT License (see `LICENSE`).
