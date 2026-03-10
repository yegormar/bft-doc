# Build for Tomorrow – Production Deployment Guide

This guide covers deploying the BFT application on the **same server** as your existing Companion app: same Nginx, **different hostname** (`buildtomorrow.duckdns.org`). From the internet you use the **public port** (e.g. **51123**); the router forwards it to Nginx on the server (e.g. **8443**). Companion keeps its hostname; BFT uses buildtomorrow.duckdns.org and its own SSL certificate.

## Prerequisites

- Node.js 18+ (for bft-api and building bft-ui)
- Nginx (already installed)
- acme.sh (already installed)
- LLM backend (Ollama or Ollama Cloud) reachable from the server

## 1. Hostname and port

- **Hostname:** `buildtomorrow.duckdns.org` (Duck DNS). Ensure the Duck DNS subdomain points to your server’s public IP (via your Duck DNS dashboard).
- **Public port:** The port open on your **router** (e.g. **51123**). Users and the browser use this in the URL: `https://buildtomorrow.duckdns.org:51123`. The router forwards 51123 → Nginx on the server.
- **Nginx listen:** On the server, Nginx listens on **8443** (same as Companion). You never put 8443 in the browser or in `VITE_API_BASE_URL` / `CORS_ORIGIN`—those must use the **public** port (51123). Nginx distinguishes apps by `server_name`.

## 2. ACME certificate with Duck DNS (acme.sh)

Use acme.sh’s **Duck DNS** plugin for DNS-01 validation. This works without opening port 80.

### 2.1 Get your Duck DNS token

