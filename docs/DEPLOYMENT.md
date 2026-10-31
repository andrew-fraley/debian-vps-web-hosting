# Deployment: Hugo static site to Debian + Caddy

Procedure for deploying a Hugo site to a domain hosted on the VPS. Build locally, copy with rsync, serve with Caddy.

Placeholders: `<domain>` (for example `example.com`), `<adminuser>`, `ionos-vps` (SSH host alias).

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

## One-time setup per site

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

## Deploy (every update)

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

## Troubleshooting

| Symptom | Check |
|---|---|
| 403 Forbidden | `namei -l /var/www/<domain>/index.html` for a parent directory Caddy cannot enter |
| Site files nested under `public/` | Missing trailing slash on the rsync source |
| Wrong links or canonical URLs | `--baseURL` does not match the public address |
| Certificate not issued | `sudo journalctl -u caddy --no-pager -o cat \| grep -E '"logger":"(tls\|acme)'`; confirm DNS and ports 80/443 |
| Config change not applied | Run `caddy validate`, then `systemctl reload caddy` |

