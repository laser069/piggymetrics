# 09 - Every change, explained line by line

Written for someone new to Java projects and CI/CD. Read top to bottom.

## 0. Vocabulary first

| Word | Meaning |
|---|---|
| Maven | Java build tool. Reads `pom.xml`, downloads libraries, compiles, runs tests, makes a `.jar`. |
| `.jar` | A zipped, runnable Java program. |
| Microservice | One small program doing one job (login, accounts, ...). This repo has 9. |
| Docker image | A sealed package: program + everything it needs. Same image runs anywhere. |
| Container | A running copy of an image. |
| Dockerfile | Recipe to build one image. |
| docker compose | Starts many containers together from one YAML file. |
| Registry (GHCR) | A shelf where images are stored. GHCR is GitHub's shelf (`ghcr.io`). |
| CI | On every change: build + test automatically. |
| CD | On approved changes: ship to a server automatically. |
| Workflow / job / step | GitHub Actions file / a group of steps on one machine / one command or action. |
| Runner | The temporary GitHub machine that runs a job. |
| Secret | A value stored in GitHub (password, key). Not visible in the repo. |
| Artifact | A file a job saves so a later job can use it. |

---

## 1. `.github/workflows/ci-cd.yml` (the pipeline)

YAML note: indentation = nesting. `- ` starts a list item. `#` starts a comment.

### Header

```yaml
name: CI/CD
```
Name shown in the GitHub "Actions" tab.

```yaml
on:
  pull_request:
  push:
    branches: [master]
  workflow_dispatch:
```
When to run:
- `pull_request`: whenever a PR is opened or updated. Gives feedback before merging.
- `push` to `master`: whenever code lands on master (e.g. a merge). This is what triggers deploy.
- `workflow_dispatch`: adds a "Run workflow" button so you can start it by hand.

```yaml
permissions:
  contents: read
```
The automatic token the job receives can only read the repo. Principle of least privilege: if a step were hacked, damage is limited. Jobs that need more ask for it themselves.

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```
If you push twice quickly to the same PR, cancel the older run (it is outdated). `github.ref` = the branch. On master we do NOT cancel, so a half-finished release is never killed.

```yaml
env:
  REGISTRY: ghcr.io/${{ github.repository_owner }}
```
A variable available to every job: `ghcr.io/laser069`. `${{ ... }}` is how Actions inserts values.

### Job 1: `build-test`

```yaml
build-test:
  runs-on: ubuntu-22.04
```
Run on a fresh Ubuntu 22.04 machine.

```yaml
- uses: actions/checkout@v4
```
Download the repo's code onto the runner. (The machine starts empty.) `uses:` runs a ready-made action; `@v4` pins its version.

```yaml
- uses: actions/setup-java@v4
  with:
    distribution: temurin
    java-version: '8'
    cache: maven
```
Install Java 8 (this project targets Java 8 / Spring Boot 2.0.3; newer Java breaks it). `temurin` = a free, trusted Java build. `cache: maven` saves downloaded libraries between runs so runs are faster.

```yaml
- name: Cache embedded MongoDB binary
  uses: actions/cache@v4
  with:
    path: ~/.embedmongo
    key: embedmongo-${{ runner.os }}-${{ hashFiles('**/pom.xml') }}
    restore-keys: embedmongo-${{ runner.os }}-
```
The tests start a tiny throwaway MongoDB database ("embedded Mongo") that downloads a binary on first use. This caches that download. `key` changes when any `pom.xml` changes; `restore-keys` falls back to an older cache.

```yaml
- name: Build and test all modules
  run: mvn -B -ntp verify
```
The core CI step. `mvn verify` = compile + run all tests + package jars + run JaCoCo report. `-B` batch mode (clean logs), `-ntp` hides download progress noise. If any test fails, this step fails and the whole pipeline stops.

```yaml
- name: Upload test and coverage reports
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: reports
    path: |
      **/target/surefire-reports/
      **/target/site/jacoco/
    if-no-files-found: ignore
```
Save test results and the coverage report so you can download and read them. `if: always()` = do this even if tests failed (that's when you most need the reports). `**` means "any folder depth".

```yaml
- name: Upload service jars
  uses: actions/upload-artifact@v4
  with:
    name: jars
    path: '*/target/*.jar'
    if-no-files-found: error
