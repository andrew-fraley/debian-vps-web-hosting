# Deployment: Hugo Sites and Lead Form Backend on Debian + Caddy

Procedure for deploying a Hugo site to a domain hosted on the VPS, connecting it to the lead form backend, and updating the backend. Build locally, copy with rsync, serve with Caddy.

Placeholders: `<domain>` (for example `example.com`), `<api-host>` (the shared API hostname, for example `api.example.com`), `<adminuser>`, `<VPS-IP>`, `ionos-vps` (SSH host alias).

## Prerequisites

- VPS with Debian 13, Caddy installed, firewalls allowing 22, 80, 443/tcp and 443/udp.
- DNS A record for `<domain>` and `www` pointing at the VPS (Cloudflare, DNS only).
- SSH alias in `~/.ssh/config`:
  ```
  Host ionos-vps
      HostName <VPS-IP>
      User <adminuser>
      IdentityFile ~/.ssh/ionos_vps
  ```
- `rsync` installed locally and on the server.
- Hugo installed locally.

## One-Time Setup per Site

Directory convention: `/var/www/<domain>/`, owned by `<adminuser>:www-data`, mode 755 (directories) and 644 (files). Caddy gets read access only.

```bash
sudo mkdir -p /var/www/<domain>
sudo chown <adminuser>:www-data /var/www/<domain>
sudo chmod 755 /var/www/<domain>
```

Caddyfile (`/etc/caddy/Caddyfile`):

```caddy
<domain> {
    root * /var/www/<domain>
    file_server
    encode zstd gzip
    log
}

www.<domain> {
    redir https://<domain>{uri} permanent
}
```

Apply:

```bash
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl reload caddy
```

Caddy requests and renews the Let's Encrypt certificate automatically. Do not reload repeatedly if issuance fails; check the logs first (Let's Encrypt rate limits failed attempts).

## Deploy (Every Update)

Run from the Hugo site directory on the local machine.

1. Build:
   ```bash
   hugo --minify --baseURL "https://<domain>/"
   ```
2. Review `public/`. Everything in it becomes public. Check for drafts, secrets and private notes.
3. Dry run (`-n` lists changes without applying them):
   ```bash
   rsync -avzn --delete --chmod=D755,F644 public/ ionos-vps:/var/www/<domain>/
   ```
4. Deploy:
   ```bash
   rsync -avz --delete --chmod=D755,F644 public/ ionos-vps:/var/www/<domain>/
   ```

Notes:
- The trailing slash on `public/` is required. It copies the contents, not the folder.
- `--delete` removes remote files that are not in the build.
- `--chmod` enforces the permission model regardless of local umask.

## Verify

```bash
# file counts match
find public -type f | wc -l
ssh ionos-vps 'find /var/www/<domain> -type f | wc -l'

# Caddy can read the files
ssh ionos-vps 'sudo -u caddy cat /var/www/<domain>/index.html | head -5'

# HTTP redirects to HTTPS
curl -sI http://<domain>

# HTTPS serves the site
curl -sI https://<domain>

# www redirects to the bare domain
curl -sI https://www.<domain>/
```

Expected: matching counts, `308` on HTTP, `HTTP/2 200` with `server: Caddy` on HTTPS, `301` to `https://<domain>/` on www. Rerunning the rsync should transfer almost nothing.

## Connect the Lead Form (Per Site)

Every site posts to the one shared API hostname. The backend tells sites apart by their `site_key`.

1. In the Hugo project, set the site key in `hugo.toml` and point the form at `https://<api-host>/leads`. Rebuild and deploy.
2. Register the site in the backend database (on the VPS). The script prompts for the site key, site name, domain and the notification email:
   ```bash
   cd /opt/lead-form-backend
   sudo -u leadform env DATA_DIR=/var/lib/lead-form-backend \
     RESEND_API_KEY=unused EMAIL_FROM=unused@example.com \
     .venv/bin/python -m scripts.create_site
   ```
3. Add the site's origin to `CORS_ORIGINS` in `/etc/lead-form-backend/env` (comma-separated list, for example `https://example.com,https://example.org`), then restart the backend:
   ```bash
   sudo vim /etc/lead-form-backend/env
   sudo systemctl restart lead-form-backend
   ```
4. Verify the browser preflight from the local machine. The response must include `access-control-allow-origin` with the site's origin:
   ```bash
   curl -sSi -X OPTIONS https://<api-host>/leads \
     -H 'Origin: https://<domain>' \
     -H 'Access-Control-Request-Method: POST' \
     -H 'Access-Control-Request-Headers: content-type' | head -12
   ```

Never paste the contents of the env file anywhere. To list only the variable names: `sudo cut -d= -f1 /etc/lead-form-backend/env`.

## Add Another Site to the Same Server

Repeat "One-Time Setup per Site" for the new domain. Append the new site block and its `www` block to the end of `/etc/caddy/Caddyfile` and leave the existing blocks alone. Then run the same `caddy fmt`, `caddy validate` and `systemctl reload caddy` commands. Add the apex and `www` DNS records first, because Caddy requests the certificate on reload.

## Update the Backend

```bash
cd /opt/lead-form-backend
git pull
.venv/bin/pip install -r requirements.txt
.venv/bin/pip check
sudo systemctl restart lead-form-backend
curl -sSi https://<api-host>/ | head -1
```

Expect `HTTP/2 200` and `pip check` reporting no broken requirements. The server holds a read-only clone, so only pull on the server and make changes in the backend repository. Take a database backup before updating if the change touches the database.

## Roll Back

Not yet tested. Record the result here after the first real rollback.

Backend (on the VPS):

```bash
cd /opt/lead-form-backend
git log --oneline -5
git checkout <commit>
sudo systemctl restart lead-form-backend
curl -sSi https://<api-host>/ | head -1
```

To return to the latest version, run `git checkout main` and restart the service. Site: check out the previous commit in the Hugo project, rebuild, and run the deploy steps again.

## Troubleshooting

| Symptom | Check |
|---|---|
| 403 Forbidden | `namei -l /var/www/<domain>/index.html` for a parent directory Caddy cannot enter |
| Site files nested under `public/` | Missing trailing slash on the rsync source |
| Wrong links or canonical URLs | `--baseURL` does not match the public address |
| Certificate not issued | `sudo journalctl -u caddy --no-pager -o cat \| grep -E '"logger":"(tls\|acme)'`; confirm DNS and ports 80/443 |
| Config change not applied | Run `caddy validate`, then `systemctl reload caddy` |
| Browser blocks the form submission | The site origin is missing from `CORS_ORIGINS`, or the service was not restarted after the change |
| API returns 403 | Unknown or inactive `site_key`. Run the create site script and check the key in `hugo.toml` |
| API returns 502 | The backend is not running. Check `systemctl status lead-form-backend` and the journal |
