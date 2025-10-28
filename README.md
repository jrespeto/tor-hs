# Tor Hidden Service Setup Guide

A comprehensive Docker-based setup for hosting a website as a Tor hidden service (.onion site). This project demonstrates how to expose any web application on the Tor network for enhanced privacy and anonymity.

## Table of Contents

- [What is a Tor Hidden Service?](#what-is-a-tor-hidden-service)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Detailed Setup Guide](#detailed-setup-guide)
- [Configuration Explained](#configuration-explained)
- [Using Nyx (Tor Monitor)](#using-nyx-tor-monitor)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Advanced Usage](#advanced-usage)
- [Common Use Cases](#common-use-cases)

---

## What is a Tor Hidden Service?

A Tor hidden service (also called an onion service) is a website or service that is only accessible through the Tor network. Key features:

- **Anonymous hosting**: The server's IP address is hidden from visitors
- **End-to-end encryption**: Traffic is encrypted through the Tor network
- **No DNS required**: Accessed via `.onion` addresses
- **Censorship resistant**: Difficult to block or take down

**Learning Resources:**
- [Tor Project: Hidden Services](https://community.torproject.org/onion-services/)
- [How Tor Works](https://2019.www.torproject.org/about/overview.html.en)

---

## Architecture Overview

This setup uses three Docker containers working together:

```
┌─────────────────────────────────────────┐
│  Internet (Tor Network)                 │
└────────────────┬────────────────────────┘
                 │
                 ▼
         ┌───────────────┐
         │  Tor Container │  (Port 80 → 8080)
         │  - Routes traffic from Tor network
         │  - Manages .onion identity
         │  - Control port: 9051
         └───────┬───────┘
                 │
                 ▼
      ┌──────────────────┐
      │  Nginx Container  │  (Port 8080 → 80)
      │  - Reverse proxy
      │  - Request routing
      │  - Header management
      └────────┬─────────┘
               │
               ▼
     ┌─────────────────────┐
     │  Backend Container   │  (Port 80)
     │  - Your web application
     │  - Can be any web service
     └─────────────────────┘
```

**Traffic Flow:**
1. User connects to your `.onion` address via Tor Browser
2. Tor container receives the request and forwards to nginx on port 8080
3. Nginx proxies the request to the backend application on port 80
4. Response flows back through the same chain

---

## Prerequisites

### Required Software

- **Docker**: Version 20.10 or higher
  - [Install Docker](https://docs.docker.com/get-docker/)
- **Docker Compose**: Version 2.0 or higher
  - Usually included with Docker Desktop
  - For Linux: [Install Docker Compose](https://docs.docker.com/compose/install/)

**Verify installations:**
```bash
docker --version
docker-compose --version
```

### Required Knowledge (Beginner-Friendly)

- Basic command line usage
- Understanding of Docker containers (optional but helpful)
- Basic web server concepts

---

## Project Structure

```
tor-hs/
├── docker-compose.yml      # Orchestrates all containers
├── config/
│   └── torrc              # Tor configuration file
├── nginx/
│   └── nginx.conf         # Nginx reverse proxy configuration
├── tor_data/              # Tor data (auto-generated, gitignored)
│   ├── hidden_service/    # Contains your .onion identity
│   │   ├── hostname       # Your .onion address
│   │   ├── hs_ed25519_public_key
│   │   └── hs_ed25519_secret_key
│   └── .nyx/              # Nyx monitoring tool cache
├── .gitignore             # Excludes sensitive data from git
└── README.md              # This guide
```

**Important Files:**
- `tor_data/hidden_service/hostname`: Your public .onion address
- `tor_data/hidden_service/hs_ed25519_secret_key`: **KEEP THIS SECRET!** This is your identity
- `config/torrc`: Tor daemon configuration

**Note:** The `tor_data/hidden_service/` directory and its contents are **automatically created** by the Tor container on first run. If this directory doesn't exist when you start the services, Tor will:
1. Create the directory structure
2. Generate a new cryptographic key pair (Ed25519)
3. Derive your unique .onion address from the public key
4. Save the keys and hostname to the directory

This means you don't need to create anything manually - just run `docker-compose up -d` and Tor handles the rest!

---

## Quick Start

### 1. Clone or Download This Project

```bash
git clone <repository-url>
cd tor-hs
```

### 2. (Optional) Replace Backend with Your Application

Edit `docker-compose.yml` and replace the backend service:

```yaml
backend:
  image: your-docker-image:tag  # Replace with your app
  container_name: backend
  # Add any environment variables, volumes, etc.
```

**For testing, you can use the default nginx container which serves a welcome page.**

### 3. Set Tor Control Password

Edit `docker-compose.yml` and change the password:

```yaml
setup-tor-pass:
  environment:
    TOR_CTRL_PASS: "YourSecurePasswordHere"  # Change this!
```

### 4. Start the Services

```bash
docker-compose up -d
```

**What happens:**
1. `setup-tor-pass` container generates a hashed password and updates `torrc`
2. `backend` container starts your web application
3. `nginx` container starts and connects to backend
4. `tor` container starts and automatically:
   - Creates the `tor_data/hidden_service/` directory (if it doesn't exist)
   - Generates a new Ed25519 key pair for your hidden service
   - Derives your unique .onion address from the public key
   - Saves the keys and hostname to `tor_data/hidden_service/`
   - Publishes your hidden service descriptor to the Tor network
   - Begins accepting connections

**First Run Note:** If you don't have a `tor_data/hidden_service/` directory, the Tor container will create it automatically with a fresh identity. This takes about 30-60 seconds. On subsequent runs, Tor will reuse the existing keys, preserving your .onion address.

### 5. Get Your .onion Address

**Wait a moment:** On first run, it takes 30-60 seconds for Tor to generate your identity and publish to the network.

```bash
docker exec tor cat /var/lib/tor/hidden_service/hostname
```

Example output: `abc123def456ghi789jklmnopqrstuvwxyz234567.onion` (56 characters for v3 onion addresses)

**If you get an error** like "No such file or directory", wait a bit longer - the Tor container is still initializing and creating the `hidden_service/` directory.

**What's happening behind the scenes:**
- First time: Tor generates new keys, creates the directory structure, and saves your .onion address
- Subsequent runs: Tor reads the existing keys and uses the same .onion address

### 6. Visit Your Site

1. Download [Tor Browser](https://www.torproject.org/download/)
2. Open Tor Browser
3. Navigate to your `.onion` address
4. Your site should be accessible!

---

## Detailed Setup Guide

### Step 1: Understanding the Docker Compose Configuration

The `docker-compose.yml` defines four services:

#### Service 1: setup-tor-pass (Init Container)

```yaml
setup-tor-pass:
  image: docker.io/dockurr/tor
  environment:
    TOR_CTRL_PASS: "PASSWORD"  # Change this!
  volumes:
    - ./config:/etc/tor
  entrypoint: ["/bin/sh","-c"]
  command:
    - |
      set -eu
      sed -i '/^HashedControlPassword/d' /etc/tor/torrc
      echo "HashedControlPassword `tor --hash-password $TOR_CTRL_PASS`"  >> /etc/tor/torrc
  restart: "no"
```

**Purpose:** Generates a hashed password for Tor's control port and writes it to `torrc`. This runs once before the Tor container starts.

**Why it matters:** The control port (9051) allows you to monitor and control Tor using tools like Nyx. The password protects this access.

#### Service 2: backend (Your Application)

```yaml
backend:
  image: docker.io/nginx  # Replace with your app
  container_name: backend
```

**Purpose:** This is your actual web application. Can be any containerized web service.

**Examples:**
- Static site: `nginx:alpine` with mounted HTML files
- Node.js app: `node:18-alpine` with your Express app
- Python app: `python:3.11-slim` with Flask/FastAPI
- WordPress: `wordpress:latest`

#### Service 3: nginx (Reverse Proxy)

```yaml
nginx:
  image: docker.io/nginx
  container_name: nginx
  depends_on:
    backend:
      condition: service_started
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
  network_mode: "service:backend"
```

**Purpose:** Acts as a reverse proxy between Tor and your backend application.

**Why use nginx?**
- Handles HTTP headers properly for Tor
- Can serve multiple backends
- Provides caching, compression, security headers
- Isolates your application from direct Tor traffic

**Network mode:** Uses the backend container's network namespace, so it can access localhost:80

#### Service 4: tor (Tor Daemon)

```yaml
tor:
  image: docker.io/dockurr/tor
  container_name: tor
  volumes:
    - ./config:/etc/tor
    - ./tor_data:/var/lib/tor
  restart: always
  depends_on:
    setup-tor-pass:
      condition: service_completed_successfully
    nginx:
      condition: service_started
  network_mode: "service:nginx"
```

**Purpose:** Runs the Tor daemon that creates and manages your hidden service.

**Network mode:** Uses nginx's network namespace, which in turn uses backend's namespace. This allows Tor to reach nginx on localhost:8080.

### Step 2: Configuring Tor (torrc)

The `config/torrc` file configures how Tor operates:

```bash
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServiceVersion 3
HiddenServicePort 80 127.0.0.1:8080
ControlPort 9051
HashedControlPassword 16:##########################################################
```

**Line-by-line explanation:**

| Directive | Explanation |
|-----------|-------------|
| `HiddenServiceDir` | Directory where Tor stores your .onion identity keys and hostname |
| `HiddenServiceVersion 3` | Use version 3 onion addresses (56 characters, more secure than v2) |
| `HiddenServicePort 80 127.0.0.1:8080` | Forward requests to .onion:80 → localhost:8080 (nginx) |
| `ControlPort 9051` | Enable control port for monitoring with Nyx |
| `HashedControlPassword` | Hashed password for control port access (auto-generated) |

**Common modifications:**

```bash
# Add multiple ports
HiddenServicePort 80 127.0.0.1:8080
HiddenServicePort 443 127.0.0.1:8443

# Multiple hidden services in one container
HiddenServiceDir /var/lib/tor/service1/
HiddenServicePort 80 127.0.0.1:8080

HiddenServiceDir /var/lib/tor/service2/
HiddenServicePort 80 127.0.0.1:9090
```

### Step 3: Configuring Nginx

The `nginx/nginx.conf` file:

```nginx
events {}

http {
    upstream backend {
        server 127.0.0.1:80;
    }

    access_log /dev/stdout;
    error_log /dev/stderr warn;

    server {
        listen 8080;
        server_name _;

        location / {
            proxy_pass http://backend/;
        }
    }
}
```

**Explanation:**

- `upstream backend`: Defines the backend server location (localhost:80 via shared network)
- `listen 8080`: Nginx listens on 8080 for requests from Tor
- `proxy_pass http://backend/`: Forwards all requests to the backend
- `access_log /dev/stdout`: Logs to Docker logs (viewable with `docker-compose logs nginx`)

**Advanced nginx configuration:**

```nginx
server {
    listen 8080;
    server_name _;

    # Security headers for Tor
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer" always;

    # Preserve client info
    location / {
        proxy_pass http://backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Step 4: Deploying Your Own Application

**Example 1: Static HTML Site**

Create an `html/` directory with your site:

```bash
mkdir html
echo "<h1>My Hidden Service</h1>" > html/index.html
```

Update `docker-compose.yml`:

```yaml
backend:
  image: docker.io/nginx:alpine
  container_name: backend
  volumes:
    - ./html:/usr/share/nginx/html:ro
```

**Example 2: Node.js Application**

If you have a Node.js app that listens on port 3000:

1. Create a Dockerfile for your app
2. Update `docker-compose.yml`:

```yaml
backend:
  build: ./your-nodejs-app
  container_name: backend
  environment:
    PORT: 80  # Make your app listen on port 80
```

3. Ensure your Node.js app binds to `0.0.0.0:80`

**Example 3: Python Flask App**

```yaml
backend:
  build: ./your-flask-app
  container_name: backend
  environment:
    FLASK_APP: app.py
  command: flask run --host=0.0.0.0 --port=80
```

---

## Configuration Explained

### Network Architecture

This setup uses Docker's `network_mode: "service:X"` to share network namespaces:

```
backend (has its own network namespace)
    ↓ (shares network)
  nginx (can access backend via 127.0.0.1:80)
    ↓ (shares network)
  tor (can access nginx via 127.0.0.1:8080)
```

**Benefits:**
- No need for a custom Docker network
- Services communicate via localhost (faster, more secure)
- Simplified networking model

**Trade-off:**
- Services share the same network namespace
- Port conflicts must be avoided

### Ports Summary

| Service | Internal Port | Purpose |
|---------|---------------|---------|
| Backend | 80 | Serves your web application |
| Nginx | 8080 | Receives traffic from Tor, proxies to backend:80 |
| Tor | 9051 | Control port (Nyx access) |
| Tor | (Hidden) | Tor network communication (handled internally) |

### Volume Mounts

| Volume | Purpose |
|--------|---------|
| `./config:/etc/tor` | Tor configuration (torrc) |
| `./tor_data:/var/lib/tor` | Tor state, keys, and .onion identity |
| `./nginx/nginx.conf:/etc/nginx/nginx.conf:ro` | Nginx configuration (read-only) |

**Important:** The `tor_data/` directory contains your secret keys. Backup this directory to preserve your .onion address!

---

## Using Nyx (Tor Monitor)

Nyx is a command-line monitor for Tor. It provides real-time information about your Tor connection, bandwidth usage, and more.

### Access Nyx

```bash
docker exec -it tor nyx
```

When prompted, enter the password you set in `TOR_CTRL_PASS` (not the hashed version).

### Nyx Interface

- **First page**: Overview of Tor connection, bandwidth, circuit info
- **Arrow keys**: Navigate
- **Page Down**: View different panels (connections, configuration, logs)
- **q**: Quit

**What to look for:**
- "Hidden service descriptor published": Your .onion is accessible
- Circuit count: Number of active Tor circuits
- Bandwidth: Upload/download speeds

### Troubleshooting Nyx Access

If you get "unable to resolve" or authentication errors:

1. Verify the password in `docker-compose.yml` matches what you're typing
2. Check that `torrc` has the `HashedControlPassword` line
3. Restart the Tor container: `docker-compose restart tor`

---

## Troubleshooting

### Service Won't Start

**Check logs:**
```bash
docker-compose logs tor
docker-compose logs nginx
docker-compose logs backend
```

**Common issues:**
- Port already in use: Change ports in `docker-compose.yml` or `nginx.conf`
- Permission errors: Ensure `./tor_data` directory is writable
- Image pull failures: Check internet connection, try `docker pull dockurr/tor`

### Can't Access .onion Address

1. **Verify Tor container is running:**
   ```bash
   docker ps | grep tor
   ```

2. **Check if hostname file exists:**
   ```bash
   docker exec tor cat /var/lib/tor/hidden_service/hostname
   ```
   If this fails, the hidden service hasn't initialized.

3. **Check Tor logs for errors:**
   ```bash
   docker-compose logs tor | grep -i error
   ```

4. **Verify with Nyx:**
   Use Nyx to check if the hidden service descriptor is published.

5. **Wait 5-10 minutes:**
   Sometimes it takes a few minutes for the hidden service to propagate through the Tor network.

### Backend Not Responding

1. **Test backend directly:**
   ```bash
   docker exec nginx curl http://127.0.0.1:80
   ```
   This should return your backend's response.

2. **Test nginx proxy:**
   ```bash
   docker exec tor curl http://127.0.0.1:8080
   ```

3. **Check nginx configuration:**
   ```bash
   docker exec nginx nginx -t
   ```

### Permission Denied Errors

```bash
sudo chown -R 1000:1000 ./tor_data
```

The Tor container runs as a specific user and needs write access to `tor_data/`.

### Hidden Service Directory Issues

**Problem:** `.onion address keeps changing` or `Can't find hostname file`

**Solution:**
1. **Check if directory exists and persists:**
   ```bash
   ls -la tor_data/hidden_service/
   ```
   You should see `hostname`, `hs_ed25519_public_key`, and `hs_ed25519_secret_key`

2. **Verify volume mount:**
   ```bash
   docker inspect tor | grep -A 10 Mounts
   ```
   Confirm that `./tor_data` is mounted to `/var/lib/tor`

3. **If directory is missing after restart:**
   - Check that `tor_data/` is not in `.gitignore` in a way that deletes it
   - Ensure Docker has permission to create the directory
   - Verify the volume is not set to `tmpfs` (temporary filesystem)

**Understanding the behavior:**
- **First run (no directory):** Tor creates `tor_data/hidden_service/` and generates new identity
- **Subsequent runs (directory exists):** Tor uses existing keys, same .onion address
- **After deletion:** If you delete `tor_data/hidden_service/`, Tor will generate a NEW identity with a DIFFERENT .onion address

**Important:** Once `tor_data/hidden_service/` is created with your keys, you MUST preserve this directory to keep the same .onion address. If you lose these files, your .onion address is lost forever and cannot be recovered.

---

## Security Considerations

### Protecting Your Identity

1. **Never expose your server's IP address:**
   - Don't reference your real domain in content
   - Avoid leaking metadata (EXIF data in images, etc.)

2. **Backup your keys securely:**
   ```bash
   tar -czf tor-backup.tar.gz tor_data/hidden_service/
   gpg -c tor-backup.tar.gz  # Encrypt with password
   ```

3. **Use strong passwords:**
   - Change `TOR_CTRL_PASS` from the default
   - Use a password manager

4. **Keep software updated:**
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

### Application Security

1. **Disable external connections:**
   - Your backend should only listen on localhost
   - Don't expose ports in `docker-compose.yml` unless necessary

2. **Use security headers:**
   - Add headers in nginx.conf (see Advanced nginx configuration)

3. **Monitor logs:**
   ```bash
   docker-compose logs -f --tail=100
   ```

4. **Avoid information disclosure:**
   - Remove server version headers
   - Use generic error pages
   - Don't expose stack traces

### What Tor DOESN'T Hide

- **Content-based identification**: If your site has unique content linked to your identity
- **Timing attacks**: Sophisticated adversaries can correlate traffic timing
- **Operational security failures**: Logging into your clearnet accounts while managing your service
- **Application vulnerabilities**: Tor doesn't protect against XSS, SQLi, etc.

---

## Advanced Usage

### Running Multiple Hidden Services

Edit `config/torrc`:

```bash
HiddenServiceDir /var/lib/tor/service1/
HiddenServicePort 80 127.0.0.1:8080

HiddenServiceDir /var/lib/tor/service2/
HiddenServicePort 80 127.0.0.1:9090
```

Update nginx.conf to proxy different ports or hostnames.

### Using Custom .onion v3 Addresses (Vanity Addresses)

You can generate custom .onion addresses with prefixes using tools like `mkp224o`:

```bash
docker run --rm -it -v $PWD:/keys ghcr.io/cathugger/mkp224o:master -d /keys someword
```

Then copy the generated keys to `tor_data/hidden_service/`.

**Warning:** Generating longer vanity addresses takes exponentially more time!

### SSL/TLS (HTTPS) for .onion

While not strictly necessary (Tor already encrypts traffic), you can add TLS:

1. Generate self-signed certificate or use Let's Encrypt
2. Configure nginx to listen on 443
3. Update torrc: `HiddenServicePort 443 127.0.0.1:8443`

### Auto-Backup Script

```bash
#!/bin/bash
# backup-tor-keys.sh
DATE=$(date +%Y%m%d_%H%M%S)
tar -czf "tor-backup-$DATE.tar.gz" tor_data/hidden_service/
gpg -c "tor-backup-$DATE.tar.gz"
rm "tor-backup-$DATE.tar.gz"
echo "Backup created: tor-backup-$DATE.tar.gz.gpg"
```

Run weekly with cron:
```bash
0 3 * * 0 /path/to/backup-tor-keys.sh
```

---

## Common Use Cases

### 1. Anonymous Blog

Deploy a static site generator (Hugo, Jekyll) or WordPress:

```yaml
backend:
  image: wordpress:latest
  environment:
    WORDPRESS_DB_HOST: db
    WORDPRESS_DB_USER: wordpress
    WORDPRESS_DB_PASSWORD: password
    WORDPRESS_DB_NAME: wordpress
```

### 2. Personal File Share

Use a file sharing service like FileBrowser:

```yaml
backend:
  image: filebrowser/filebrowser
  volumes:
    - ./files:/srv
```

### 3. Private API Service

Deploy a REST API that's only accessible via Tor:

```yaml
backend:
  build: ./my-api
  environment:
    DATABASE_URL: postgresql://...
```

### 4. Development Testing

Test how your application behaves behind Tor without deploying publicly:

```yaml
backend:
  image: my-app:dev
  volumes:
    - ./src:/app/src  # Live code reloading
```

---

## Frequently Asked Questions

### Is `tor_data/hidden_service/` auto-created?

**Yes!** You don't need to create this directory manually. Here's exactly what happens:

**First Run (directory doesn't exist):**
1. Docker Compose starts the Tor container
2. Tor reads the `torrc` configuration file
3. Tor sees `HiddenServiceDir /var/lib/tor/hidden_service/`
4. Since the directory doesn't exist, Tor automatically:
   - Creates the directory
   - Generates a new Ed25519 key pair (public + private keys)
   - Derives your .onion address from the public key
   - Writes `hostname`, `hs_ed25519_public_key`, and `hs_ed25519_secret_key` files
5. Tor publishes your hidden service descriptor to the Tor network
6. Your site becomes accessible at the `.onion` address

**Subsequent Runs (directory exists):**
1. Tor sees the `hidden_service/` directory already exists
2. Tor reads the existing keys
3. Tor uses the same `.onion` address (derived from the existing keys)
4. Your hidden service comes back online with the same address

### What if I delete `tor_data/hidden_service/`?

**Important:** If you delete this directory, you will get a **completely new .onion address** the next time you start Tor.

- Your old .onion address will be **permanently lost**
- There is **no way to recover** the old address without the original key files
- Any bookmarks, links, or references to your old .onion will stop working

This is why backing up `tor_data/hidden_service/` is crucial!

### Can I move my .onion address to a different server?

**Yes!** Just copy the `tor_data/hidden_service/` directory to the new server:

```bash
# On old server
tar -czf tor-identity-backup.tar.gz tor_data/hidden_service/

# Transfer to new server (use scp, sftp, or physical media)
scp tor-identity-backup.tar.gz user@newserver:/path/to/tor-hs/

# On new server
cd /path/to/tor-hs/
tar -xzf tor-identity-backup.tar.gz
docker-compose up -d
```

Your .onion address will work on the new server!

### Why is my .onion address so long?

Version 3 onion addresses are 56 characters long (e.g., `vww6ybal4bd7szmgncyruucpgfkqahzddi37ktceo3ah7ngmcopnpyyd.onion`). This is because they use stronger cryptography (Ed25519) compared to the old 16-character v2 addresses.

**Benefits of v3 addresses:**
- Much more secure
- Better encryption
- Resistant to enumeration attacks
- Cannot be practically brute-forced

### How long does it take to generate the .onion address?

**First generation:** 30-60 seconds
- This includes key generation and descriptor publication

**Subsequent starts:** 5-10 seconds
- Tor just reads existing keys and republishes the descriptor

### Do I need to create `tor_data/` manually?

**No.** Docker will create the `tor_data/` directory automatically when you run `docker-compose up -d` because it's specified as a volume mount. The internal `hidden_service/` subdirectory is then created by Tor itself.

**Directory creation flow:**
1. Docker creates `./tor_data/` (if it doesn't exist)
2. Tor creates `./tor_data/hidden_service/` (if it doesn't exist)
3. Tor creates the key files inside `hidden_service/`

---

## Useful Commands

```bash
# Start services
docker-compose up -d

# Stop services
docker-compose down

# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f tor

# Restart a service
docker-compose restart tor

# Get .onion address
docker exec tor cat /var/lib/tor/hidden_service/hostname

# Check Tor status with Nyx
docker exec -it tor nyx

# Access backend container
docker exec -it backend sh

# Update all images
docker-compose pull && docker-compose up -d

# Remove everything (including volumes)
docker-compose down -v
```

---

## Learning Resources

### Tor Documentation
- [Tor Project Homepage](https://www.torproject.org/)
- [Onion Services Best Practices](https://community.torproject.org/onion-services/setup/)
- [Tor Metrics](https://metrics.torproject.org/)

### Docker Resources
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Docker Networking](https://docs.docker.com/network/)

### nginx Resources
- [nginx Documentation](https://nginx.org/en/docs/)
- [nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

### Security Resources
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Security Headers](https://securityheaders.com/)

---

## Contributing

Found an issue or have a suggestion? Please open an issue or submit a pull request.

---

## License

This project is provided as-is for educational purposes. Use responsibly and in compliance with local laws.

---

## Disclaimer

This guide is for educational purposes. Running a Tor hidden service may be illegal in some jurisdictions. The authors are not responsible for how you use this software. Always comply with local laws and regulations.

**Remember:** With great privacy comes great responsibility. Don't use Tor for illegal activities.
