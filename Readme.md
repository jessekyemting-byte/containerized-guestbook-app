# My Containerized Guestbook Application

## Project Overview
This project deploys a multi-container web application using a Python Flask frontend and a Redis database. Users can submit guestbook messages through a simple form; entries are saved to and retrieved from a backend Redis instance running in its own container.

## Architecture Diagram
```text
      User (Browser)
            │
            ▼ [Port 5000:5000]
   ┌─────────────────────────────────┐
   │    Flask Frontend Container     │
   └─────────────────────────────────┘
            │
            ▼ [guestbook-network]
   ┌─────────────────────────────────┐
   │    Redis Database Container     │
   └─────────────────────────────────┘
            │
            ▼
   ┌─────────────────────────────────┐
   │  Named Volume (redis-data)      │
   └─────────────────────────────────┘
```

## Configuration Files

### 1. Dockerfile

```dockerfile
# Step 1: Use an official Python runtime as the base image
FROM python:3.10-slim

# Step 2: Set the working directory inside the container
WORKDIR /app

# Step 3: Copy the dependencies file first
COPY requirements.txt .

# Step 4: Install the Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Step 5: Copy the rest of the application code
COPY . .

# Step 6: Expose the Flask port
EXPOSE 5000

# Step 7: Define the command to run the application
CMD ["python", "app.py"]
```

### 2. docker-compose.yml

```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    networks:
      - guestbook-network
    depends_on:
      - redis

  redis:
    image: "redis:alpine"
    networks:
      - guestbook-network
    volumes:
      - redis-data:/data

networks:
  guestbook-network:
    driver: bridge

volumes:
  redis-data:
```

## Build Steps

To build the custom Flask frontend image from the local Dockerfile:

```bash
docker compose build
```

*Note: The deployment command below will also build the image automatically if it doesn't exist locally.*

## Run Steps

1. **Launch the containers:**

   ```bash
   docker compose up -d --build
   ```

2. **Open the app:**

   ```
   http://localhost:5000
   ```

3. **Tear down:**

   To stop and remove running containers without losing stored guestbook data:

   ```bash
   docker compose down
   ```

## Docker Compose Explanation

Docker Compose manages the lifecycle of both services from a single `docker-compose.yml` file. Rather than running containers manually with long `docker run` commands and hand-crafted network flags, Compose handles environment variables, build contexts, port mappings, and dependencies declaratively.

## Docker Network Explanation

Both containers communicate over a custom bridge network named `guestbook-network`. External traffic can't reach Redis directly through this network. The Flask app connects to Redis using the service name `redis` as the hostname, so no hardcoded IP addresses are needed.

## Docker Volume Explanation

A named volume called `redis-data` is mounted to `/data` inside the Redis container. This keeps guestbook entries on the host regardless of what happens to the container — restarts, replacements, and full teardowns all leave the data intact.

## Docker Scout Results

Image analyzed: `kyemtingguestbook-app-web:latest`

* **Base image:** `python:3.10-slim` (linux/amd64)
* **Packages audited:** 150 total, 12 with flagged vulnerabilities

| Severity | Count | Notable CVEs |
|---|---|---|
| 🔴 Critical | 1 | CVE-2026-12087 in `perl` |
| 🟠 High | 3 | CVE-2026-24049 (path traversal) in `wheel` |
| 🟡 Medium | 8 | CVE-2023-5752 (command injection) in `pip` |
| 🔵 Low | 26 | In `glibc`, `systemd`, and other base libraries |
| ⚪ Unspecified | 2 | |

**Remediation:** Upgrading the base image to `python:3.14-slim` should clear most of these. For quicker fixes, update `pip` to `>=23.3` and `wheel` to `>=0.46.2` directly in the build steps.

## Docker Hub Image

* **Repository:** [https://hub.docker.com/r/jessekyemting/guestbook](https://hub.docker.com/r/jessekyemting/guestbook)