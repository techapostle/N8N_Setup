# Running n8n in Docker on WSL (with Docker Compose)

This guide walks through:

- Verifying Docker is running under WSL (systemd)  
- Creating a minimal `docker-compose.yml` for n8n  
- Starting n8n  
- Viewing live logs  
- Shutting it down cleanly  

This is for **local dev**, not hardened production.

---

## 1. Prerequisites (native Docker in WSL)

### 1.1. WSL2 distro and updates

Use an Ubuntu (or similar) WSL2 distro and update it:

```bash
sudo apt update && sudo apt upgrade -y
```

This document assumes:

- Docker Engine is already installed **inside WSL**.
- `docker compose` (the plugin, not the old `docker-compose` Python package) is installed.

### 1.2. Make sure Docker’s systemd service is running

Check the Docker service:

```bash
sudo systemctl status docker
```

- If you see `active (running)`, you’re good.
- If it’s `inactive`, `failed`, or similar, start it:

```bash
sudo systemctl start docker
```

Optionally enable it to start automatically when WSL boots (assuming systemd is enabled for your WSL distro):

```bash
sudo systemctl enable docker
```

Sanity check:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

If these commands work without errors, Docker is ready.

If you get permission errors, add your user to the `docker` group:

```bash
sudo usermod -aG docker $USER
# then completely close and reopen your WSL session
```

---

## 2. Project structure

Pick a folder for n8n inside WSL:

```bash
mkdir -p ~/n8n && cd ~/n8n
mkdir n8n_data
```

We’ll put the Compose file in `~/n8n/docker-compose.yml` and persist n8n data in `./n8n_data`.

---

## 3. Minimal `docker-compose.yml` for n8n

Create the file:

```bash
nano docker-compose.yml
```

Paste this:

```yaml
version: "3.8"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped

    # Expose n8n on localhost:5678
    ports:
      - "5678:5678"

    # Persistent data (credentials, workflows, etc.)
    volumes:
      - ./n8n_data:/home/node/.n8n

    environment:
      # Basic local dev settings
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http

      # For local dev ONLY; production should use HTTPS + secure cookies
      - N8N_SECURE_COOKIE=false

      # Optional: simple HTTP auth in front of n8n UI
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=changeme-strong
```

You can tweak ports, auth, and other env vars later as needed.

---

## 4. Start n8n with Docker Compose

From the `~/n8n` directory:

```bash
docker compose up -d
```

- `-d` runs it detached (in the background).

Check that the container is up:

```bash
docker compose ps
```

You should see `n8n` with `State` = `running`.

### 4.1. Accessing the UI

On Windows or inside WSL, open your browser and go to:

```text
http://localhost:5678
```

If you left basic auth enabled, log in with `admin / changeme-strong` (change that to something sane).

---

## 5. Viewing live logs

Tail logs in real time:

```bash
docker compose logs -f n8n
```

- `-f` = follow (like `tail -f`).
- Ctrl-C to stop watching logs; the container keeps running.

Limit initial history if you want:

```bash
docker compose logs --tail=100 -f n8n
```

---

## 6. Stopping and shutting down n8n

Two levels of shutdown:

### 6.1. Temporarily stop containers (keep everything defined)

```bash
docker compose stop
```

Later, start again:

```bash
docker compose start
```

Data in `./n8n_data` stays untouched.

### 6.2. Tear down the stack

Stop and remove containers (but keep data on disk):

```bash
docker compose down
```

Also remove any named volumes defined in Compose (not our bind mount):

```bash
docker compose down --volumes
```

If you truly want a full reset, you can also nuke the directory:

```bash
rm -rf n8n_data
```

---

## 7. WSL-specific notes

- `ports: "5678:5678"` exposes n8n to Windows automatically at `http://localhost:5678` under WSL2.
- Keep the project on the Linux side (e.g., `/home/youruser/n8n`), not under `/mnt/c/...`, for better performance.
- If `systemctl` doesn’t work at all, you probably don’t have systemd enabled in WSL; then you either have to enable it or start Docker with whatever method you originally installed it with.

That’s your full lifecycle: make the Compose file → ensure `docker` service is running → `docker compose up -d` → `docker compose logs -f n8n` → `docker compose down`.
