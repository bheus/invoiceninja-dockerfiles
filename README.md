# Debian Docker for [Invoice Ninja](https://www.invoiceninja.com/) — personal fork

Fork of [invoiceninja/dockerfiles](https://github.com/invoiceninja/dockerfiles) (`debian` branch), adapted for self-hosted deployment via Portainer.

Differences from upstream:

- **Locally built images.** Both `app` and `nginx` build from `debian/Dockerfile` and `debian/Dockerfile.nginx`; nothing is pulled from Docker Hub. Update with `docker compose build`, not `docker compose pull`.
- **Nginx is a separate service** (not bundled into the app image). Exposed on host port `8012`.
- **`stack.env` lives one directory above `debian/docker-compose.yml`** (i.e. at the repo root) and is not committed. See `stack.env.example` for the schema.
- **MySQL tuning + slow-query log** baked into the compose file (see below).

## Features

- NGINX as a separate service (own image, own Dockerfile)
- Built-in Chrome for PDF generation
- Saxon XSLT 2 engine
- OPcache
- Multi-language support

## Get started

```bash
git clone git@github.com:bheus/invoiceninja-dockerfiles.git
cd invoiceninja-dockerfiles
cp stack.env.example stack.env
# edit stack.env — at minimum: APP_KEY, IN_USER_EMAIL, IN_PASSWORD, DB_*
cd debian
docker compose build
docker compose up -d
```

Nginx is then reachable at `http://localhost:8012/`. Adjust `APP_URL` in `stack.env` to match the public URL you front this with.

## Initial account setup

### Primary account

Before first start, set `IN_USER_EMAIL` and `IN_PASSWORD` in `stack.env`. These bootstrap the primary account on first run and can be removed afterwards.

> ⚠️ If `IN_USER_EMAIL` / `IN_PASSWORD` are unset, the defaults are `admin@example.com` / `changeme!`.

### Generate an APP_KEY

```bash
# Before containers exist:
docker run --rm -it invoiceninja-debian:local php artisan key:generate --show

# Or with containers already running:
docker compose exec app php artisan key:generate --show
```

Paste the result into `stack.env` as `APP_KEY=base64:...`.

> ℹ️ For local PDF generation, the host portion of `APP_URL` must end in `.test` (Chrome DNS resolver quirk).

## Updating

```bash
git pull
cd debian
docker compose build
docker compose up -d
```

Take a backup before non-trivial upgrades.

## MySQL configuration

The `mysql` service runs `mysql:8.4` with these tuning flags:

- `--innodb-buffer-pool-size=512M` — InnoDB cache (default 128 MiB is too small)
- `--slow-query-log=ON` + `--long-query-time=1` — log any query >1s
- `--slow-query-log-file=/var/log/mysql/slow.log` — bind-mounted to `/srv/invoiceninja/mysql-logs/` on the host

The bind mount expects `/srv/invoiceninja/mysql-logs/` to exist on the host, owned by UID 999 (the in-container `mysql` user):

```bash
sudo mkdir -p /srv/invoiceninja/mysql-logs
sudo chown 999:999 /srv/invoiceninja/mysql-logs
sudo chmod 750 /srv/invoiceninja/mysql-logs
```

## Deployment (this fork)

This repo is deployed via a Portainer Git stack on `apple-pi.lan`. Redeploys pull from `origin/debian`, rebuild images locally, and recreate containers (~30s of nginx unavailability).

## Support

Bug reports for upstream behavior: open an issue at [invoiceninja/dockerfiles](https://github.com/invoiceninja/dockerfiles). Bug reports for fork-specific changes: open an issue here.