```
Save the built programs. The next job reuses these exact files, so what we tested is exactly what we ship ("build once"). `error` = fail loudly if no jars were produced.

### Job 2: `docker`

```yaml
docker:
  needs: build-test
  if: github.event_name != 'pull_request' && github.ref == 'refs/heads/master'
```
`needs`: wait for build-test, and skip if it failed. `if`: only on master and never on PRs, so PRs can't publish images.

```yaml
permissions:
  contents: read
  packages: write
```
This job (only) may upload images to GHCR.

```yaml
strategy:
  fail-fast: false
  matrix:
    service: [config, registry, gateway, auth-service, account-service, mongodb]
```
Matrix = run this job 6 times in parallel, once per service name (available as `${{ matrix.service }}`). `fail-fast: false`: if one fails, let the others finish so you see all problems at once. These 6 are the services our small VM runs.

```yaml
- uses: actions/checkout@v4
- uses: actions/download-artifact@v4
  with:
    name: jars
```
Get the code (the Dockerfiles live in it) and the jars saved by job 1. They land at e.g. `gateway/target/gateway.jar`, which is where each Dockerfile expects them.

```yaml
- uses: docker/setup-buildx-action@v3
```
Enables Docker's modern builder (needed for caching).

```yaml
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```
Log in to GHCR. `GITHUB_TOKEN` is created automatically per run; you don't make it.

```yaml
- id: meta
  uses: docker/metadata-action@v5
  with:
    images: ${{ env.REGISTRY }}/piggymetrics-${{ matrix.service }}
    tags: |
      type=sha,prefix=sha-,format=short
      type=raw,value=latest
```
Works out the image name and tags. Result e.g. `ghcr.io/laser069/piggymetrics-gateway:sha-1a2b3c4` and `:latest`. The `sha-` tag identifies one exact commit (perfect for rollback); `latest` is a moving label.

```yaml
- uses: docker/build-push-action@v6
  with:
    context: ./${{ matrix.service }}
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    cache-from: type=gha,scope=${{ matrix.service }}
    cache-to: type=gha,mode=max,scope=${{ matrix.service }}
```
Build the image using that service's folder (and its Dockerfile) and push it. `cache-*` reuses unchanged Docker layers from earlier runs so builds are faster.

### Job 3: `deploy`

```yaml
deploy:
  needs: docker
  if: (same master-only condition)
  environment: production
  concurrency:
    group: deploy-production
    cancel-in-progress: false
```
Runs only after all images are pushed. `environment: production` links to a GitHub Environment where you can add protection (e.g. require your manual approval). `concurrency` ensures two deploys never run at once and a running one is never cancelled.

```yaml
env:
  SSH_OPTS: -i ~/.ssh/deploy_key -o StrictHostKeyChecking=accept-new -o ServerAliveInterval=30
  TARGET: ${{ secrets.VM_USER }}@${{ secrets.VM_HOST }}
```
Shortcuts reused below. `-i` = which private key. `accept-new` = trust the VM's identity the first time, complain if it later changes. `ServerAliveInterval` keeps the connection from timing out. `TARGET` = `user@ip`.

```yaml
- name: Prepare ssh key
  env:
    VM_SSH_KEY: ${{ secrets.VM_SSH_KEY }}
  run: |
    mkdir -p ~/.ssh && chmod 700 ~/.ssh
    printf '%s\n' "$VM_SSH_KEY" > ~/.ssh/deploy_key
    chmod 600 ~/.ssh/deploy_key
```
Writes your private key (from the secret) to a file; `chmod 600` because SSH refuses keys other users could read. The secret goes through an env var, not pasted into the script, which is safer.

```yaml
- name: Copy compose file
  run: |
    ssh $SSH_OPTS "$TARGET" 'mkdir -p ~/piggymetrics'
    scp $SSH_OPTS docker-compose.prod.yml "$TARGET":~/piggymetrics/docker-compose.yml
```
Create a folder on the VM and copy the production compose file there (renamed `docker-compose.yml` so plain `docker compose` finds it).

```yaml
- name: Write .env on the VM
  env: (the five passwords from secrets)
  run: |
    { echo "REGISTRY=..."; echo "TAG=sha-${GITHUB_SHA::7}"; echo "...PASSWORD=..." ... } \
      | ssh $SSH_OPTS "$TARGET" 'umask 077 && cat > ~/piggymetrics/.env'
