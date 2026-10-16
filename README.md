# kaneo_build

Komodo stack for [Kaneo](https://github.com/usekaneo/kaneo), the open-source
project management tool, running on the node's **shared** Postgres and Redis.

| Service | Image | Port | Purpose |
| --- | --- | --- | --- |
| `kaneo` | `ghcr.io/usekaneo/kaneo:2.46.2` | 5173 (loopback) | API + web app in one container |

There is no bundled database. Kaneo uses:

- **Postgres** from [`postgres_build`](https://github.com/svnix-solutions/postgres_build)
- **Redis** from [`redis_build`](https://github.com/svnix-solutions/redis_build)
  (optional; only relays live updates between multiple API instances)

Both are reached by name over the external `db` network.

Public traffic flows cloudflared → Nginx Proxy Manager → kaneo. Kaneo joins only
the internal `proxy` network it shares with Nginx Proxy Manager, not the
`cloudflared` network, so the tunnel connector can't reach it directly.

---

## Setup

1. Networks, once per host:

   ```bash
   docker network create db 2>/dev/null || true
   docker network create proxy 2>/dev/null || true
   ```

2. Provision the database on the shared Postgres:

   ```bash
   docker exec -i postgres psql -U postgres <<'SQL'
   CREATE ROLE kaneo LOGIN PASSWORD '<hex password>';
   CREATE DATABASE kaneo OWNER kaneo;
   SQL
   ```

   Kaneo runs its own migrations at startup.

3. Fill the environment from `.env.example`. Generate secrets with
   `openssl rand -hex 32`. Use hex for the DB password because it is embedded in
   `DATABASE_URL` unencoded.

4. Deploy, then route the public hostname to `http://kaneo:5173` in Nginx Proxy
   Manager. Set **Websockets Support** on the NPM
   proxy host: live updates use them.

5. Open the site and create the first account. It becomes the instance owner.
   Then set `KANEO_DISABLE_REGISTRATION=true` and redeploy.

---

## Notes

- `KANEO_AUTH_SECRET` is required here. The image would otherwise generate a new
  one on every start, logging everyone out.
- `KANEO_CLIENT_URL` must be the exact browser origin. The API URL is derived as
  `<origin>/api`, and CORS and WebSocket origins are checked against it.
- Optional settings (SMTP, GitHub OAuth, S3 uploads, access policy) pass through
  from the environment via `env_file`. See the upstream
  [`.env.sample`](https://github.com/usekaneo/kaneo/blob/main/.env.sample).
- Backups: all state is in the `kaneo` database, covered by the shared
  Postgres stack's nightly `pg_dumpall`. Uploads, if enabled, live in S3.

## Files

| File | Purpose |
| --- | --- |
| `compose.yaml` | the stack definition |
| `.env.example` | template for the Komodo Environment field |
