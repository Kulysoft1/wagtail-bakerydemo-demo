# Deploying to Railway

Railway build/deploy for this repo is defined by the `Dockerfile` at the repo root, pinned with
`railway.json` (`builder: DOCKERFILE`). You deploy the app as one service and reference a Postgres
service's `DATABASE_URL` from it.

> Railway needs a paid plan (Hobby/Pro) or trial credit for app deploys. Postgres is a tiny add-on.

Prerequisites: your repo is on GitHub for the dashboard flow
([see the shared setup](DEPLOYMENT.md#1-publish-the-repo)); the CLI flow also works straight
from your machine.

## 1. Option A — deploy from your machine (no GitHub round-trip)

```bash
npm install -g @railway/cli
cd ~/wagtail-bakerydemo-demo
railway login

railway init --name bakerydemo          # creates an empty project, links your local dir
railway add --database postgres         # adds a "Postgres" service and its DATABASE_URL var

railway up                              # creates + builds the app service; first build takes ~10 min
                                        # (the first boot crashes until the variables below exist)

railway service                         # (re)select the app service from the project
railway variables --set "DATABASE_URL=${{Postgres.DATABASE_URL}}"
railway variables --set "DJANGO_SECRET_KEY=$(openssl rand -hex 32)"
railway variables --set "DJANGO_LOAD_INITIAL_DATA=on"
railway variables --set "DJANGO_ALLOWED_HOSTS=*"
railway variables --set "PORT=8000"

railway up                              # redeploy if the variable change did not trigger one
railway domain                          # generates a public <*.up.railway.app> URL (port 8000)
```

## 2. Option B — deploy from the dashboard (GitHub)

1. **New Project → Deploy from GitHub repo** → pick your repo.
   Railway detects the `Dockerfile` at the repo root (`railway.json` also pins `builder: DOCKERFILE`).
2. **New → Database → PostgreSQL** → a "Postgres" service appears (it exposes `DATABASE_URL`).
   No linking needed — reference it from the app service via `${{Postgres.DATABASE_URL}}` (step 3).
3. On the app service, go to **Variables** → *Raw editor* and add:

```
DATABASE_URL=${{Postgres.DATABASE_URL}}
DJANGO_SECRET_KEY=<openssl rand -hex 32>
DJANGO_LOAD_INITIAL_DATA=on
DJANGO_ALLOWED_HOSTS=*
PORT=8000
```

4. **Settings → Networking → Generate Domain** → pick port `8000`.
5. Trigger a deploy (**Deployments → Deploy latest**) and open the generated URL.

## Notes

- `${{Postgres.DATABASE_URL}}` is a *reference variable* — it stays correct if the database
  connection details change.
- If `railway variables --set ...` targets the wrong service, run `railway service` first to switch
  the linked service (or set variables in the dashboard).
- To stream logs: `railway logs` (or *Deployments → View Logs*).
- Admin: `https://<project>.up.railway.app/admin/`.

## 3. Updating

GitHub-connected projects: push → Railway auto-deploys. CLI-flow projects: run `railway up` again.
Migrations run automatically on each start (from `docker-entrypoint.sh`).

## See also

- [Persist uploaded media](DEPLOYMENT.md#3-media-uploads-the-persistence-trade-off)
  (volume at `/code/bakerydemo/media`) — this deployment guide works for both platforms.
- [Troubleshooting checklist](DEPLOYMENT.md#4-troubleshooting-checklist)
- [Production hardening](DEPLOYMENT.md#5-production-hardening-beyond-the-demo)