```
Builds the `.env` file (KEY=value lines) and sends it through the SSH connection into a file on the VM. `GITHUB_SHA::7` = first 7 chars of the commit id, so the VM runs exactly the images built for this commit. `umask 077` makes the file readable only by its owner. Passwords never appear in command arguments or the repo.

```yaml
- name: Pull and restart containers
  env: GHCR_PAT, OWNER
  run: |
    printf '%s' "$GHCR_PAT" | ssh ... "docker login ghcr.io -u $OWNER --password-stdin"
    ssh ... 'cd ~/piggymetrics && docker compose pull && docker compose up -d --remove-orphans && docker image prune -f'
```
1. Log the VM into GHCR (new GHCR images are private by default, so the VM needs a token with `read:packages`).
2. `pull` downloads the new images; `up -d` starts/updates containers in the background (only changed ones restart); `--remove-orphans` removes containers no longer in the file; `image prune -f` deletes old unused images to save disk.

```yaml
- name: Health check
  run: |
    for i in $(seq 1 40); do
      code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 "http://$VM_HOST/" || true)
      ...
      if [ "$code" = "200" ]; then exit 0; fi
      sleep 15
    done
    ... print container status and logs ...
    exit 1
```
"Containers started" does not mean "app works". This asks the website (port 80) up to 40 times, 15 s apart (~10 min, as a small VM starts slowly). HTTP 200 = success. If it never answers, the step prints container logs for debugging and fails, turning the pipeline red.

---

## 2. `.github/workflows/codeql.yml` (security scan)

- Triggers: PRs, pushes to master, and a weekly `cron` schedule (`'0 3 * * 1'` = Mondays 03:00 UTC) because new vulnerability patterns are discovered over time.
- `security-events: write` lets it publish findings to the repo's Security tab.
- Steps: install Java -> `codeql-action/init` (start watching) -> `mvn -DskipTests package` (CodeQL needs to watch a compile; tests aren't needed) -> `analyze` (scan and report).

## 3. `.github/dependabot.yml`

Tells GitHub's Dependabot to check weekly for newer versions of: Maven libraries (`/`), the actions used in workflows, and Docker base images (`/gateway` as a sample Dockerfile). It opens PRs so versions don't go stale silently.

---

## 4. `docker-compose.prod.yml` (what runs on the VM)

```yaml
x-logging: &logging
  options:
    max-size: "10m"
    max-file: "3"
```
Defines a reusable block named `logging` (`&logging` = give it a name; `*logging` below = reuse it). Caps each container's log to 3 files of 10 MB so logs don't fill the small disk.

Each service block follows one pattern (example `gateway`):

```yaml
gateway:
  image: ${REGISTRY}/piggymetrics-gateway:${TAG}
  restart: always
  environment:
    CONFIG_SERVICE_PASSWORD: $CONFIG_SERVICE_PASSWORD
    JAVA_OPTS: -Xmx128m -Xss512k
  depends_on:
    config:
      condition: service_healthy
  ports:
    - 80:4000
  mem_limit: 300m
  logging: *logging
```
- `image`: which image to run; `REGISTRY` and `TAG` come from the `.env` the deploy wrote.
- `restart: always`: restart after a crash or VM reboot.
- `environment`: values the program reads. `$NAME` is filled from `.env`.
- `JAVA_OPTS`: Java settings. `-Xmx128m` caps Java's heap (working memory) at 128 MB; `-Xss512k` shrinks per-thread stack. Needed because the VM has <=2 GB.
- `depends_on ... service_healthy`: wait until the `config` service reports healthy before starting. Every other service needs config first (it hands out their settings).
- `ports: 80:4000`: VM port 80 -> container port 4000. This makes the website reachable at `http://VM_IP/`.
- `mem_limit`: hard memory cap so one container cannot starve the rest.

Services included: `config` (settings server), `registry` (service directory), `gateway` (front door + web UI), `auth-service` + `auth-mongodb` (login + its DB), `account-service` + `account-mongodb` (accounts + its DB). `INIT_DUMP: account-service-dump.js` tells the account DB which sample data to load. Left out to save RAM: monitoring, turbine, statistics, notification, rabbitmq.

