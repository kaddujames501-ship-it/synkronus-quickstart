# Synkronus Quickstart

Welcome! This repository provides a ready-to-run setup of **ODE: Synkronus**, including a Postgres database, so you can try it out quickly or use it as a starting point for your own deployments.

---

## Features

* Fully containerized Synkronus server
* Includes Postgres database
* Supports local usage and GitHub Codespaces
* Easy environment variable configuration

---

## Quick Start

We recommend **Podman** with **podman compose** (or `podman-compose`); the same steps work with Docker and Docker Compose if you prefer.

> **Clean Ubuntu server?** Install the needed tools with:
> ```bash
> sudo apt update
> sudo apt install -y podman podman-compose git
> ```

### Easiest: run the installer

Clone the repo (shallow clone is enough), then from the repo root run the installer:

```bash
git clone --depth 1 https://github.com/OpenDataEnsemble/synkronus-quickstart.git
cd synkronus-quickstart
chmod +x ./install.sh
./install.sh
podman compose up -d
```

(With Docker, use `docker compose up -d` instead.)

The installer will:

- Generate strong passwords and inject them into `docker-compose.yml`
- Ask whether you have a **domain name** for this server:
  - **Yes** → You enter your domain. **Caddy** is placed in front and will automatically obtain and renew a TLS certificate (Let’s Encrypt). No Certbot or manual steps.
  - **No** → You enter this server’s **public IP** or `localhost`:
    - **Public IP** → The installer uses **&lt;ip&gt;.sslip.io** as the hostname so Caddy can still provision a real TLS certificate. You get HTTPS with no domain.
    - **localhost** → Caddy serves on port 80 only (no TLS), for local testing.

The script prints admin username and password (save them). Once the server is up, log in with those credentials and you can create new users from the UI. Use **https://your-domain/** or **https://&lt;your-ip&gt;.sslip.io/** (with TLS), or **http://localhost/** for local-only.

---

### Local Installation (manual)

1. Clone this repo:

```bash
git clone https://github.com/OpenDataEnsemble/synkronus-quickstart.git
cd synkronus-quickstart
```

1. Adjust env variables the `docker-compose.yml` file.

  - In the postgres service:
    - POSTGRES_PASSWORD
  - In the synkronus service:
    - DB_CONNECTION (update to match POSTGRES_PASSWORD)
    - JWT_SECRET (generate a new one with: 'openssl rand -base64 32')
    - ADMIN_USERNAME
    - ADMIN_PASSWORD

Optionally you can choose to map the volumes to specific mountpoints on the host.

1. Prepare a database for synkonus

   Start only the `db` service:

   ```bash
   podman compose up db
   ```

   (Use `docker compose up db` if you use Docker.) This keeps the database container running in the foreground.

   In a separate terminal, make the `create_sync_db.sh` script executable:

   ```bash
   chmod +x ./create_sync_db.sh
   ```

   Then run the script to create the Synkronus database and user:

   ```bash
   ./create_sync_db.sh
   ```

   The script will connect to the running `db` container and set up the required database and user account.

1. Start the services:

```bash
podman compose up -d
```

(Use `docker compose up -d` with Docker.)

1. Verify the server is running:

```bash
curl http://localhost:8080/health
# Should return "OK"
```

---

### Using GitHub Codespaces

1. Click **“Open in Codespaces”** on this repository.
2. Codespaces will automatically start Synkronus + Postgres.
3. Check the **Ports tab** for the forwarded port to access the API.
4. Test the server:

```bash
curl <forwarded-url>/health
```

> Notes:
>
> * Perfect for experimenting or as a base for production setups.

---

## Contributing / Feedback

We welcome feedback! Feel free to open issues or pull requests. If you’re trying this out for the first time, check the **Codespaces** instructions for the quickest setup.

---

Enjoy exploring Synkronus!
