# 08 - CI/CD pipeline (GitHub Actions)

Workflow: `.github/workflows/ci-cd.yml`.

## Flow

```
PR / push ──► build-test ──(master only)──► docker (one image per service, pushed to GHCR)
```

| Job | Runs on | What | Why |
|---|---|---|---|
| build-test | every PR + push | JDK 8, `mvn verify`: compile, run tests, package jars | Catch broken code before it is merged or published. |
| docker | push to master, after tests pass | Build an image per service (config, registry, gateway, auth-service, account-service, mongodb) and push to `ghcr.io/<owner>/piggymetrics-<service>` tagged `sha-<commit>` and `latest` | Turns tested code into a ready-to-run artifact. The jars built in the first job are reused, so what was tested is what ships. |

PRs only run `build-test`, so unreviewed code can never publish images.

## Run the published images

```
docker login ghcr.io
REGISTRY=ghcr.io/<owner> TAG=latest docker compose -f docker-compose.prod.yml up -d
```
(set the passwords from `.env.example` in a `.env` first)

## Other changes made along the way

- Dockerfiles use `eclipse-temurin:8-jre` and read the heap size from `JAVA_OPTS`.
- `docker-compose.yml` image names come from `${REGISTRY}` / `${TAG}`.
- `.env` is no longer committed (`.env.example` shows the format).
- Old `.travis.yml` removed.
- Two tests that call a live exchange-rate API (needs a key now) are `@Ignore`d.

See `09-changes-explained.md` for a detailed walkthrough.
