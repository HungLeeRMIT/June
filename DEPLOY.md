# Deploy to Hostinger — `rmithanoitheatre.shop`

This app is a **Node.js + SQLite** service, so it needs a **VPS** (not shared hosting). Hostinger's VPS plans (KVM 1 upwards) are fine — any Linux box with root access works the same way.

Architecture on the box:

```
  rmithanoitheatre.shop   (DNS A → VPS IP)
           │ 443 / 80
           ▼
      ┌─────────┐   HTTP    ┌──────────────┐
      │  Nginx  │ ────────▶ │ Node (PM2)   │ ──▶  theatre.db (SQLite, WAL)
      │  (TLS)  │  127.0.0  │  server.js   │
      └─────────┘  .1:3000  └──────────────┘
```

- Nginx terminates TLS on `rmithanoitheatre.shop` and reverse-proxies to Node on `127.0.0.1:3000`.
- PM2 keeps Node alive, restarts on crashes, and starts on boot.
- Certbot provisions and auto-renews a Let's Encrypt cert for `rmithanoitheatre.shop` + `www.rmithanoitheatre.shop`.
- A cron job backs up the SQLite DB nightly (`deploy/backup.sh`).

---

## 1. Point DNS at your VPS

In your Hostinger hPanel → **Domains → `rmithanoitheatre.shop` → DNS Zone Editor**, set:

| Type | Name  | Points to           | TTL |
|------|-------|---------------------|-----|
| A    | `@`   | `<your VPS IPv4>`   | 300 |
| A    | `www` | `<your VPS IPv4>`   | 300 |

Remove any conflicting `A` / `AAAA` / `CNAME` records for `@` and `www` that Hostinger may have created automatically (e.g. redirects to their park page).

Wait for DNS to resolve:

```bash
dig rmithanoitheatre.shop +short         # should print the VPS IPv4
dig www.rmithanoitheatre.shop +short     # same
```

Don't move on until both resolve — Certbot will fail otherwise.

## 2. One-time VPS setup

SSH into the VPS (`ssh root@<ip>`), then create a non-root user (recommended):

```bash
adduser theatre
usermod -aG sudo theatre
rsync --archive --chown=theatre:theatre ~/.ssh /home/theatre     # copy SSH key
su - theatre
```

Install dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git nginx ufw sqlite3 build-essential

# Node 20 LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# PM2 process manager
sudo npm i -g pm2
```

Firewall (only expose SSH + HTTP/S):

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw --force enable
```

## 3. Clone the app

```bash
sudo mkdir -p /var/www && sudo chown theatre:theatre /var/www
cd /var/www
git clone <your-repo-url> theatre         # or rsync from your laptop
cd theatre
```

If you're not using git, rsync from your laptop:

```bash
rsync -avz --exclude node_modules --exclude .env --exclude '*.db*' \
    ./ theatre@<vps-ip>:/var/www/theatre/
```

## 4. Configure secrets

```bash
cp .env.example .env
nano .env
```

Fill in, at minimum:

```ini
NODE_ENV=production
PORT=3000
BIND_HOST=127.0.0.1
TRUST_PROXY=1

GMAIL_USER=rmithanoitheatrclub@gmail.com
GMAIL_APP_PASSWORD=xxxxxxxxxxxxxxxx     # 16-char Gmail App Password

ADMIN_USERNAME=admin
ADMIN_PASSWORD=<generate-a-strong-one>  # e.g. openssl rand -base64 24
```

Lock the file:

```bash
chmod 600 .env
```

## 5. First launch under PM2

```bash
./deploy/deploy.sh first-run
```

That does: `npm ci --omit=dev` → `pm2 start ecosystem.config.js --env production` → `pm2 save`.

Smoke-test it's alive:

```bash
pm2 status theatre
pm2 logs theatre --lines 30
curl -I http://127.0.0.1:3000/           # → 200 OK
```

Enable auto-start on reboot — PM2 will print the exact command; run the `sudo` line it gives you:

```bash
pm2 startup systemd -u $USER --hp $HOME
# copy & run the suggested `sudo env PATH=... pm2 startup ...` line
pm2 save
```

## 6. Nginx + HTTPS for rmithanoitheatre.shop

The template already names your domain, so you can symlink it straight in:

```bash
sudo cp deploy/nginx.conf.example /etc/nginx/sites-available/theatre
sudo ln -sf /etc/nginx/sites-available/theatre /etc/nginx/sites-enabled/theatre
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Sanity-check plain HTTP now reaches the app through Nginx:

```bash
curl -I http://rmithanoitheatre.shop/    # → 200 OK
```

Provision a Let's Encrypt cert (auto-renews every 90 days):

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx \
    -d rmithanoitheatre.shop \
    -d www.rmithanoitheatre.shop \
    --agree-tos --email you@example.com --redirect --non-interactive
```

