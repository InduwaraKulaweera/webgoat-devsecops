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
| 4 | IDOR | `IDORViewOtherProfile`, `IDOREditOtherProfile` | Server-side ownership check: the requested `userId` (path, and body for PUT) must equal the authenticated user's id from the session, otherwise the request is rejected |

> **Security note:** `docker-compose.yml` publishes the ports on `127.0.0.1` only.
> WebGoat is intentionally vulnerable, so it should never be reachable from other
> machines on your network.

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

## CI/CD pipeline (`.github/workflows/devsecops-pipeline.yml`)
Runs on every push and pull request to `main`.

| Stage | Tool | What it checks | Fails the job when |
|---|---|---|---|
| Build & Test | Maven wrapper, JDK 25 | The patched source compiles and packages | The build fails |
| Gate 1 - SAST | Semgrep (`config: auto`) | Insecure code patterns in the source | The Semgrep action reports a blocking result |
| Gate 2 - SCA | OWASP Dependency-Check | Known CVEs in dependencies | A dependency has CVSS >= 9, or `NVD_API_KEY` is missing |
| Gate 3 - Secrets | Gitleaks (full history) | Committed credentials | A secret is detected |
| Gate 4 - Container | Trivy | CRITICAL/HIGH CVEs in the built image | Any CRITICAL/HIGH finding (`exit-code: 1`) |

Notes on how the gates behaved during development:
- Gate 4 (Trivy) blocked the pipeline on real dependency vulnerabilities. They were
  fixed by pinning newer Jackson, Tomcat and Guava versions in `WebGoat/pom.xml`.
- Gate 2 first failed because no NVD API key was passed to the plugin (a
  configuration error, not a finding). The key now comes from the `NVD_API_KEY`
  GitHub Actions secret.
- Semgrep OSS has no IDOR rule, and the XXE "before" finding was a CSRF-related
  rule, so for those two lessons the functional exploit test is the primary
  proof of the fix, not the Semgrep count.

Dependency-Check findings handled after the first full scan (CVSS >= 9 blocks the build):
- `underscore` 1.10.2 (bundled UI library, CVE-2021-23358 / CVE-2026-27601) was
  upgraded to 1.13.8 in `static/js/libs/underscore-min.js`.
- `prism-parser-api` (a Java Ruby-parser library pulled in via asciidoctorj/jruby) was
  matched by name to the Ruby *interpreter* CPE, giving 2008 Ruby CVEs. This is a
  false positive and is suppressed narrowly in `WebGoat/dependency-check-suppressions.xml`
  with the reason documented. No other finding is suppressed and the CVSS 9 threshold is unchanged.
- Lower-severity findings (old jQuery, jQuery UI, Bootstrap 3 bundled in the UI) remain
  reported in the uploaded HTML report but do not reach the CVSS 9 threshold.

## Known limitations
- The IDOR fix compares against a lesson-session value. A real application would
  take the identity from its authentication principal (for example Spring Security).
- Only the four lessons above were fixed. WebGoat contains many other
  intentionally vulnerable lessons that remain unchanged.
- No DAST stage (for example OWASP ZAP) yet.

## Secrets management (see report Section 2.5)
- No credentials are hardcoded in `docker-compose.yml` or any source file.
- Local development config lives only in `.env`, which is git-ignored.
- CI/CD secrets (`NVD_API_KEY`) are GitHub Actions encrypted secrets, read in the workflow via `${{ secrets.NVD_API_KEY }}` and never written to the repository.
- Gitleaks scans the full git history on every run.
