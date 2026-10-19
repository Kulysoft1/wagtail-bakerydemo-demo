# Deploying to Render (free tier)

This guide covers deploying the bakerydemo to Render on a **free account**.

## Free Render accounts support Blueprints (with one database)

Unlike what older docs may say, Render's free tier **does** support Blueprints. The only hard limit that matters here is:

| Limit | What it means here |
|-------|--------------------|
| One free Postgres per workspace | A workspace can have only **one** free database at a time. This blueprint creates exactly one, so delete any existing free Postgres before applying it. |
| Blueprint supported on free tier | Yes — `render.yaml` can be applied with one-click. |
| 0.1 CPU / 512 MB RAM | Enough for the demo, but migrations and seeding the full demo fixture are slow on the free instance. |
| Sleeps after 15 minutes idle | The next request shows Render's loading page and takes ~1 minute to wake the service. |
| 750 free instance hours per workspace per month | One service running non-stop uses most of the allowance; spun-down time does not count. Exceeding it suspends all free services until the next month. |
| Ephemeral filesystem, no persistent disks | Uploaded media is lost on every restart, redeploy, or spin-down. Seeded demo images are baked into the image and reload only when the database is empty. |
| Free database expires after 30 days | The database becomes inaccessible at expiry. After a 14-day grace period Render deletes it and its data. There are no backups. |
| No Shell and no one-off jobs | Run management commands from your machine against the database's external URL when needed. |

