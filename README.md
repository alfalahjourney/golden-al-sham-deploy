# Golden Al Sham — Deploy

Orchestration for the Golden Al Sham stack (Express backend + React frontend +
nginx reverse proxy), wired together with Docker Compose.

This repo tracks **only the orchestration files**. The two applications live in
their own repos and are cloned as siblings next to `docker-compose.yml`:

```
golden-al-sham/
  docker-compose.yml          # this repo
  nginx/nginx.conf            # this repo — reverse proxy (path-based routing)
  .env.example                # this repo
  .env                        # you create from .env.example (gitignored)
  Golden-Al-Sham-Backend/     # github.com/alfalahjourney/Golden-Al-Sham-Backend
  Golden-Al-Sham-frontend/    # github.com/alfalahjourney/Golden-Al-Sham-frontend
```

## Architecture

Everything is served from **one origin** (`http://<IP>` now, `https://<domain>`
later). nginx routes by URL path:

| Path | → |
|------|---|
| `/admin`, `/api`, `/auth`, `/getUser`, `/user`, `/socket.io` | backend (Express :5000) |
| everything else | frontend (React SPA) |

Same-origin means no CORS preflight and no subdomain/DNS needed for the API.

## Deploy on a fresh Ubuntu server

Prereqs: Docker + Docker Compose v2 installed, ports 22 and 80 open in the
EC2 security group, an Elastic IP attached.

```bash
# 1. Clone all three repos side by side
mkdir -p ~/golden-al-sham && cd ~/golden-al-sham
git clone https://<TOKEN>@github.com/alfalahjourney/golden-al-sham-deploy.git .
git clone https://<TOKEN>@github.com/alfalahjourney/Golden-Al-Sham-Backend.git
git clone https://<TOKEN>@github.com/alfalahjourney/Golden-Al-Sham-frontend.git

# 2. Root .env — set your public address
cp .env.example .env
#   edit .env: replace YOUR_ELASTIC_IP with the real Elastic IP

# 3. Backend secrets — copy from a secure source to:
#   Golden-Al-Sham-Backend/.env   (MONGO_URL, JWT_SECRET, Cloudinary, SMTP, ADMIN_*)

# 4. Build & launch
docker compose up -d --build
docker compose ps         # all three "Up"; backend/frontend healthy
```

Verify: `curl -i http://localhost/api/health` (backend) and open
`http://<Elastic-IP>` in a browser.

## Updating

```bash
cd ~/golden-al-sham
git pull                                   # deploy repo (compose/nginx)
( cd Golden-Al-Sham-Backend  && git pull ) # backend code
( cd Golden-Al-Sham-frontend && git pull ) # frontend code
docker compose up -d --build
```

## Changing the public address (new IP, or adding a domain)

The frontend bakes the address in at build time, so after editing `.env`:

```bash
docker compose up -d --build client
```

For a domain, also set `server_name` in `nginx/nginx.conf` and add a
`listen 443 ssl;` block via Certbot — see below.

## Enabling HTTPS (domain + Let's Encrypt)

`nginx.conf` already serves HTTPS for **goldenalsham.com**, but nginx won't
start until the certificate exists. Obtain it once with the bootstrap config,
then switch to the real one.

Prereqs: DNS A records for `goldenalsham.com` and `www` point at the Elastic
IP (`dig +short goldenalsham.com` returns it), and ports **80 + 443** are open
in the security group.

```bash
cd ~/golden-al-sham
mkdir -p certbot/conf certbot/www

# 1. Start on plain HTTP with the ACME challenge served (bootstrap nginx)
docker compose -f docker-compose.yml -f docker-compose.bootstrap.yml up -d --build

# 2. Obtain the certificate (edit the email)
docker run --rm \
  -v ~/golden-al-sham/certbot/conf:/etc/letsencrypt \
  -v ~/golden-al-sham/certbot/www:/var/www/certbot \
  certbot/certbot certonly --webroot -w /var/www/certbot \
  -d goldenalsham.com -d www.goldenalsham.com \
  --email you@example.com --agree-tos --no-eff-email

# 3. Set .env for HTTPS
#   VITE_API_BASE_URL=https://goldenalsham.com
#   APP_ORIGIN=https://goldenalsham.com
#   COOKIE_SECURE=true

# 4. Switch to the real HTTPS nginx + rebuild the frontend for the new URL
docker compose up -d --build
```

Open https://goldenalsham.com — padlock should show and http:// redirects.

### Auto-renewal (certs expire every 90 days)

```bash
crontab -e
```
```
0 3 * * * docker run --rm -v ~/golden-al-sham/certbot/conf:/etc/letsencrypt -v ~/golden-al-sham/certbot/www:/var/www/certbot certbot/certbot renew --quiet && docker compose -f ~/golden-al-sham/docker-compose.yml exec -T nginx nginx -s reload
```
