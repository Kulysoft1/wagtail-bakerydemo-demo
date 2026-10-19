# Documentation index

| Document | Covers |
|----------|--------|
| [DEPLOYMENT.md](DEPLOYMENT.md) | Shared overview: what the deployment stack includes, publishing the repo, and everything common to both platforms — media persistence (disk / volume / S3), Redis, troubleshooting, hardening, free-tier trade-offs |
| [DEPLOY_RENDER.md](DEPLOY_RENDER.md) | Deploying to **Render** on a free account: dashboard-only setup (no Blueprint), Postgres + Docker web service, environment variables, free-tier limits |
| [DEPLOY_RAILWAY.md](DEPLOY_RAILWAY.md) | Deploying to **Railway** on the Trial or Free plan: CLI (`railway up`) and dashboard flows, reference variables, credit limits |
| [../readme.md](../readme.md) | Project overview and local setup (venv, Docker, Vagrant) |
| [../CONTRIBUTING.md](../CONTRIBUTING.md) | Making changes, linting/testing, exporting fixtures |

Deployment artifacts referenced by all guides live at the repo root:
`render.yaml`, `railway.json`, `Dockerfile`, `docker-entrypoint.sh`, and
`bakerydemo/settings/production.py`.

Notes:

- `readme.md`, `CONTRIBUTING.md` and `AGENTS.md` intentionally stay at the repo root, so the
  folder structure keeps matching `wagtail/bakerydemo` upstream (`git fetch upstream` merges stay
  painless).
- `DEPLOYMENT.md` holds all shared content (media persistence, troubleshooting, hardening); the
  per-platform guides (`DEPLOY_RENDER.md`, `DEPLOY_RAILWAY.md`) only contain their own deploy
  steps. If you change shared behavior, update `DEPLOYMENT.md` and keep platform files thin.
