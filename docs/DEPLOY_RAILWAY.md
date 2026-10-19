# Deploying to Railway (trial or Free plan)

Railway builds the `Dockerfile` at the repo root; `railway.json` only pins the builder and restart
policy, and is read automatically. There is no Blueprint step and no infrastructure file to apply.

## What "free" means on Railway

Railway has no ongoing always-free tier for an app plus database. New accounts start on a **30-day
Trial** with a one-time **$5 credit**; when that ends (or the credit runs out) the account moves to
the **Free** plan with **$1 of credit per month**.

| Resource | Trial | Free |
|----------|-------|------|
| Projects | 2 | 1 |
| Services per project | 5 | 3 |
| Per service | 2 vCPU / 1 GB RAM | 1 vCPU / 0.5 GB RAM |
| Volume storage | 500 MB | 500 MB |
| Custom domains | 1 | 0 |
| Log history | 7 days | 3 days |

The app and its Postgres service fit comfortably within the service count. The constraint is
spend:

- Railway bills by usage while services are running: **$10/GB RAM per month** and **$20/vCPU per
  month**. A 0.5 GB app that is always on is therefore about $5/month at full memory usage, before
  Postgres. The Free plan's $1 credit will not keep this stack running 24/7; the $5 trial credit is
  a short demo budget, not a month of hosting.
- When credit runs out Railway stops the workloads. If you need this demo online for more than a
  short period, plan on the Hobby plan ($5/month minimum) or move to another host.
- Railway does not spin idle services down, so **stop or delete the project when you are done** to
  stop consuming credit.
- The Free plan cannot attach a custom domain; use the generated `*.up.railway.app` domain.
- Keep the service's memory allocation at or below 0.5 GB to stretch the trial credit (the demo
  runs in 512 MB).

Prerequisites: a Railway account. The dashboard flow needs your repo on GitHub
([shared setup](DEPLOYMENT.md#1-publish-the-repo)); the CLI flow can deploy straight from your
machine.

## 1. Option A — deploy from your machine

```bash
npm install -g @railway/cli
cd ~/wagtail-bakerydemo-demo
railway login

railway init --name bakerydemo          # creates an empty project and links this directory
railway add --database postgres         # adds a Postgres service

railway up                              # builds the Dockerfile; first build takes ~10 min
                                        # (the app crashes until the variables below exist)

railway service                         # select the app service, not Postgres
railway variables --set 'DATABASE_URL=${{Postgres.DATABASE_URL}}'
railway variables --set "DJANGO_SECRET_KEY=$(openssl rand -hex 32)"
railway variables --set 'DJANGO_LOAD_INITIAL_DATA=on'
railway variables --set 'DJANGO_ALLOWED_HOSTS=*'
railway variables --set 'PORT=8000'

railway up                              # redeploy if the variable change did not trigger one
railway domain                          # generates a public <*.up.railway.app> URL (port 8000)
```

If you already have a project (the Free plan allows only one), use `railway link` instead of
`railway init`.

## 2. Option B — deploy from the dashboard

1. **New Project → Deploy from GitHub repo** → pick your repo. Railway detects the `Dockerfile` at
   the repo root (`railway.json` pins `builder: DOCKERFILE`).
2. **New → Database → PostgreSQL**. A `Postgres` service appears and exposes `DATABASE_URL`.
3. On the app service, open **Variables → Raw editor** and add:

   ```
   DATABASE_URL=${{Postgres.DATABASE_URL}}
   DJANGO_SECRET_KEY=<openssl rand -hex 32>
   DJANGO_LOAD_INITIAL_DATA=on
   DJANGO_ALLOWED_HOSTS=*
   PORT=8000
   ```

4. **Settings → Networking → Generate Domain** and choose port `8000`.
5. Trigger a deploy (**Deployments → Deploy latest**) and open the generated URL.

On the Trial, lower the app service's memory under **Settings → Resources** (0.5 GB is enough for
the demo) so the credit lasts longer.

## 3. Notes for free accounts

- `${{Postgres.DATABASE_URL}}` is a reference variable, so it stays correct if the database
  connection details change.
- If `railway variables --set ...` targets the wrong service, run `railway service` first (or edit
  the variables in the dashboard).
- No custom domain is available on the Free plan; the generated Railway domain is the public URL.
- **Persisting uploads**: the Free plan includes one 500 MB volume. In the app service, add a
  volume with mount path `/code/bakerydemo/media` if uploads should survive redeploys. The seeded
  demo media is about 9 MB, so 500 MB is plenty for a demo; without a volume, media is ephemeral.
- **Running commands**: from a local checkout with dependencies installed, `railway run python
  manage.py <command>` runs against the linked project's variables, which is useful for migrations
  or `reset_admin_password`.
- To stream logs: `railway logs` (or **Deployments → View Logs**).
- Admin: `https://<generated-domain>/admin/` — demo login **`admin` / `changeme`**; change it
  immediately.
- Before deleting the project, export anything you want to keep (for example with `pg_dump` against
  `DATABASE_URL`). Railway deletes data from Trial/Free projects 30 days after expiry or cancellation.

## 4. Updating

GitHub-connected projects redeploy on push. CLI-flow projects need another `railway up`. Migrations
run automatically on each start from `docker-entrypoint.sh`; demo data is loaded only when the
database is empty.

## See also

- [Persist uploaded media](DEPLOYMENT.md#3-media-uploads-the-persistence-trade-off)
- [Troubleshooting checklist](DEPLOYMENT.md#4-troubleshooting-checklist)
- [Production hardening](DEPLOYMENT.md#5-production-hardening-beyond-the-demo)
