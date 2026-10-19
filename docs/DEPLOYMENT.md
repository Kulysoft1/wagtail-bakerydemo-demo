# Deploying the demo

This repo ships with everything needed to deploy the Wagtail bakery demo as-is:

| What                                   | Where                        |
|----------------------------------------|------------------------------|
| Docker image (Python 3.12, uWSGI, static + media serving) | `Dockerfile`, `etc/uwsgi.ini` |
| Production settings (reads everything from env vars)      | `bakerydemo/settings/production.py` |
| Render Blueprint (web service + Postgres)                 | `render.yaml`                          |
| Railway config (Dockerfile build)                         | `railway.json`                         |
| Wait-for-DB, migrate, seed demo data once                 | `docker-entrypoint.sh`                 |

Default stack: **managed Postgres** (via `DATABASE_URL` from `dj-database-url`), **uWSGI** on `$PORT`,
**WhiteNoise** for static assets, and the full demo content (pages, users, images) loaded on the
first boot.

Demo admin credentials: **`admin` / `changeme`** at `/admin/` — change this right after deploying.

---

## 1. Publish the repo

Both platforms deploy from a Git repository (Railway can also deploy straight from your machine).

```bash
# Create an empty repo on GitHub (web UI, or with the gh CLI):
gh repo create wagtail-bakerydemo-demo --public --source=. --push

# ...or manually:
cd ~/wagtail-bakerydemo-demo
git remote add origin https://github.com/<your-account>/wagtail-bakerydemo-demo.git
git push -u origin main
```

The `upstream` remote still points at `wagtail/bakerydemo`, so you can pull demo updates with:

```bash
git fetch upstream && git merge upstream/main
```

---

## 2. Pick your platform

| Platform | Approach | Guide |
|----------|----------|-------|
| **Render** | Blueprint: `render.yaml` provisions the web service + Postgres in one click | [DEPLOY_RENDER.md](DEPLOY_RENDER.md) |
| **Railway** | Dockerfile build + Postgres service referenced as a variable | [DEPLOY_RAILWAY.md](DEPLOY_RAILWAY.md) |

Both platforms produce the same result — a Dockerized app, a managed Postgres, and the demo site
seeded on first boot.

---

## 3. Media uploads: the persistence trade-off

By default, media (uploaded images/documents) lives on the **container filesystem**, which resets on
every redeploy or instance restart:

- The **seeded demo content re-loads itself** on boot when the database is empty (`docker-entrypoint.sh`
  checks for existing content first, so restarts won't duplicate media).
- **Admin-uploaded media is ephemeral** — great for demos, useless for real content.

To persist uploads:

### Render: persistent disk

Web service → *Settings → Disks* → add a Disk (Starter plan+, e.g. 1 GB) with **Mount path**

```
/code/bakerydemo/media
```

(MEDIA_ROOT inside the container; mounted under the working directory.)

### Railway: volume

App service → *Volumes* (or Settings → Volumes) → *New Volume* with **Mount path**

```
/code/bakerydemo/media
```

### S3 / object storage (optional, both platforms)

`bakerydemo/settings/production.py` auto-switches media to S3 when `AWS_STORAGE_BUCKET_NAME` is set.
Add (on either platform's env tab):

```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_STORAGE_BUCKET_NAME=bakerydemo-media
AWS_REGION=eu-central-1               # or AWS_S3_REGION_NAME for custom domains
AWS_S3_CUSTOM_DOMAIN=...              # optional CDN alias
```

Notes:
- Requires `django-storages` + `boto3`, both already in `requirements/production.txt`.
- This demo sets `AWS_QUERYSTRING_AUTH = False` with `AWS_DEFAULT_ACL = private` — so make the
  bucket **publicly readable** (bucket policy) or use a CDN in front.
- Anything already in Postgres/file-media won't migrate automatically; upload again on the new media.

**Redis (optional on both platforms)** — this stack happily uses Django's local-memory cache. If you
provision a Redis anywhere, set `REDIS_URL=...` on the app's env vars; production.py automatically
enables `django-redis` for cache/renditions.

---

## 4. Troubleshooting checklist

| Symptom                                                      | Fix                                                                                                          |
|--------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Logs scream `Postgres is unavailable - sleeping`             | `DATABASE_URL` not set or Postgres not ready yet. Verify the database is running and the variable is a *reference*. |
| Render deploys but "502" or long startup                     | Free instance waking up after inactivity; wait ~1 minute.                                                    |
| `DisallowedHost at /` / admin POST gives CSRF error          | Set `DJANGO_ALLOWED_HOSTS` to your real hostname (comma separated), or leave `*` for a throwaway demo           |
| Logs show `Missing staticfiles manifest entry`               | Static is collected at image build time — a rebuild fixes it (`Deploy latest`).                              |
| `relation "..." does not exist`                              | Migrations somehow skipped. One-off shell: Render *Shell* (paid instances), or on Railway connect a local `railway run python manage.py migrate` using the database's public `DATABASE_URL`. |
| Nothing but a blank home page                                | Set `DJANGO_LOAD_INITIAL_DATA=on` and redeploy with an empty database (drop the Postgres or start a new project) |
| Change the demo admin password                               | Admin UI → Users → admin, or one-off shell: `python manage.py reset_admin_password` (uses `ADMIN_PASSWORD` env) |

---

## 5. Production hardening (beyond the demo)

The deployment is intentionally demo-friendly:

- `DJANGO_ALLOWED_HOSTS=*` → set it to your real hostname(s).
- Add a database backup schedule on Render / enable Postgres point-in-time restore on Railway.
- Rotate `DJANGO_SECRET_KEY` (on Render: *Environment → Change value*).
- Set `PRIMARY_HOST=<yourdomain>` so Wagtail's notification emails build correct URLs.
- `heroku.yml` / `Procfile` (Heroku deployment path) is untouched if you need it later.
