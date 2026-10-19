# Documentation index

| Document | Covers |
|----------|--------|
| [DEPLOYMENT.md](DEPLOYMENT.md) | Deploying this demo to **Render** and **Railway**: one-click Blueprint / CLI and dashboard flows, environment variables, demo-data seeding, media persistence (disk / volume / S3), Redis, troubleshooting, hardening checklist |
| [../readme.md](../readme.md) | Project overview and local setup (venv, Docker, Vagrant) |
| [../CONTRIBUTING.md](../CONTRIBUTING.md) | Making changes, linting/testing, exporting fixtures |

Deployment artifacts referenced by [DEPLOYMENT.md](DEPLOYMENT.md) live at the repo root:
`render.yaml`, `railway.json`, `Dockerfile`, `docker-entrypoint.sh`, and
`bakerydemo/settings/production.py`.

Notes:

- `readme.md`, `CONTRIBUTING.md` and `AGENTS.md` intentionally stay at the repo root, so the
  folder structure keeps matching `wagtail/bakerydemo` upstream (`git fetch upstream` merges stay
  painless).
- The full deployment guide is a single `DEPLOYMENT.md` to avoid duplicated drift between
  platforms; it is sectioned so you can link readers straight to the Render or Railway parts.
