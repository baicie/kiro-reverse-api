# Deploying this service

Audience: an operator or an automated agent doing a first deploy or a release.
Everything below is idempotent unless a step says otherwise.

## 0. What you are deploying

One Go binary serving the admin UI and the API. All state is a single SQLite
file under `DATA_DIR`. It runs as a container behind a reverse proxy that
terminates TLS; the container itself only listens on loopback.

## 1. Release path

```
git tag v<date>.<seq>  ->  push tag
  ->  GitHub Actions builds the image (linux/amd64)
  ->  pushes ghcr.io/<owner>/<repo>:<tag>
  ->  verifies the tag exists in the registry
  ->  ssh to the host, pulls that digest, runs docker compose
  ->  gate: container running AND GET /health returns 200
```

A release that fails either gate must not be reported as deployed.

## 2. Files that matter

| path | purpose |
|---|---|
| `Dockerfile` | builds the UI stage (Node + pnpm) then the Go binary into an Alpine image |
| `deploy/docker-compose.deploy.yml` | image-only compose; loopback port, `./data` bind mount, env passthrough |
| `.github/workflows/deploy.yml` | the release pipeline described in section 1 |
| `.github/workflows/docker.yml` | branch/PR smoke build only; must NOT listen on `v*` tags |

Do not teach the release pipeline to build on the host. The host only pulls.

## 3. One-time host preparation

1. Docker Engine with the Compose v2 plugin. Verify: `docker compose version`.
2. Access model: either the deploy user is in the `docker` group, or passwordless
   `sudo -n docker` works. Both pipelines detect this automatically.
3. Create the deploy directory (default `~/kiro-reverse-api`) and `./data`.
4. Write `./data`-owning uid to `PUID`/`PGID` if the container does not run as root.
5. Create `.env` next to the compose file, mode 600, with the values from section 7.
6. Point a reverse proxy vhost at `127.0.0.1:<HOST_PORT>` and give it a
   certificate. A grey-cloud (DNS-only) A record avoids a CDN in front of the API.

## 4. Repository configuration

Values only; never commit them.

| kind | name | purpose |
|---|---|---|
| secret | `DEPLOY_SSH_KEY` | preferred: key-based SSH |
| secret | `DEPLOY_SSH_PASSWORD` | fallback when no key is provisioned |
| variable | `DEPLOY_HOST` | target host |
| variable | `DEPLOY_USER` | ssh user |
| variable | `DEPLOY_PORT` | ssh port, default 22 |
| variable | `DEPLOY_DIR` | deploy directory on the host |
| variable | `CONTAINER_NAME` | container name, default `kiro-proxy` |
| variable | `HOST_PORT` | loopback port, default 8080 |
| variable | `PUID` / `PGID` | owner of the bind-mounted data dir |
| variable | `GHCR_MIRRORS` | comma-separated registry mirrors, tried in order |

## 5. Release procedure

```bash
git tag v2026.01.01.1200
git push origin v2026.01.01.1200
gh run list      --workflow deploy.yml --limit 1
gh run view <id> --json status,conclusion,jobs
```

Then verify from outside the host: `curl -sS https://<public-host>/health`.

## 6. Rollback

```bash
cd <DEPLOY_DIR>
sudo -n env IMAGE_REPO=<image> IMAGE_TAG=<previous-tag> CONTAINER_NAME=kiro-proxy \
  HOST_PORT=8080 COMPOSE_PROJECT_NAME=kiro-proxy \
  docker compose -f docker-compose.deploy.yml up -d --pull never
```

Keep at least the last two tags pulled on the host so a rollback needs no network.

## 7. Runtime configuration

| variable | meaning | default |
|---|---|---|
| `DATA_DIR` | directory holding `kiro.db` | `/app/state` in the image |
| `ADMIN_PASSWORD` | admin panel password; overrides the stored one | generated on first run |
| `LOG_LEVEL` | `debug` / `info` / `warn` | `info` |
| `KIRO_VERSION` | IDE version reported upstream | built-in default |
| `KIRO_SYSTEM_VERSION` | platform string reported upstream, e.g. `darwin#24.6.0` | derived from the runtime OS |
| `KIRO_NODE_VERSION` | Node version reported upstream | built-in default |
| `KIRO_PROFILE_ARN` | pin one profile ARN for every account (single-account hosts only) | unset |
| `KIRO_ADMIN_SESSION_TTL` | console login lifetime, Go duration | `12h` |

The `KIRO_*` values live in the environment on purpose: they describe the client
identity that upstream sees, and a rebuilt database must not silently fall back
to a Linux UA.

## 8. Upstream credentials

Accounts are imported, not guessed:

1. Sign in with the official IDE or CLI on a workstation.
2. Read the credentials from its local store (the CLI keeps a `social:token`
   record and the resolved profile ARN).
3. `POST /admin/api/auth/credentials` with `accessToken`, `refreshToken`,
   `provider`, `region`, `email`, `authMethod` and `profileArn`.

Rules learned the hard way:

- Importing without `profileArn` yields `400 profileArn is required`.
- Pinning another account's ARN yields `403 bearer token invalid` and can get the
  account flagged; the ARN must come from the same account it is used with.
- Builder-ID device login creates a different AWS identity than the IDE session,
  and cannot look an ARN up at all. Prefer importing a social-identity token.
- An official client and this service sharing one account evict each other's
  session. Authenticate once, then leave the client alone.
- Verify an account with a single `POST /v1/chat/completions`. The panel's refresh
  and test buttons misreport and can break a working token.

## 9. Data and backup

`DATA_DIR/kiro.db` is the entire state. Back it up with SQLite's online backup
API (not a file copy while the service runs), then `VACUUM` to reclaim space.
Keep the `.env` next to the compose file in the same backup set.

## 10. Failures seen in production

| symptom | cause | fix |
|---|---|---|
| pull stalls at one layer, then times out | registry blob CDN unreachable from the host | set `GHCR_MIRRORS`; pulls are by digest, so a mirror cannot change the image |
| `context deadline exceeded` on the first release | cold pull of a ~50 MB image can take minutes | keep the pull timeout at 600s or more |
| deploy step exits 1 with no output | ssh joins forwarded argv into one string; an empty optional argument collapses and `${10}` is unbound under `set -u` | pass optional arguments last and read them as `${10:-}` |
| `exit 255` mid-pull | idle SSH session dropped during a slow pull | `-o ServerAliveInterval=15 -o ServerAliveCountMax=10` |
| compose starts the wrong image or port | `sudo` resets the environment | pass `IMAGE_REPO`, `IMAGE_TAG`, `HOST_PORT`, `PUID`, `PGID` through `sudo -n env` |
| container up but the pipeline still red | health gate uses the wrong path | this service exposes `/health`, not `/healthz` or `/api/status` |
