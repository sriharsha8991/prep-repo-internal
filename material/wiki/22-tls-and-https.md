# 22 — TLS / HTTPS for the Backend

> _Changed 2026-09-29: the `api.wellsynth.ai` certificate expired on
> 2026-09-08 because automatic renewal could never succeed. nginx now mounts
> `/etc/letsencrypt` whole plus an ACME webroot, and certificates are issued
> and renewed with `deploy/renew-ssl.sh`. Owns: how the public API gets and
> keeps a valid certificate, the DNS records it depends on, and how to roll
> nginx/TLS changes onto the prod VM._

The frontend is an **Azure Static Web App** served over HTTPS. Browsers only
let it call the backend if `https://api.wellsynth.ai` presents a **valid,
publicly trusted** certificate. An expired or self-signed certificate breaks
every API call from the frontend, even though the backend itself is healthy.
CORS is a separate concern (`CORS_ALLOW_ORIGINS`, see
[08-deployment-and-ops.md](08-deployment-and-ops.md)).

---

## Topology

```
Browser (Azure Static Web App)
        │  HTTPS
        ▼
GoDaddy DNS  api.wellsynth.ai  A → 135.235.219.241
        │
        ▼
Prod VM ── nginx container (ports 80/443, TLS terminates here)
             ├─ api.wellsynth.ai      → app:8080          (prod backend)
             └─ api-dev.wellsynth.ai  → nginx-dev:80      (dev stack, via the
                                                           external "edge" network;
                                                           block is commented out today)
```

Port 80 only redirects to HTTPS, except `/.well-known/acme-challenge/`, which
Let's Encrypt uses to verify that we control the domain.

## Where things live

