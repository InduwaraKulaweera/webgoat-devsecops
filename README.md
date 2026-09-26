# IE3142 DevSecOps Pipeline — OWASP WebGoat

## Application
- **Selected app:** OWASP WebGoat (official all-in-one image, bundles WebGoat + WebWolf)
- **Stack:** Java / Spring Boot
- **Components:**
  1. `webgoat` — the app container, serving WebGoat on port 8080 and its WebWolf
     companion service on port 9090 (both ship inside the one official image), using
     WebGoat's own embedded database for accounts and lesson progress

> **Why not an external database?** An external PostgreSQL container was tried
> first. It failed: the official `webgoat/webgoat` image does not ship the
> PostgreSQL JDBC driver on its classpath, so `SPRING_DATASOURCE_URL` overrides
> fail with `Cannot load driver class: org.postgresql.Driver`. Making that work
> would require building a custom image just to add the driver jar, which isn't
> worth the added complexity here. The module's own Appendix A already confirms
> WebGoat's app + embedded database satisfies the two-component requirement, so
> this is the architecture used. Worth a line in the report's reflection section.

## Prerequisites
- Docker + Docker Compose installed
- No cloud account or Kubernetes needed — runs fully offline once images are pulled

## Setup (Week 1 checkpoint)

1. Clone this repository.
2. Copy the environment template:
   ```bash
   cp .env.example .env
   ```
3. Bring up WebGoat with one command:
   ```bash
   docker compose up -d
   ```
4. Give it 20-60 seconds to fully boot (Spring Boot startup), then verify:
   - WebGoat: http://127.0.0.1:8080/WebGoat
   - WebWolf: http://127.0.0.1:9090/WebWolf
5. Register a WebGoat account, log in, and open a lesson.
6. Confirm progress survives a restart:
   ```bash
   docker compose restart webgoat
   ```
   Log back in — your account and lesson progress should still be there, because
   `/home/webgoat/.webgoat` is mounted as a named Docker volume (`webgoat_data`),
   not left inside the ephemeral container filesystem.

## Tearing down
```bash
docker compose down          # stop containers, keep DB volume
docker compose down -v       # stop containers and wipe DB data
```

## Secrets management (see report Section 2.5)
- No credentials are hardcoded in `docker-compose.yml` or any source file.
- Local development secrets live only in `.env`, which is git-ignored.
- CI/CD pipeline secrets are provisioned via GitHub Actions encrypted secrets (see `.github/workflows/`).

## Notes
- `webgoat_db` deliberately has no `ports:` mapping — it is only reachable from
  the `backend` Docker network, not from the host. This is the trust boundary
  referenced in the architecture diagram and threat model.
