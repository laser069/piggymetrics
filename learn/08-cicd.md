# 08 - CI/CD pipeline (GitHub Actions)

Workflow: `.github/workflows/ci-cd.yml`. Extras: `codeql.yml`, `dependabot.yml`.

## Flow

```
PR / push ──► build-test ──(master only)──► docker (x6 matrix) ──► deploy (SSH to VM) ──► health check
```

| Stage | Runs on | What | Why |
|---|---|---|---|
| build-test | every PR + push | `mvn verify` on JDK 8: compile, unit tests (embedded Mongo), JaCoCo coverage, package jars | Catch broken code before anything is published. Fast feedback on PRs. |
| docker | push to master | Build 1 image per service (config, registry, gateway, auth, account, mongodb), push to GHCR as `sha-<7>` + `latest` | Immutable artifact. Same jar tested in CI is the jar shipped (no rebuild). `sha` tag = exact rollback target. |
| deploy | after docker | scp compose, write `.env` from secrets, `docker compose pull && up -d`, poll `http://VM/` | Auto-release. Health check proves the app answers, not just that containers started. |
| CodeQL | PR, master, weekly | Static security analysis | Finds vulnerable patterns early. |
| Dependabot | weekly | PRs for maven / actions / docker updates | Keeps an old Spring Boot 2.0.3 stack from rotting silently. |

## Design choices

- **Least privilege**: workflow default `contents: read`; only the docker job gets `packages: write`.
- **Gates**: `needs:` chains jobs. Failed tests = no image = no deploy.
- **Concurrency**: PR runs cancel older runs; deploys queue, never overlap, never cancel mid-flight.
- **Secrets**: never in repo. `.env` is gitignored (`.env.example` shows shape); the deploy job writes it on the VM over ssh stdin.
- **Low RAM VM (<=2GB)**: `docker-compose.prod.yml` runs only config, registry, gateway, auth, account + 2 Mongo. Each JVM gets `-Xmx128m` through `JAVA_OPTS` (Dockerfiles default `-Xmx200m`) and a `mem_limit`.
- **Image names** in `docker-compose.yml` use `${REGISTRY}` / `${TAG}`.

## One-time setup

VM (Linux):
1. Install Docker + compose plugin; add deploy user to `docker` group.
2. Swap: `sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile` (+ `/etc/fstab` entry).
3. Open ports 80 (and 8761 if you want Eureka).
4. Add the CI public key to `~/.ssh/authorized_keys` (`ssh-keygen -t ed25519 -f deploy_key`).

GitHub (Settings -> Secrets and variables -> Actions), create Environment `production`, then secrets:
`VM_HOST`, `VM_USER`, `VM_SSH_KEY` (private key), `GHCR_PAT` (classic PAT, `read:packages`),
`CONFIG_SERVICE_PASSWORD`, `NOTIFICATION_SERVICE_PASSWORD`, `STATISTICS_SERVICE_PASSWORD`, `ACCOUNT_SERVICE_PASSWORD`, `MONGODB_PASSWORD`.

## Rollback

On the VM: edit `TAG=sha-<old>` in `~/piggymetrics/.env`, then `docker compose up -d`.