## 5. `docker-compose.yml` (original, edited)

Only the `image:` lines changed:
`sqshq/piggymetrics-config` -> `${REGISTRY:-ghcr.io/laser069}/piggymetrics-config:${TAG:-latest}`.
`${VAR:-default}` = use VAR if set, else the default. Before, it pulled the original author's images; now it uses yours, by default `latest`.

## 6. The 9 `Dockerfile`s (config, registry, gateway, auth-service, account-service, statistics-service, notification-service, monitoring, turbine-stream-service)

Before:
```dockerfile
FROM java:8-jre
...
CMD ["java", "-Xmx200m", "-jar", "/app/gateway.jar"]
```
After:
```dockerfile
FROM eclipse-temurin:8-jre
...
ENV JAVA_OPTS="-Xmx200m"
CMD ["sh", "-c", "exec java $JAVA_OPTS -jar /app/gateway.jar"]
```
- `FROM` = starting image. `java:8-jre` is abandoned/deprecated; `eclipse-temurin:8-jre` is its maintained replacement (same Java 8 runtime).
- `ENV JAVA_OPTS="-Xmx200m"` sets a default heap size of 200 MB.
- `CMD` previously hard-coded `-Xmx200m`. Now `sh -c "... $JAVA_OPTS ..."` reads the variable so compose can override it (the VM uses 128m). `exec` makes Java replace the shell as the main process so it receives stop signals properly.
- Unchanged lines: `ADD ./target/gateway.jar /app/` copies the jar from the CI build into the image; `EXPOSE` documents the port; config's `HEALTHCHECK curl ...` is what `service_healthy` uses.

The `mongodb/Dockerfile` is unchanged (still built and pushed as an image).

## 7. Root `pom.xml` (JaCoCo)

Added a `<build><plugins>` block with `jacoco-maven-plugin` 0.8.5:
- `prepare-agent` goal: attaches a recorder to the test run that notes which lines of code ran.
- `report` goal bound to the `verify` phase: after tests, writes an HTML coverage report under `target/site/jacoco/`.
Every module inherits from this root pom, so all get coverage with one edit.

## 8. The four service `pom.xml`s (account, auth, notification, statistics)

These were already your uncommitted edits. Each adds the `lombok` library (marked `optional`), which generates boilerplate such as getters/setters at compile time.

## 9. Secrets hygiene

- `.env` was committed with real values. Now `git rm --cached .env` untracked it (file stays on your disk), `.gitignore` gets a `.env` line so it cannot be re-added by accident, and `.env.example` shows the shape with fake values.
- Anything already in git history stays visible there; rotate real passwords.

## 10. `.travis.yml` deleted

The original author's old CI (Travis). It contained encrypted values only they can use, and would do nothing for you now. GitHub Actions replaces it.

## 11. `README.md`

The two badges at the top (Travis, Codecov) now point to your CI/CD and CodeQL workflows, so the README shows green/red status.

## 12. Test change: `statistics-service/.../ExchangeRatesClientTest.java`

Added `import org.junit.Ignore;` and `@Ignore("...")` on the class. Those two tests call a real public exchange-rate website; it now answers `401 missing_access_key`, so they failed in CI for reasons unrelated to our code. `@Ignore` tells JUnit to skip them (with the reason in the report). The proper long-term fix is mocking that web call in the test.

## 13. `learn/` docs

`learn/01`-`07` were your own notes; `08-cicd.md` is the short overview; this file is the detailed walk-through.

---

## 14. How a change flows, end to end

1. You commit on a branch and open a PR.
2. `build-test` runs: compile, tests, coverage. CodeQL also scans. Red = fix before merging.
3. You merge into `master`.
4. `build-test` runs again on master, then `docker` builds and pushes 6 images tagged `sha-<commit>` and `latest`.
5. `deploy` SSHes into the VM, writes `.env`, pulls those exact images, restarts containers.
6. Health check hits `http://VM_IP/`. Green = live. Red = logs printed, previous containers keep the old state until you fix it.

Rollback: on the VM edit `TAG=sha-<older>` in `~/piggymetrics/.env`, run `docker compose up -d`.
