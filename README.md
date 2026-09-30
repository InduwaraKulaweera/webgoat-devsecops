# IE3142 DevSecOps Pipeline — OWASP WebGoat

## Application
- **Selected app:** OWASP WebGoat (custom-built image, bundles WebGoat + WebWolf)
- **Stack:** Java / Spring Boot
- **Components:**
  1. `webgoat` — single container serving WebGoat (port 8080 internally) and its
     WebWolf companion service (port 9090 internally), using WebGoat's own
     embedded database for accounts and lesson progress. Mapped to host ports
     `18080`/`19090` (see "Why non-default host ports" below).

> **Why not an external database?** An external PostgreSQL container was tried
> first. It failed: the official `webgoat/webgoat` image does not ship the
> PostgreSQL JDBC driver on its classpath, so `SPRING_DATASOURCE_URL` overrides
> fail with `Cannot load driver class: org.postgresql.Driver`. The module's own
> Appendix A already confirms WebGoat's app + embedded database satisfies the
> two-component requirement, so this is the architecture used. Documented in
> the report's reflection section.

> **Why a custom-built image instead of the official one?** This repository
> contains secure-coding fixes for four vulnerabilities (see below), applied
> directly to WebGoat's Java source. The official `webgoat/webgoat` Docker Hub
> image only contains the original, unmodified (vulnerable) code, so we build
> our own image from the patched source in `./WebGoat` using the `Dockerfile`
> in that folder, instead of pulling the official image.

> **Why non-default host ports (18080/19090)?** On some Windows + Docker
> Desktop (WSL2) setups, ports 8080/9090 fell inside a Windows-reserved
> dynamic port range, causing connections to reset (`ERR_EMPTY_RESPONSE`)
> even though the container was running correctly. Remapping to 18080/19090
> avoided the conflict. If this doesn't affect your machine, feel free to
> change these back to 8080/9090 in `docker-compose.yml`.

## Fixed vulnerabilities (Section 2.3)
| # | Vulnerability | Lesson class | Fix applied |
|---|---|---|---|
| 1 | SQL Injection | `SqlInjectionLesson9` | Parameterised query (`PreparedStatement`) |
| 2 | Reflected XSS | `CrossSiteScriptingLesson5a` | Output encoding (`HtmlUtils.htmlEscape`) |
| 3 | XXE | `SimpleXXE` / `CommentsCache` | Enabled DTD/external-entity restrictions |
| 4 | IDOR | *(see report for class/fix details)* | Server-side ownership check |

## Prerequisites
- Docker + Docker Compose installed
- JDK 25 and Maven — only needed if you want to rebuild the jar from source
  (e.g. after further code changes). Not needed just to run the container.
- No cloud account or Kubernetes needed — runs fully offline once the image is built

## Setup

1. Clone this repository (includes the `WebGoat/` source folder with our fixes
   already applied, and the already-built jar under `WebGoat/target/`, if present).
2. Copy the environment template:
   ```bash
   cp .env.example .env
   ```
3. Build the image and start the container:
   ```bash
   docker compose up -d --build
   ```
   (`--build` is required the first time, and any time the source/jar changes.
   For a plain restart with no code changes, `docker compose up -d` is enough.)
4. Give it 20-60 seconds to fully boot (Spring Boot startup), then verify:
   - WebGoat: http://127.0.0.1:18080/WebGoat
   - WebWolf: http://127.0.0.1:19090/WebWolf
5. Register a WebGoat account, log in, and open a lesson.
6. Confirm progress survives a restart:
   ```bash
   docker compose restart webgoat
   ```
   Log back in — your account and lesson progress should still be there, because
   `/home/webgoat/.webgoat` is mounted as a named Docker volume (`webgoat_data`),
   not left inside the ephemeral container filesystem.

## Rebuilding after a source code change
```bash
cd WebGoat
./mvnw.cmd clean install -DskipTests   # rebuild the jar (Windows: mvnw.cmd, Mac/Linux: ./mvnw)
cd ..
docker compose up -d --build           # rebuild the image with the new jar
```

## Tearing down
```bash
docker compose down          # stop containers, keep progress data
docker compose down -v       # stop containers and wipe all data
```

## Secrets management (see report Section 2.5)
- No credentials are hardcoded in `docker-compose.yml` or any source file.
- Local development config lives only in `.env`, which is git-ignored.
- CI/CD pipeline secrets are provisioned via GitHub Actions encrypted secrets (see `.github/workflows/`).