| What | Where |
|---|---|
| Prod compose stack | **Portainer** stack #22, UI at `https://135.235.219.241:9443`. There is no compose folder on the host; `/data/compose/22` exists only inside the Portainer container. |
| nginx config used by prod | Host file `/home/azureuser/wellsynth-data/data/ngnix/nginx.conf` (bind-mounted into the container). **Not** the repo's `nginx.conf`, which is the template. |
| Certificate + key | `/etc/letsencrypt/live/api.wellsynth.ai/{fullchain,privkey}.pem` (symlinks into `/etc/letsencrypt/archive/…`) |
| Renewal settings | `/etc/letsencrypt/renewal/api.wellsynth.ai.conf` |
| ACME webroot | Host `/var/www/certbot` → container `/var/www/html` |
| certbot | `/usr/bin/certbot` on the VM |
| Issue/renew script | [`deploy/renew-ssl.sh`](../deploy/renew-ssl.sh). `deploy/` is gitignored, so the script is also reproduced in the [appendix](#appendix--deployrenew-sslsh). |

## DNS records (GoDaddy)

Let's Encrypt only needs each name to resolve to the VM. No CSR, TXT record,
or purchased certificate is involved.

| Type | Name | Value | Needed for |
|---|---|---|---|
| A | `api` | `135.235.219.241` | Prod API (exists) |
| A | `api-dev` | `135.235.219.241` | Dev API. Add this before putting `api-dev.wellsynth.ai` into the certificate. |

## How certificates are issued and renewed

1. `certbot certonly --webroot` writes a challenge file to `/var/www/certbot`.
2. nginx serves it at `http://<domain>/.well-known/acme-challenge/<file>`;
   Let's Encrypt fetches it and issues a 90-day certificate.
3. The `--deploy-hook` (`docker exec nginx nginx -s reload`) is saved in the
   renewal settings, so certbot's systemd timer renews the certificate about
   30 days before expiry and reloads nginx, with nobody involved.
4. Because the whole `/etc/letsencrypt` directory is mounted, nginx sees the
   new files as soon as it reloads.

## What broke in September 2026

The certificate issued on 2026-06-10 was never renewed and expired on
2026-09-08. Two setup mistakes guaranteed it:

- **No webroot mount.** `nginx.conf` served the challenge path from
  `/var/www/html`, but nothing was mounted there, so every challenge returned
  404 and renewal failed silently.
- **Single-file cert mounts.** The compose file mounted `live/fullchain.pem`
  and `live/privkey.pem` individually. Those are symlinks, and Docker pins the
  file they point to when the container starts. Even a successful renewal
  would have stayed invisible until the container was recreated.

Both are fixed in `docker-compose.yml` and `nginx.conf`:

```yaml
  nginx:
    volumes:
      - /home/azureuser/wellsynth-data/data/ngnix/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/www/certbot:/var/www/html:ro
```

```nginx
ssl_certificate     /etc/letsencrypt/live/api.wellsynth.ai/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/api.wellsynth.ai/privkey.pem;
```

---

## Runbook: roll out nginx / TLS changes to the prod VM

Pushing to `main` does **not** update the VM. The prod pipeline only builds
the backend image and bumps its tag in `docker-compose.yml`. nginx config and
volume changes have to be applied by hand, as below.

On the VM, your user (`wellsynthai`) needs `sudo` for `docker` and to read
`/home/azureuser`. `sudo cd` doesn't work; use `sudo -i` for a root shell.

### 1. Check the starting point (VM)

```bash
sudo docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}' | grep -E 'nginx|portainer'
```

```bash
sudo ls -l /etc/letsencrypt/live/api.wellsynth.ai/ /home/azureuser/wellsynth-data/data/ngnix/
```

```bash
which certbot
```

### 2. Update the host nginx config (VM)

Edit the live file in place rather than copying the repo's version over it.
The two can differ, and only the certificate paths need to change. `-i.bak`
keeps a backup next to the file.

```bash
sudo sed -i.bak 's#/etc/nginx/ssl/fullchain.pem#/etc/letsencrypt/live/api.wellsynth.ai/fullchain.pem#; s#/etc/nginx/ssl/privkey.pem#/etc/letsencrypt/live/api.wellsynth.ai/privkey.pem#' /home/azureuser/wellsynth-data/data/ngnix/nginx.conf
```

```bash
sudo grep -n "ssl_certificate\|acme-challenge\|root " /home/azureuser/wellsynth-data/data/ngnix/nginx.conf
```

Expect both `ssl_certificate` lines on `/etc/letsencrypt/live/…` and an
`acme-challenge` location with `root /var/www/html;`.

```bash
sudo mkdir -p /var/www/certbot
```

Go straight on to step 3. Until the stack is updated, the running container
lacks the new mounts, so a restart in between would crash nginx.

### 3. Update the Portainer stack

Open Portainer → **Stacks** → stack #22.

- **Git-backed stack** (shows a *Git repository* section): click **Pull and
  redeploy**.
- **Editor stack** (only an *Editor* tab): in the `nginx` service, replace the
  two `/etc/letsencrypt/live/…/*.pem` volume lines with
  `- /etc/letsencrypt:/etc/letsencrypt:ro` and
  `- /var/www/certbot:/var/www/html:ro`, then **Update the stack** with
  *Re-pull image* off so the backend is left alone. Future
  `docker-compose.yml` changes must also be pasted in by hand; switching the
  stack to Git-backed removes that step.

Verify:

```bash
sudo docker inspect nginx --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{println}}{{end}}'
```

```bash
sudo docker exec nginx nginx -T | grep ssl_certificate
```

If nginx keeps restarting, check `sudo docker logs nginx --tail 30`. To roll
back, restore `nginx.conf.bak` and revert the stack edit.

### 4. Copy the script to the VM (laptop, PowerShell)

`scp` runs on the laptop, where the file is, and asks for the VM password.

```bash
scp D:\Sriharsha\professional\Wellsynthai\deploy\renew-ssl.sh wellsynthai@135.235.219.241:~/
```

### 5. Issue / renew the certificate (VM)

```bash
sudo bash ~/renew-ssl.sh
```

The script:

1. writes a probe file to the webroot and fetches it over public HTTP for
   every domain, stopping early if nginx doesn't serve it (a failed real
   attempt would count against Let's Encrypt rate limits);
2. runs `certbot certonly --webroot … --keep-until-expiring` with the nginx
   reload saved as the deploy hook;
3. validates and reloads nginx, then prints the certificate's subject, issuer,
   expiry and names;
4. runs `certbot renew --dry-run` to prove unattended renewal works.

To add the dev hostname (only once its A record resolves):

```bash
sudo DOMAINS="api.wellsynth.ai api-dev.wellsynth.ai" bash ~/renew-ssl.sh
```

Then uncomment the `api-dev.wellsynth.ai` server blocks in the host
`nginx.conf` and reload nginx.

### 6. Confirm automatic renewal (VM)

```bash
systemctl list-timers | grep -i certbot
```

```bash
sudo grep -E "authenticator|webroot|renew_hook" /etc/letsencrypt/renewal/api.wellsynth.ai.conf
```

Expect a certbot timer, `authenticator = webroot`, and a `renew_hook` that
reloads nginx.

### 7. Confirm from outside (laptop)

```bash
echo | openssl s_client -connect api.wellsynth.ai:443 -servername api.wellsynth.ai 2>/dev/null | openssl x509 -noout -issuer -enddate
```

The issuer should be Let's Encrypt and the expiry about 90 days out. After a
renewal, `live/fullchain.pem` points at the next `archive/fullchainN.pem`.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `docker: not found` | You're in `sh`. Run `bash`. |
| `permission denied … docker.sock` | Prefix with `sudo`. |
| `cd: /data/compose/22: No such file or directory` | That path is inside the Portainer container. Update the stack in Portainer (step 3). |
| `cd: /home/azureuser: Permission denied` | Use `sudo` on the command, or `sudo -i`. |
| `scp: stat local "…": No such file` | `scp` was run on the VM. Run it on the laptop. |
| Script preflight fails | DNS doesn't point at the VM, or nginx lacks the webroot mount. Redo step 3 and check `docker inspect` mounts. |
| nginx restart loop after config change | Config points at `/etc/letsencrypt/…` but the container still has the old mounts. Finish step 3, or restore `nginx.conf.bak`. |
| Frontend still fails but certificate is valid | Check `CORS_ALLOW_ORIGINS` includes the Static Web App origin. |

## Things this page does not own

- Backend image build and tag bumps → `azure-pipelines.prod.yml`.
- Env vars, CORS and scaling knobs → [08-deployment-and-ops.md](08-deployment-and-ops.md).
- Other operational procedures → [10-runbooks.md](10-runbooks.md).

---

## Appendix — `deploy/renew-ssl.sh`

`deploy/` is gitignored, so this is the tracked copy. Keep it in sync with the
file.

```bash
#!/usr/bin/env bash
# Issue or renew the Let's Encrypt certificate served by the prod nginx.
#
# Run on the prod VM:   sudo bash deploy/renew-ssl.sh
#
# HTTP-01 webroot flow: certbot drops challenge files in /var/www/certbot, which
# docker-compose.yml mounts into the nginx container at /var/www/html, and
# nginx.conf serves /.well-known/acme-challenge/ from there. nginx keeps port 80
# the whole time. (The earlier setup had no webroot mount, so the automatic
# renewal could never answer the challenge and the cert expired on 2026-09-08.)
#
# Prerequisites (once):
#   1. GoDaddy DNS: A record for every name in DOMAINS -> the VM's public IP.
#   2. The VM's nginx.conf copy and docker-compose.yml match the repo, and nginx
#      was recreated with them (Portainer stack #22 -> update the stack).
#
# The --deploy-hook is saved into /etc/letsencrypt/renewal/<name>.conf, so the
# certbot timer/cron renews via the webroot and reloads nginx from now on.
#
# Env overrides:
#   DOMAINS="api.wellsynth.ai api-dev.wellsynth.ai"   (add api-dev only once its
#                                                       A record exists, or the
#                                                       whole issuance fails)
#   LETSENCRYPT_EMAIL=you@example.com                 (expiry notices; only used
#                                                       when registering an account)
set -euo pipefail

CERT_NAME="api.wellsynth.ai"
DOMAINS="${DOMAINS:-api.wellsynth.ai}"
WEBROOT="/var/www/certbot"
NGINX_CONTAINER="${NGINX_CONTAINER:-nginx}"
EMAIL="${LETSENCRYPT_EMAIL:-}"

if [ "$(id -u)" -ne 0 ]; then
  echo "Run with sudo (certbot writes to /etc/letsencrypt)." >&2
  exit 1
fi
command -v certbot >/dev/null || { echo "certbot not installed: sudo snap install --classic certbot" >&2; exit 1; }

domain_args=()
for d in $DOMAINS; do domain_args+=(-d "$d"); done

account_args=(--agree-tos --non-interactive)
if [ -n "$EMAIL" ]; then
  account_args+=(-m "$EMAIL")
elif [ -z "$(ls -A /etc/letsencrypt/accounts 2>/dev/null)" ]; then
  account_args+=(--register-unsafely-without-email)
fi

# --- 1. Preflight: prove nginx serves the webroot before asking Let's Encrypt.
# A failed real attempt counts against LE's rate limits; this doesn't.
mkdir -p "$WEBROOT/.well-known/acme-challenge"
probe="preflight-$$"
echo ok > "$WEBROOT/.well-known/acme-challenge/$probe"
trap 'rm -f "$WEBROOT/.well-known/acme-challenge/$probe"' EXIT
for d in $DOMAINS; do
  got="$(curl -s --max-time 10 "http://$d/.well-known/acme-challenge/$probe" || true)"
  if [ "$got" != "ok" ]; then
    echo "Preflight failed for $d: http://$d/.well-known/acme-challenge/ is not served from $WEBROOT." >&2
    echo "Check the GoDaddy A record for $d and that nginx was recreated with the new mounts (Portainer stack update)." >&2
    exit 1
  fi
  echo "Preflight ok: $d"
done

# --- 2. Issue / renew (no-op if the current cert has >30 days left).
certbot certonly --webroot -w "$WEBROOT" \
  --cert-name "$CERT_NAME" "${domain_args[@]}" \
  --keep-until-expiring "${account_args[@]}" \
  --deploy-hook "docker exec $NGINX_CONTAINER nginx -s reload"

# --- 3. Reload nginx and show what is being served.
docker exec "$NGINX_CONTAINER" nginx -t
docker exec "$NGINX_CONTAINER" nginx -s reload
openssl x509 -noout -subject -issuer -enddate -ext subjectAltName \
  -in "/etc/letsencrypt/live/$CERT_NAME/fullchain.pem"

# --- 4. Confirm unattended renewal works with the saved config.
certbot renew --dry-run --cert-name "$CERT_NAME"