In [Duck DNS](https://www.duckdns.org), sign in → your subdomain → copy the **token**. Store it securely; **never commit it**.

### 2.2 Issue the certificate

Export the token and run acme.sh with the `dns_duckdns` plugin:

```bash
# Export token (replace with your actual token from Duck DNS)
export DuckDNS_Token="YOUR_DUCKDNS_TOKEN"

# Staging first (avoids Let's Encrypt rate limits)
~/.acme.sh/acme.sh --issue -d buildtomorrow.duckdns.org --dns dns_duckdns --staging

# Production (after staging succeeds)
~/.acme.sh/acme.sh --issue -d buildtomorrow.duckdns.org --dns dns_duckdns
```

acme.sh saves the token for renewals, so you only need to export it for this session.

**Security:** Never commit the Duck DNS token. If it was exposed (e.g. in logs or chat), regenerate it in the Duck DNS dashboard and re-issue the cert.

### 2.3 Certificate paths after issue

acme.sh stores certs under `~/.acme.sh/`. For ECC certs (typical):

- Certificate: `~/.acme.sh/buildtomorrow.duckdns.org_ecc/fullchain.cer`
- Private key: `~/.acme.sh/buildtomorrow.duckdns.org_ecc/buildtomorrow.duckdns.org.key`

Use these paths in the Nginx config below.

### 2.4 Renewal

acme.sh cron handles renewal. The Duck DNS token is stored in its account config. To force a renewal:

```bash
~/.acme.sh/acme.sh --renew -d buildtomorrow.duckdns.org --force
```

---

## 3. Directory structure on the server

Deploy BFT under a dedicated directory, e.g.:

```
/opt/bft/
├── bft_ui/          # Built React app (contents of bft-ui/dist)
├── bft_api/         # Node.js backend (clone or rsync of bft-api)
│   ├── src/
│   │   └── scripts/
│   ├── config/
│   ├── conf/
│   ├── server.js
│   ├── app.js
│   ├── package.json
│   └── .env         # Production .env (create from env.example)
```

Create the base directory (both `bft_ui` and `bft_api` will be empty until you deploy in steps 4 and 5):

```bash
sudo mkdir -p /opt/bft/bft_ui /opt/bft/bft_api
# If you use a dedicated user (e.g. bft):
# sudo useradd -r -s /bin/false bft  # optional
# sudo chown -R bft:bft /opt/bft      # optional
```

---

## 4. Build frontend (bft-ui)

### 4.1 Production env file

In **bft-ui** on your build machine, create `.env.production` (used by Vite for `npm run build`). Use the **public** BFT URL: the port must be the one open on your **router** (e.g. **51123**), not the port Nginx listens on on the server (8443). No trailing slash.

```bash
cd bft-ui

cat > .env.production << 'EOF'
# Production API base URL (origin; /api is appended by the app)
# Use the PUBLIC port (router), e.g. 51123—not the server’s Nginx listen port (8443)
VITE_API_BASE_URL=https://buildtomorrow.duckdns.org:51123
VITE_APP_NAME=Build for Tomorrow
EOF
```

- If your router exposes standard HTTPS port 443, omit the port: `https://buildtomorrow.duckdns.org`
- Do **not** put secrets here; this file is baked into the client bundle.

### 4.2 Build and deploy

```bash
cd bft-ui
npm ci
npm run build
# Deploy dist contents to server
rsync -a dist/ user@your-server:/opt/bft/bft_ui/
```

---

## 5. Backend configuration (bft-api)

The directory `/opt/bft/bft_api/` on the server is empty until you copy the project there. Do the copy from your **dev machine**, then install dependencies on the **server**.

### 5.1 Copy project and install

**Step A – On your dev machine** (in the repo root, the folder that contains `bft-api/`):

Copy the backend code to the server. Replace `root@YOUR_SERVER` with your server user and host (e.g. `root@192.168.1.10` or `root@buildtomorrow.duckdns.org`).

```bash
rsync -a --exclude node_modules --exclude .env bft-api/ root@YOUR_SERVER:/opt/bft/bft_api/
```

This fills `/opt/bft/bft_api/` on the server (excluding local `node_modules` and `.env`; you will create `.env` on the server in 5.2).

**Step B – On the server** (SSH in, then). **Required:** this installs dependencies (e.g. `mathjs`, `express`). Without it you get `Cannot find module 'mathjs'` or similar.

```bash
cd /opt/bft/bft_api
npm ci --omit=dev
```

After any deploy that updates `package.json` or replaces files in `/opt/bft/bft_api/`, run `npm ci --omit=dev` again in that directory, then restart the service.

### 5.2 Production .env

Create `/opt/bft/bft_api/.env` from `env.example`. All values are required unless marked optional in `env.example`. Example:

```bash
# Server – use a port not used by Companion (e.g. Companion=8080, BFT=8081)
PORT=8081
NODE_ENV=production
# Must match the public origin: use the PUBLIC port (e.g. 51123), not Nginx’s listen port
CORS_ORIGIN=https://buildtomorrow.duckdns.org:51123

# LLM (ollama or ollama_cloud)
LLM_PROVIDER=ollama
LLM_BASE_URL=http://ubuntu-llm:11434
LLM_MODEL=llama3:latest
LLM_API_KEY=your-key-if-using-ollama-cloud
LLM_WEB_SEARCH=true
LLM_TEMPERATURE=0.5
LLM_MAX_TOKENS=32768
LLM_TOP_P=0.9

# Prompts (paths relative to project root)
LLM_SYSTEM_PROMPT_FILE=conf/assessment_system_prompt.txt
LLM_REPORT_PROFILE_SYSTEM_PROMPT_FILE=conf/report_profile_system_prompt.txt
LLM_REPORT_HYBRID_SYSTEM_PROMPT_FILE=conf/report_hybrid_system_prompt.txt
# Careers on the Recommendations page: set this to enable career directions and directions-to-avoid
LLM_REPORT_RECOMMENDATIONS_SYSTEM_PROMPT_FILE=conf/report_recommendations_system_prompt.txt
```

- **PORT:** Must be free on the server (e.g. **8081**). Nginx will proxy to `http://127.0.0.1:8081`.
- **CORS_ORIGIN:** Exactly the origin the browser uses—same as `VITE_API_BASE_URL` (public port, e.g. 51123).
- **Careers:** If `LLM_REPORT_RECOMMENDATIONS_SYSTEM_PROMPT_FILE` is not set, the Careers/Recommendations page will show “careers will appear once generated.” Set it as above (the file exists in `conf/`), restart the API, then refresh the Results page or run a new discovery so the report is regenerated with career recommendations.

---

## 6. Nginx configuration

Add a **new** `server` block for BFT. Same `listen` as Companion (`8443 ssl` on the server); the router forwards the public port (e.g. 51123) to 8443. Different `server_name` so Nginx routes by hostname.

Example (adjust paths and cert if needed):

```nginx
# Build for Tomorrow – same port as Companion, different hostname
server {
    listen 8443 ssl;
    server_name buildtomorrow.duckdns.org;

    root /opt/bft/bft_ui;
    index index.html;

    ssl_certificate     /root/.acme.sh/buildtomorrow.duckdns.org_ecc/fullchain.cer;
    ssl_certificate_key /root/.acme.sh/buildtomorrow.duckdns.org_ecc/buildtomorrow.duckdns.org.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 10M;

    # SPA routing
    location / {
        try_files $uri /index.html;
    }

    # BFT API – do NOT strip /api; backend is mounted at /api
    location /api/ {
        proxy_pass http://127.0.0.1:8081;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
        proxy_connect_timeout 10s;
        proxy_send_timeout 120s;
    }
}
```

- **Important:** `proxy_pass http://127.0.0.1:8081;` has **no** trailing slash so that `/api/...` is passed to the backend as `/api/...` (bft-api mounts routes at `/api`).
- Reload Nginx after changes:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 7. Run backend as a systemd service

Create `/etc/systemd/system/bft-api.service`:

```ini
[Unit]
Description=Build for Tomorrow API
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/bft/bft_api
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

If you use a dedicated user (e.g. `bft`), set `User=bft` and ensure that user owns `/opt/bft/bft_api` and can read `.env` and `conf/`.

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable bft-api
sudo systemctl start bft-api
sudo systemctl status bft-api
```

---

## 8. Checklist and testing

- [ ] DNS: `buildtomorrow.duckdns.org` points to the server (Duck DNS updated).
- [ ] Certificate: Issued and paths in Nginx match.
- [ ] `.env.production`: `VITE_API_BASE_URL` and `VITE_APP_NAME` set; build done and `dist/` deployed to `/opt/bft/bft_ui/`.
- [ ] Backend: `.env` in `/opt/bft/bft_api/` with correct `PORT`, `CORS_ORIGIN`, and LLM settings; `bft-api` service running.
- [ ] Nginx: New server block for BFT hostname (listen 8443); `nginx -t` and reload.

Test (use the **public** port, e.g. 51123):

```bash
# UI
curl -I -k "https://buildtomorrow.duckdns.org:51123/"

# API (e.g. health or sessions)
curl -k "https://buildtomorrow.duckdns.org:51123/api/sessions" -X POST -H "Content-Type: application/json" -d '{}'
```

In the browser: open `https://buildtomorrow.duckdns.org:51123`, complete a short flow, and check the console for errors.

---

## 9. Logs and updates

- **API:** `sudo journalctl -u bft-api -f`
- **Nginx:** `sudo tail -f /var/log/nginx/access.log /var/log/nginx/error.log`

**Updates:**

- **UI:** Rebuild with `.env.production`, then `rsync -a dist/ user@server:/opt/bft/bft_ui/`.
- **API:** Rsync code (keep `--exclude node_modules`), then **on the server** run `cd /opt/bft/bft_api && npm ci --omit=dev`, then `sudo systemctl restart bft-api`.

**Troubleshooting: `Cannot find module 'mathjs'` (or similar)**  
Dependencies were not installed in the API directory. On the server run:

```bash
cd /opt/bft/bft_api
npm ci --omit=dev
sudo systemctl restart bft-api
```

---

## 10. Summary: same ports, different hostname

| App        | Hostname                  | Public URL port (router) | Nginx listen (server) | Backend local port |
|-----------|----------------------------|--------------------------|------------------------|---------------------|
| Companion | companion.legacyavatar.ai   | 51123                    | 8443 ssl               | 8080                |
| BFT       | buildtomorrow.duckdns.org  | 51123                    | 8443 ssl               | 8081                |

Only the **public port** (e.g. 51123) is open on the router; it is forwarded to Nginx on 8443. Use **51123** in browser URLs, `VITE_API_BASE_URL`, and `CORS_ORIGIN`. Nginx chooses the `server` block by `server_name`. Each app has its own hostname and ACME certificate.
