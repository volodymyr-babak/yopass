# ThingsBoard Secrets — Build & Run Guide

This is a ThingsBoard-rebranded fork of [yopass](https://github.com/jhaals/yopass) for sharing encrypted secrets and files with customers (and from customers back to us). End-to-end encrypted in the browser via OpenPGP — the server never sees plaintext.

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Go | 1.21+ | `apt install golang` or [go.dev/dl](https://go.dev/dl) |
| Node | 22.x (anything ≥20 should work) | [nvm](https://github.com/nvm-sh/nvm) recommended |
| Yarn | 1.x classic | `npm install -g yarn` |
| Docker | any | for memcached, optional if you have memcached natively |

Clone:

```bash
git clone <this-repo-url> yopass
cd yopass
```

---

## Build

Two artifacts: the React SPA (must be built before the Go binary can serve it) and the Go server binary.

```bash
# 1. Build the SPA — output goes to website/dist/
cd website
yarn install
yarn build
cd ..

# 2. Build the Go server — output is ./yopass
go build -o yopass ./cmd/yopass-server
```

Verify the SPA artifact exists and contains the TB branding:

```bash
ls website/dist/                              # should list: assets/ index.html theme-init.js thingsboard.svg
grep -o "ThingsBoard Secrets" website/dist/index.html
```

---

## Run (local dev)

Yopass needs a key/value store. Easiest is memcached in Docker.

```bash
# 1. memcached
docker run -d --name yopass-memcached -p 11211:11211 memcached

# 2. yopass server — IMPORTANT: --asset-path must point to the SPA build output
./yopass \
  --memcached=127.0.0.1:11211 \
  --address=127.0.0.1 \
  --port=1337 \
  --asset-path=./website/dist
```

> **Gotcha:** the default `--asset-path` is the relative string `"public"`, which is **not** the SPA build directory. If you forget `--asset-path=./website/dist` the server returns `404 page not found` for `/`, `/logo`, the favicon, and every static asset, while `/config` still works. If you see this — that's the cause.

Open <http://127.0.0.1:1337/>.

Stop:

```bash
# kill yopass (Ctrl-C in its terminal, or)
pkill -f './yopass --memcached'
docker stop yopass-memcached && docker rm yopass-memcached
```

---

## Validation checklist

After starting, open the app and confirm:

1. **Tab title** reads `ThingsBoard Secrets: Share Secrets Securely`
2. **Favicon** is the blue ThingsBoard wheel (force-reload with Ctrl-Shift-R if browser cached the old one)
3. **Navbar** shows the blue wheel + the text `ThingsBoard Secrets`
4. **Primary buttons** (`Encrypt Message`, `Upload`, `Decrypt secret`, `Copy`) are ThingsBoard blue `#305680` — not green
5. **Theme toggle** (sun/moon, top-right) switches between `corporate` (light blue) and `business` (dark blue). Primary stays TB blue on both.
6. **Feature cards** at the bottom mention "ThingsBoard Secrets", not "Yopass"
7. **Footer** reads `Powered by Yopass — created by Johan Haals` — required by the upstream MIT license

Smoke test:

```text
Home page → paste any text → click "Encrypt Message"
→ copy the one-click link
→ open it in an incognito window
→ click "Reveal Secure Message"
→ confirm the original text comes back
```

Repeat with a small file via the `Upload` tab.

```bash
# from CLI — should all return 200
curl -sS -o /dev/null -w "/                : HTTP %{http_code}\n" http://127.0.0.1:1337/
curl -sS -o /dev/null -w "/logo            : HTTP %{http_code}\n" http://127.0.0.1:1337/logo
curl -sS -o /dev/null -w "/thingsboard.svg : HTTP %{http_code}\n" http://127.0.0.1:1337/thingsboard.svg
curl -sS                                            http://127.0.0.1:1337/config | jq '{THEME_LIGHT,THEME_DARK}'
# expected: {"THEME_LIGHT":"corporate","THEME_DARK":"business"}
```

---

## What's customised vs. upstream

Everything below is hardcoded so the rebrand applies **without** a yopass license key (which is what gates the runtime `--app-name`/`--logo-url`/`--theme-*` flags upstream).

| File | What we changed |
|------|----------------|
| `website/public/thingsboard.svg` | New asset — ThingsBoard wheel in brand color `#305680` |
| `website/index.html` | Tab title + favicon `<link>` |
| `website/public/theme-init.js` | Initial daisyUI theme constants |
| `website/src/shared/theme/theme.ts` | `DEFAULT_LIGHT_THEME` / `DEFAULT_DARK_THEME` |
| `website/src/shared/context/ConfigContext.tsx` | `defaultConfig` theme fallback |
| `website/src/shared/components/Navbar.tsx` | Logo/name fallback when `LOGO_URL` not set |
| `website/src/shared/styles/index.css` | Pinned `--color-primary` to `oklch(40% 0.08 254)` ≈ `#305680` on every daisyUI theme |
| `website/src/shared/locales/en.json` | App name, feature copy, footer credit string |
| `pkg/server/server.go` | No-license theme fallback + `logoHandler` serves `thingsboard.svg` |

**Not changed**: the 12 non-English locale files. The UI defaults to English; if a user switches language they'll still see "Yopass" strings. Translate `header.appName` and `features.*` in those files if you need full multilingual coverage.

---

## Deploy (production)

Same artifacts as local — just behind a TLS-terminating reverse proxy. Two common patterns:

### A) Single Docker image

```dockerfile
# Multi-stage Dockerfile
FROM node:22 AS webbuild
WORKDIR /src/website
COPY website/ ./
RUN yarn install --frozen-lockfile && yarn build

FROM golang:1.22 AS gobuild
WORKDIR /src
COPY . .
COPY --from=webbuild /src/website/dist ./website/dist
RUN CGO_ENABLED=0 go build -o /yopass ./cmd/yopass-server

FROM gcr.io/distroless/static-debian12
COPY --from=gobuild /yopass /yopass
COPY --from=webbuild /src/website/dist /assets
EXPOSE 1337
ENTRYPOINT ["/yopass", "--asset-path=/assets"]
```

Run:

```bash
docker build -t thingsboard-secrets .
docker run -d --name secrets -p 1337:1337 \
  --link memcached \
  thingsboard-secrets \
  --memcached=memcached:11211 \
  --address=0.0.0.0
```

### B) docker-compose (recommended — published image + Redis)

A ready-to-use prod-grade compose is committed at `deploy/docker-compose/insecure/`. It runs the published image `volodymyrbabak/tb-yopass:1.0.0` with a Redis backend (persistence + password), healthchecks, restart policy, and security hardening (`no-new-privileges`, `read_only`). Redis is not exposed externally — only reachable on an internal bridge network. Yopass binds to `127.0.0.1:1337`, so put a TLS-terminating reverse proxy (nginx / traefik / ALB) in front for public exposure.

**Files**

| Path | Purpose |
|------|---------|
| `deploy/docker-compose/insecure/docker-compose.yml` | The stack: `redis` + `yopass` services, internal network, `redis-data` volume |
| `deploy/docker-compose/insecure/.env.example` | Template for `REDIS_PASSWORD` |

**First-time setup**

```bash
cd deploy/docker-compose/insecure

# 1. Create .env with a strong Redis password
cp .env.example .env
REDIS_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | head -c 48)
sed -i "s|^REDIS_PASSWORD=.*|REDIS_PASSWORD=${REDIS_PASSWORD}|" .env

# 2. Pull & start
docker compose pull
docker compose up -d

# 3. Verify
docker compose ps                              # both must show (healthy)
curl -fsS http://127.0.0.1:1337/ready          # {"status":"ready"}
curl -fsS http://127.0.0.1:1337/health         # {"status":"healthy"}
```

Open <http://127.0.0.1:1337/> through your reverse proxy.

**Setting public URL & other flags**

Add environment variables to the `yopass` service in `docker-compose.yml` — all server flags are exposed as env vars (uppercase, `-` → `_`):

```yaml
    environment:
      - DATABASE=redis
      - REDIS=redis://:${REDIS_PASSWORD}@redis:6379/0
      - PORT=1337
      - PUBLIC_URL=https://secrets.thingsboard.io   # so share links use this domain
      - DEFAULT_EXPIRY=24h
      - MAX_FILE_SIZE=512KB
      - PRIVACY_NOTICE_URL=https://thingsboard.io/privacy
      # - FORCE_ONETIME_SECRETS=true                # uncomment to disallow multi-view secrets
```

Apply with `docker compose up -d` (recreates only the changed container).

**Day-2 operations**

```bash
# Logs
docker compose logs -f yopass
docker compose logs -f redis

# Update to a newer image tag — edit image: line, then:
docker compose pull && docker compose up -d

# Restart everything
docker compose restart

# Stop (keeps data)
docker compose down

# Wipe everything including Redis data — destructive
docker compose down -v
```

**Backups**

Secrets live in the `redis-data` volume (AOF persistence, fsync every 1s). To back up:

```bash
# Trigger an AOF rewrite, then copy the volume contents
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" BGREWRITEAOF
docker run --rm \
  -v insecure_redis-data:/data:ro \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/redis-$(date +%F).tar.gz -C /data .
```

Restore by stopping the stack, replacing the volume contents from the tar, and starting again.

**Hardening notes already applied**

- `no-new-privileges: true` on both services
- `read_only: true` on the yopass container (works because no local file-store is configured; switch to a tmpfs/volume if you set `FILE_STORE=disk`)
- Redis: `requirepass`, AOF persistence, `maxmemory 256mb`, `maxmemory-policy noeviction` so writes fail loudly on memory pressure instead of silently dropping secrets
- yopass exposed on `127.0.0.1` only — public exposure must go through a reverse proxy

**Adding TLS / public DNS**

The `with-nginx-proxy-and-letsencrypt/` directory contains the upstream pattern using `jwilder/nginx-proxy` + `letsencrypt-nginx-proxy-companion`. To use it with our setup, port the `redis` service + Redis env vars from `insecure/docker-compose.yml` into it and set `VIRTUAL_HOST` / `LETSENCRYPT_HOST` / `LETSENCRYPT_EMAIL` on the yopass service.

---

## Useful server flags

| Flag | Purpose |
|------|---------|
| `--asset-path=./website/dist` | **Required.** Path to the built SPA. |
| `--memcached=host:11211` | Storage backend (alternative: `--redis=...` or `--database-file=...`). |
| `--address=127.0.0.1` | Bind address. Use `0.0.0.0` behind a reverse proxy or in Docker. |
| `--port=1337` | HTTP port. |
| `--max-length=10000` | Max bytes for a text secret. Default 10000. |
| `--max-file-size=512` | Max file size in KB. Increase for larger uploads. |
| `--default-expiry=24h` | Default expiration. Accepts `1h`, `24h`, `168h`. |
| `--force-onetime-secrets` | Force every secret to be one-time-view. |
| `--require-auth` | Require login (needs OIDC config). Useful if you want only TB-internal users to *create* secrets. |
| `--public-url=https://secrets.thingsboard.io` | Domain used in generated share links. Set in prod. |
| `--privacy-notice-url=...` / `--imprint-url=...` | Footer legal links. |

All flags also work as env vars (uppercase, dashes → underscores), e.g. `ASSET_PATH=/assets`.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `404 page not found` on `/` but `/config` works | `--asset-path` not pointed at the SPA build | Add `--asset-path=./website/dist` |
| Favicon still says Yopass | Browser cache | Hard-reload (Ctrl-Shift-R) |
| Primary buttons are green | Stale SPA build — old `index.css` | `cd website && yarn build` again, restart server |
| `dial tcp :11211: connection refused` | Local dev: memcached not running | `docker start yopass-memcached` |
| `dial tcp redis:6379: ... connection refused` or auth errors | Redis container unhealthy or wrong password | `docker compose ps`; confirm `REDIS_PASSWORD` in `.env` matches what's in the running container |
| Generated share link uses `localhost` in prod | `--public-url` not set | Set `--public-url=https://your-domain` |
| OIDC login redirects loop | OIDC `redirect_uri` mismatch | Make sure the IdP allows `<public-url>/auth/callback` |

---

## License & attribution

This fork inherits the upstream **MIT** license. The original yopass project by [Johan Haals](https://github.com/jhaals) is the source of all encryption, storage, and routing logic. The footer attribution must stay. Source: <https://github.com/jhaals/yopass>.