Certbot rewrites `/etc/nginx/sites-available/theatre` to:
- keep port 80 as a 301 redirect to HTTPS
- add a `server { listen 443 ssl; ... }` block with the issued cert

Confirm renewal is scheduled:

```bash
sudo systemctl list-timers | grep certbot     # certbot.timer should be listed
sudo certbot renew --dry-run                  # exercise the renewal path
```

Open `https://rmithanoitheatre.shop` in a browser — you should see the booking page.  `https://rmithanoitheatre.shop/admin` is the admin panel.

## 7. Nightly DB backup (optional but recommended)

The SQLite DB is the only stateful thing on disk. Back it up:

```bash
crontab -e
# add:
0 3 * * *  /var/www/theatre/deploy/backup.sh >> /var/www/theatre/logs/backup.log 2>&1
```

Backups land in `/var/www/theatre/backups/theatre-YYYYMMDD-HHMMSS.db.gz`, keeping the last 14.

For off-site copies, add a second cron that ships the `backups/` folder to Hostinger object storage / Google Drive / S3 via `rclone`.

## 8. Subsequent deploys

Any time you push new code:

```bash
# on the server
cd /var/www/theatre
./deploy/deploy.sh           # git pull + npm ci + pm2 reload (zero-downtime)
```

---

## Alternative: Hostinger "Node.js hosting" (shared / Business plan)

If you bought a **Hostinger Cloud / Business** plan with Node.js support instead of a VPS:

1. In hPanel go to **Websites → `rmithanoitheatre.shop` → Advanced → Node.js**.
2. Create a new Node app:
   - **App mode:** Production
   - **Node version:** 20.x
   - **Application URL:** `rmithanoitheatre.shop`
   - **Application root:** `public_html/theatre` (or wherever you uploaded)
   - **Application startup file:** `server.js`
3. Upload these files via File Manager / SFTP:
   - `server.js`, `package.json`, `package-lock.json`, `club.html`, `admin.html`, `automatic_mail.md`, `Screenshot 2026-04-17 at 23.54.39.png`, `.env`
   - **Skip** `node_modules/` — Hostinger runs `npm install` for you.
4. Click **Run NPM Install** in the panel.
5. Set environment variables (the panel has a UI — take them from `.env.example`). Set `TRUST_PROXY=1` — Hostinger's fronting proxy is one hop.
6. Start / restart from the panel.
7. Enable **Let's Encrypt SSL** from the **SSL** tab of hPanel — Hostinger handles the cert for you.

Caveats for this tier:

- Passenger may evict idle processes. SQLite + WAL survives eviction, but in-memory admin sessions reset on restart — admins have to re-login.
- Long-lived processes like PM2 aren't used; Hostinger manages the lifecycle.

For anything heavier than a small show, the VPS path above is more predictable.

---

## Troubleshooting

| Symptom                                               | Fix |
|-------------------------------------------------------|-----|
| `502 Bad Gateway` from Nginx                          | `pm2 status theatre` — if offline: `pm2 logs theatre` and check for crash. Also `sudo systemctl status nginx`. |
| Admin cookie set but login doesn't stick              | Make sure you're on **HTTPS** — `Secure` cookies are rejected on plain HTTP in prod. For local debugging set `SECURE_COOKIES=false`. |
| Certbot: "DNS problem: NXDOMAIN"                      | DNS hasn't propagated yet. `dig rmithanoitheatre.shop +short` should return the VPS IP before you run certbot. Wait, then retry. |
| `req.ip` always shows `::ffff:127.0.0.1`              | `TRUST_PROXY` is too low. Behind one Nginx set `TRUST_PROXY=1`; behind Cloudflare + Nginx set `=2`. |
| `better-sqlite3` build fails on install               | `sudo apt install -y build-essential python3` then `rm -rf node_modules && npm ci --omit=dev`. |
| DB corruption after hard reboot                       | Unlikely with WAL, but restore from the latest `backups/*.db.gz` via `gunzip` + `sqlite3 theatre.db ".restore backup.db"`. |
| Gmail rejects mail                                    | You must use a 16-char **App Password** (not your normal Google password). 2-Step Verification must be enabled on the Google account first. |
| `www.rmithanoitheatre.shop` works but apex doesn't    | Missing apex `A` record at `@`. Add it in hPanel DNS, wait a few minutes, re-run certbot. |
