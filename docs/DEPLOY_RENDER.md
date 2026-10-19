# Deploying to Render (free tier)

This walkthrough uses only the Render dashboard. It assumes a **free Render account**, which
does not include Blueprint access, so the `render.yaml` at the repo root cannot be applied.
The Blueprint file is kept for accounts that do have access; the manual steps below create the
same two resources it describes: a free Postgres database and a free Docker web service.

## Free-tier limits that shape this guide

| Limit | What it means here |
|-------|--------------------|
| No Blueprint access | Create the database and web service by hand (steps 1-2). |
| 0.1 CPU / 512 MB RAM | Enough for the demo, but builds and admin pages are slow. |
| Sleeps after 15 minutes idle | The next request shows Render's loading page and takes about a minute to wake the service. |
| 750 free instance hours per workspace per month | One service running non-stop uses most of the allowance; spun-down time does not count. Exceeding it suspends all free services until the next month. |
| Ephemeral filesystem, no persistent disks | Uploaded media is lost on every restart, redeploy, or spin-down. Seeded demo images are baked into the image and reload only when the database is empty. |
| One free Postgres per workspace, 1 GB, expires after 30 days | The database becomes inaccessible at expiry. After a 14-day grace period Render deletes it and its data. There are no backups. |
| No Shell and no one-off jobs | Run management commands from your machine against the database's external URL when needed. |
| Bandwidth and build-minute caps | Heavy traffic or repeated rebuilds can suspend free services/builds until the next month. |

Prerequisites: your repo is on GitHub ([see the shared setup](DEPLOYMENT.md#1-publish-the-repo))
and you have a Render account. No credit card is required for free instances.

Pick a region (for example Frankfurt) and use the **same region** for the database and the web
service so they can talk over Render's private network.

## 1. Create the free Postgres database

1. Render dashboard → **New → Postgres**.
2. **Name**: `bakerydemo-db`.
3. **Region**: the region you will use for the web service.
4. **Compute plan**: **Free** (1 GB storage).
5. Click **Create Database** and wait until the status is *Available*.
6. Open the database's **Connect** menu (top right) and copy the **Internal Database URL**.
   Keep it for the next step.

> Only one free Postgres can be active per workspace. If you already have one, reuse it or
> delete it before creating another.

## 2. Create the free web service

1. Render dashboard → **New → Web Service** → **Build and deploy from a Git repository**.
2. Connect GitHub and choose your fork of this repo.
3. Fill in:
   - **Name**: `bakerydemo`
   - **Region**: same as the database
   - **Branch**: `main`
   - **Language**: **Docker**
   - **Dockerfile Path**: `Dockerfile` (the repo root)
   - **Instance Type**: **Free**
4. Add these environment variables on the **Environment** tab (or under *Advanced*):

   | Variable | Value |
   |----------|-------|
   | `DATABASE_URL` | the **Internal Database URL** copied in step 1 |
   | `DJANGO_SECRET_KEY` | output of `openssl rand -hex 32` |
   | `DJANGO_ALLOWED_HOSTS` | `*` |
   | `DJANGO_LOAD_INITIAL_DATA` | `on` |

   Do **not** set `PORT`. Render injects it, and `etc/uwsgi.ini` already listens on `$PORT`.
5. Under **Advanced**, set **Health Check Path** to `/` and leave auto-deploy enabled.
6. Click **Create Web Service**.

The first build takes roughly 5-10 minutes (Python image + dependencies). In the **Logs** tab you
should see the entrypoint wait for Postgres, run migrations, load the demo fixture once, and then
start uWSGI.

## 3. Open the site

- Site: `https://<service-name>.onrender.com`
- Admin: `https://<service-name>.onrender.com/admin/`
- Demo login: **`admin` / `changeme`** — change it immediately (see below).

If the first request arrives while the service is asleep, Render shows a loading page for about a
minute while it spins up.

## 4. Free-tier housekeeping

**Keeping uploads.** Free web services cannot attach a persistent disk. The demo images are part
of the image and are re-created when the database is seeded, but anything uploaded through the
admin disappears on the next restart or deploy. If uploads must survive, use S3-compatible object
storage instead (see [media uploads](DEPLOYMENT.md#3-media-uploads-the-persistence-trade-off)).

**Before the 30-day database expiry.** Either upgrade the database to a paid compute plan, or take
a dump through the **External Database URL** and restore it into a fresh free database. Export
before day 30: once expired, the database is inaccessible and is deleted after the 14-day grace
period.

**Management commands without a Shell.** Free services do not get Shell access or one-off jobs.
From a local checkout with the project dependencies installed (see the [installation
guide](../readme.md#installation)), point Django at the database's external URL:

```bash
export DATABASE_URL='<External Database URL>'
export DJANGO_SETTINGS_MODULE=bakerydemo.settings.production

python manage.py migrate
ADMIN_PASSWORD='<new-password>' python manage.py reset_admin_password
```

External connections require TLS; keep `sslmode=require` in the URL if Render includes it.

## 5. Updating

Push to your GitHub repo and Render redeploys automatically. Migrations run on every start from
`docker-entrypoint.sh`; demo data is loaded only when the database is empty. Saving an environment
variable also triggers a new deploy. Expect uploads to be lost on each deploy.

## 6. If your account does have Blueprint access

Accounts with Blueprint access can skip steps 1-2 and apply the included `render.yaml`:

- Dashboard → **New → Blueprint** → pick your repo → **Apply**, or
- open `https://render.com/deploy?repo=https://github.com/<your-account>/wagtail-bakerydemo-demo`.

It creates the same free web service and free database with the environment variables already
wired up. This is optional — the manual flow above is the supported path for free accounts.

## See also

- [Persist uploaded media](DEPLOYMENT.md#3-media-uploads-the-persistence-trade-off)
- [Troubleshooting checklist](DEPLOYMENT.md#4-troubleshooting-checklist)
- [Production hardening](DEPLOYMENT.md#5-production-hardening-beyond-the-demo)
