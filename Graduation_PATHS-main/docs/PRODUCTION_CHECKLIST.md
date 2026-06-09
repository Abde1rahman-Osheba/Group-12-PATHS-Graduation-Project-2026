# PATHS Production Deployment Checklist

> Use this checklist when deploying the PATHS platform to production.  
> Mark each item as complete before going live.

---

## Environment Configuration

- [ ] Set `APP_ENV=production` (or `PRODUCTION=true`)
- [ ] Set `DEBUG=false`
- [ ] Verify `CORS_ORIGINS` lists only the production frontend URL(s)
- [ ] Verify `allowed_hosts` for `TrustedHostMiddleware` includes your production domain(s)

## Secrets & Credentials

- [ ] Generate a strong random `SECRET_KEY` (e.g. `openssl rand -hex 32`) — never use the default
- [ ] Rotate `POSTGRES_PASSWORD` — do not use `change_me`
- [ ] Set `OPENROUTER_API_KEY` for scoring agent
- [ ] Set `SMTP_USERNAME` and `SMTP_PASSWORD` for email outreach
- [ ] Set `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` for OAuth (if using Google integration)
- [ ] Review `QDRANT_API_KEY` — set a real key if Qdrant Cloud is used
- [ ] Ensure no real secrets are committed — verify `.env` is in `.gitignore`

## Database

- [ ] Run `alembic upgrade head` to apply all pending migrations
- [ ] Verify PostgreSQL is reachable and connection strings are correct
- [ ] Verify Apache AGE graph exists (`paths_graph`)
- [ ] Verify Qdrant collections exist (`paths_candidates`, `paths_jobs`)

## Provider Configuration

- [ ] Disable mock providers — set `CANDIDATE_SOURCING_PROVIDER` to a real provider (e.g. `linkedin`) or disable it
- [ ] Set `SCORING_ALLOW_OFFLINE_FALLBACK=false`
- [ ] Verify `LLM_PROVIDER` is set appropriately (e.g. `openrouter`)
- [ ] If using Google Calendar: set `GOOGLE_CALENDAR_SERVICE_ACCOUNT_FILE` and `GOOGLE_CALENDAR_ID`

## Security Headers

- [ ] Verify `Strict-Transport-Security` header is present (non-dev environments only)
- [ ] Verify `X-Content-Type-Options: nosniff`
- [ ] Verify `X-Frame-Options: DENY`
- [ ] Verify `Referrer-Policy: strict-origin-when-cross-origin`
- [ ] Verify `Permissions-Policy: camera=(), microphone=(), geolocation=()`
- [ ] Verify `TrustedHostMiddleware` rejects requests with unexpected `Host` headers

## Rate Limiting

- [ ] Consider adding `SlowAPI` or a reverse-proxy-level rate limiter for login and API endpoints
- [ ] Verify `OUTREACH_SEND_RATE_LIMIT_PER_MINUTE` is set appropriately

## Reverse Proxy

- [ ] Place PATHS behind **nginx** or **Caddy** for TLS termination, rate limiting, and request buffering
- [ ] Configure TLS with a valid certificate (Let's Encrypt / Caddy auto-TLS)
- [ ] Ensure reverse proxy forwards `Host` header correctly

## Monitoring & Health

- [ ] Verify `/health` endpoint returns all services as healthy
- [ ] Verify `/health/databases` endpoint returns PostgreSQL, Apache AGE, and Qdrant status
- [ ] Set up external monitoring (e.g. UptimeRobot, Prometheus) to poll health endpoints
- [ ] Configure log shipping and alerting for `error`-level logs

## Scheduler & Background Jobs

- [ ] Set `ENABLE_SCHEDULER=true` if job importing is needed
- [ ] Verify `JOB_SCRAPER_SOURCE` is a production-safe feed (e.g. `remoteok_rss`)
- [ ] Ensure `JOB_SCRAPER_STUB=false` for real scraping

## Final Checks

- [ ] All `.env.example` files contain **placeholder values only** (no real secrets)
- [ ] Run `python -c "from app.main import app; print('ok')"` to verify the app boots
- [ ] Test a full login flow against the production environment
- [ ] Verify CORS allows the production frontend origin
- [ ] Confirm `SECRET_KEY` is not the default (`CHANGE-ME-TO-A-RANDOM-SECRET`)