Prerequisites: your repo is on GitHub ([see the shared setup](DEPLOYMENT.md#1-publish-the-repo)) and you have a Render account. No credit card is required for free instances.

## Option A — one-click Blueprint (recommended)

The `render.yaml` at the repo root creates the full stack: a free Docker web service and a free Postgres database, with all environment variables wired up.

1. Delete any existing free Postgres in your Render workspace (free tier allows only one).
2. Apply the blueprint:

   - Render dashboard → **New → Blueprint** → pick your repo → **Apply**, or
   - open `https://render.com/deploy?repo=https://github.com/<your-account>/wagtail-bakerydemo-demo`.

3. Render reads `render.yaml` and creates the services. First deploy sequence (watch the **Logs** tab):
   the entrypoint waits for Postgres → `migrate` → seeds the full demo site once → uWSGI starts.

4. Open `https://<service-name>.onrender.com` (site) and `/admin/` (Wagtail admin).

Render's first build takes roughly 5-10 minutes (Python image + dependency install). On the free 512 MB instance, migrations and demo seeding can be slow; if the health check needs more time, retry **Deploy latest**.

## Option B — manual dashboard flow

Use this if you prefer to create resources by hand, or if the one-database limit blocks the blueprint.

### 1. Create the free Postgres database

1. Render dashboard → **New → Postgres**.
2. **Name**: `bakerydemo-db`.
3. **Region**: pick one (for example Frankfurt).
4. **Compute plan**: **Free** (1 GB storage).
5. Click **Create Database** and wait until the status is *Available*.
6. Open the database's **Connect** menu (top right) and copy the **Internal Database URL**.

> Only one free Postgres can be active per workspace. If you already have one, reuse it or delete it before creating another.

### 2. Create the free web service

1. Render dashboard → **New → Web Service** → **Build and deploy from a Git repository**.
2. Connect GitHub and choose your fork of this repo.
3. Fill in:
   - **Name**: `bakerydemo`
   - **Region**: same as the database
   - **Branch**: `main`
   - **Language**: **Docker**
   - **Dockerfile Path**: `Dockerfile` (the repo root)
   - **Instance Type**: **Free**
4. Add these environment variables on the **Environment** tab:

   | Variable | Value |
   |----------|-------|
   | `DATABASE_URL` | the **Internal Database URL** copied in step 1 |
   | `DJANGO_SECRET_KEY` | output of `openssl rand -hex 32` |
   | `DJANGO_ALLOWED_HOSTS` | `*` |
   | `DJANGO_LOAD_INITIAL_DATA` | `on` |

   Do **not** set `PORT`; Render injects it and `etc/uwsgi.ini` listens on `$PORT`.
5. Under **Advanced**, set **Health Check Path** to `/` and leave auto-deploy enabled.
6. Click **Create Web Service**.

### 3. Open the site

- Site: `https://<service-name>.onrender.com`
- Admin: `https://<service-name>.onrender.com/admin/`
- Demo login: **`admin` / `changeme`** — change it immediately (see below).

If the first request arrives while the service is asleep, Render shows a loading page for about a minute while it spins up.

## How this maps to the local "Setup with venv" steps

The local venv flow ([readme.md#setup-with-venv](../readme.md#setup-with-venv)) does:

| Local venv step | Render equivalent |
|-----------------|-------------------|
| `python -m venv .venv` / `uv venv .venv` | Dockerfile builds a venv with `uv` (`ENV VIRTUAL_ENV=/venv`) |
| `pip install -r requirements/development.txt` | Dockerfile runs `uv pip install -r /requirements/production.txt` |
| `cp .env.example .env` / `cp local.py.example local.py` | Not needed on Render — set the same values as environment variables instead |
| `./manage.py migrate` | `docker-entrypoint.sh` runs `migrate --noinput` on every start |
| `./manage.py load_initial_data` | `docker-entrypoint.sh` seeds demo data once when the DB is empty (`DJANGO_LOAD_INITIAL_DATA=on`) |
| `./manage.py runserver` | `CMD ["uwsgi", "/code/etc/uwsgi.ini"]` serves the app on `$PORT` |

The blueprint sets `DJANGO_LOAD_INITIAL_DATA=on` so demo content loads automatically, matching `./manage.py load_initial_data`. After deploy, log into the admin and change the demo password.

## Free-tier housekeeping

**Keeping uploads.** Free web services cannot attach a persistent disk. The demo images are part of the image and are re-created when the database is seeded, but anything uploaded through the admin disappears on the next restart or deploy. If uploads must survive, use S3-compatible object storage instead (see [media uploads](DEPLOYMENT.md#3-media-uploads-the-persistence-trade-off)).

**Before the 30-day database expiry.** Either upgrade the database to a paid compute plan, or take a dump through the **External Database URL** and restore it into a fresh free database. Export before day 30: once expired, the database is inaccessible and is deleted after the 14-day grace period.

**Management commands without a Shell.** Free services do not get Shell access or one-off jobs. From a local checkout with the project dependencies installed (see the [installation guide](../readme.md#installation)), point Django at the database's external URL:

```bash
export DATABASE_URL='<External Database URL>'
export DJANGO_SETTINGS_MODULE=bakerydemo.settings.production

python manage.py migrate
ADMIN_PASSWORD='<new-password>' python manage.py reset_admin_password
```

External connections require TLS; keep `sslmode=require` in the URL if Render includes it.

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|--------------------|
| Build fails during `collectstatic` | The Dockerfile builds static assets at image-build time with a dummy `DATABASE_URL`. If your repo has custom static files that fail processing, the build fails. Rebuild after fixing static files. |
| Deploy hangs / health check times out | Free 512 MB instance is slow to run migrations + seed demo data + start uWSGI. Wait, then click **Deploy latest** again. |
| "502" or long startup on first visit | Free instance woke up from spin-down; wait ~1 minute. |
| Database unreachable after ~30 days | Free Postgres expired. Upgrade it or restore a dump into a fresh free database before the 14-day grace period ends. |
| Blueprint apply fails with a database error | Your workspace already has a free Postgres (free tier allows only one). Delete it, then re-apply the blueprint. |
| `DisallowedHost` / CSRF error on admin POST | Set `DJANGO_ALLOWED_HOSTS` to your real hostname, or leave `*` for a throwaway demo. |

## Updating

Push to your GitHub repo and Render redeploys automatically. Migrations run on every start from `docker-entrypoint.sh`; demo data is loaded only when the database is empty. Saving an environment variable also triggers a new deploy. Expect uploads to be lost on each deploy.

## See also

- [Persist uploaded media](DEPLOYMENT.md#3-media-uploads-the-persistence-trade-off)
- [Troubleshooting checklist](DEPLOYMENT.md#4-troubleshooting-checklist)
- [Production hardening](DEPLOYMENT.md#5-production-hardening-beyond-the-demo)
