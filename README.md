# Guacamole PostgreSQL + Cloudflare Tunnel (AutoInit)

A reproducible, self-hosted **Apache Guacamole** stack using **Docker Compose**, **PostgreSQL**, and **Cloudflare Tunnel**.

This repository automatically provisions PostgreSQL, applies the Guacamole database schema (idempotently), and starts **guacd**, the **Guacamole web application**, and an optional **Cloudflare Tunnel** with minimal manual configuration.

---

## Overview

This project provides a ready-to-run Docker Compose stack that:

- Boots PostgreSQL with a persistent Docker volume.
- Automatically creates the Guacamole database and user.
- Applies the Guacamole JDBC schema only when required.
- Starts `guacd` and the official Guacamole web application.
- Optionally exposes Guacamole securely through Cloudflare Tunnel.
- Uses an idempotent initialization process that is safe across restarts and existing database volumes.

---

## Features

- ✅ Automatic PostgreSQL provisioning
- ✅ Idempotent schema installation
- ✅ Persistent database storage
- ✅ Optional Cloudflare Tunnel integration
- ✅ Production deployment recommendations
- ✅ Minimal setup

---

# Quick Start

## 1. Clone the repository

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
````

---

## 2. Create a `.env` file

```env
POSTGRES_PASSWORD="yourStrongPassword"
TUNNEL_TOKEN="your_cloudflare_tunnel_token"
```

---

## 3. (Optional) Create `initdb.sql`

This script only executes during the **first PostgreSQL initialization**.

```sql
CREATE DATABASE IF NOT EXISTS guacamole_db;

CREATE USER IF NOT EXISTS guacamole_user
WITH PASSWORD 'yourStrongPassword';

GRANT ALL PRIVILEGES
ON DATABASE guacamole_db
TO guacamole_user;
```

---

## 4. (Optional) Download the Guacamole schema

If you prefer to apply the schema manually:

```bash
curl -fsSL -o schema.sql \
"https://raw.githubusercontent.com/apache/guacamole-client/master/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-postgresql/src/main/resources/schema/postgresql/schema.sql"
```

---

## 5. Start the stack

```bash
sudo docker compose up -d
```

---

## 6. Watch the logs

```bash
sudo docker compose logs -f postgres guacamole
```

---

## 7. Access Guacamole

Local:

```
http://127.0.0.1:8080
```

Cloudflare Tunnel:

If `TUNNEL_TOKEN` is configured, `cloudflared` automatically exposes the application securely.

---

# Project Structure

```
.
├── docker-compose.yaml
├── .env
├── initdb.sql
├── schema.sql
└── README.md
```

---

# Configuration

## Important files

| File                  | Description                                                      |
| --------------------- | ---------------------------------------------------------------- |
| `docker-compose.yaml` | Defines the PostgreSQL, Guacamole, guacd and Cloudflare services |
| `.env`                | Environment variables and secrets                                |
| `initdb.sql`          | Optional SQL executed on the first PostgreSQL startup            |
| `schema.sql`          | Guacamole PostgreSQL schema                                      |

---

## Environment Variables

| Variable            | Description                             |
| ------------------- | --------------------------------------- |
| `POSTGRES_PASSWORD` | Password for PostgreSQL                 |
| `POSTGRES_USER`     | PostgreSQL user (optional)              |
| `POSTGRES_DB`       | Database name (default: `guacamole_db`) |
| `TUNNEL_TOKEN`      | Cloudflare Tunnel token                 |

---

# Example Docker Compose

```yaml
services:
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_DB: guacamole_db
      POSTGRES_USER: guacamole_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql
      - ./initdb.sql:/docker-entrypoint-initdb.d/initdb.sql:ro

  guacd:
    image: guacamole/guacd:latest

  guacamole:
    image: guacamole/guacamole:latest
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRESQL_HOSTNAME: postgres
      POSTGRESQL_DATABASE: guacamole_db
      POSTGRESQL_USERNAME: guacamole_user
      POSTGRESQL_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "8080:8080"

  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token ${TUNNEL_TOKEN}
```

---

# Applying the Guacamole Schema

If the schema was not automatically installed:

## Download the schema

```bash
curl -fsSL -o schema.sql \
"https://raw.githubusercontent.com/apache/guacamole-client/master/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-postgresql/src/main/resources/schema/postgresql/schema.sql"
```

## Copy into the PostgreSQL container

```bash
sudo docker compose cp schema.sql postgres:/tmp/schema.sql
```

## Apply it

```bash
sudo docker compose exec postgres \
psql -U guacamole_user \
-d guacamole_db \
-f /tmp/schema.sql
```

> If `guacamole_user` does not have sufficient privileges, execute the command as the PostgreSQL superuser.

---

# Troubleshooting

## SCRAM authentication or empty password errors

Verify the password inside `.env` matches the password stored in PostgreSQL.

Update the password without recreating the database:

```bash
sudo docker compose exec postgres \
psql -U <existing-superuser> \
-c "ALTER USER guacamole_user WITH PASSWORD 'yourStrongPassword';"

sudo docker compose restart guacamole
```

---

## Database does not exist

```bash
sudo docker compose exec postgres \
psql -U <superuser> \
-c "CREATE DATABASE guacamole_db;"

sudo docker compose exec postgres \
psql -U <superuser> \
-c "GRANT ALL PRIVILEGES ON DATABASE guacamole_db TO guacamole_user;"
```

---

## Guacamole tables are missing

Apply the Guacamole schema as described above.

---

## Init scripts did not run

The PostgreSQL Docker image only executes files inside:

```
/docker-entrypoint-initdb.d/
```

when the database directory is **empty**.

If PostgreSQL has already been initialized, use SQL commands (`CREATE`, `ALTER`, etc.) or recreate the Docker volume if a full reset is acceptable.

---

# Useful Diagnostic Commands

View Guacamole PostgreSQL environment variables:

```bash
sudo docker compose exec guacamole env | grep -i POSTGRES
```

List tables:

```bash
sudo docker compose exec postgres \
psql -U guacamole_user \
-d guacamole_db \
-c "\dt"
```

View Guacamole logs:

```bash
sudo docker compose logs -f guacamole
```

---

# Security Recommendations

* Use **Docker Secrets** or a dedicated secret manager instead of plaintext `.env` files in production.
* Quote passwords inside `.env` to preserve special characters.
* Bind Guacamole to `127.0.0.1:8080` if only local access is required.
* Use Cloudflare Tunnel instead of exposing port `8080` publicly.
* Regularly back up the PostgreSQL volume.

---

# Contributing

Contributions are welcome.

Please include:

* Reproduction steps
* Logs
* Expected behaviour
* Environment information

Documentation improvements and examples are always appreciated.

---

# License

This project is released under the **MIT License**.

Add a `LICENSE` file to the repository root before publishing.

```
```