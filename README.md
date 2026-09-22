# fastapi-tasks

**Deploy a Python FastAPI app from a Dockerfile with a database and no server to run.** A task API in a container with no database driver: every task is one REST call to a managed [Back4app](https://www.back4app.com/) backend, and the validation rule lives there, not in Python. 80 lines of FastAPI, a 6-line Dockerfile, no connection string, no migrations.

Measured on September 15–16, 2026, on Back4app Containers: Deploy click → `DEPLOYMENT READY` in **86 s**, the first `200` from the URL **17 s before** the dashboard said Ready, the full create/read/update/validate/delete round trip passing 12 s later. Every number in the article comes from this exact code.

> **Read the article:** [Deploy a Python FastAPI App From a Dockerfile — With a Database, No Servers to Manage](https://www.back4app.com/blog/deploy-python-fastapi-dockerfile-database)

## What it does

Six routes in `main.py`: `POST /tasks`, `GET /tasks` (with `?done=true`), `GET`, `PATCH` and `DELETE` on `/tasks/{id}`, and `GET /healthz`. Pydantic validates the *shape* of a request; the request then becomes one `httpx` call to the backend's REST API with two headers (App ID and REST key). Swagger UI works unchanged at `/docs`.

Parse error codes are mapped to the status a client expects: `101 → 404`, `142 → 422`, `209 → 401`, anything else `→ 502`.

```
client ──HTTP──▶ ┌────────────────────────┐        ┌──────────────────────┐
                 │ container (python:3.12)│ ─REST─▶│ Back4app backend      │
                 │ FastAPI + httpx        │        │ Task + beforeSave     │
                 └────────────────────────┘        └──────────────────────┘
```

## The finding this repo exists for

`Field(min_length=1)` accepted a title of three spaces; three characters is three characters. The backend's `beforeSave` hook (`cloud/main.js`, 9 lines) trims the title, rejects an empty one with code `142`, and the container turns that into the same `422` Pydantic would have used. Keep the rule in the backend and a `curl`, the Python app and the next app you write all get the same answer.

## What we measured

| Measurement | Result |
|---|---|
| Deploy click → `DEPLOYMENT READY` (first deploy, free plan) | 86 s (46 s of it `pip install` + image build) |
| First `200` from the URL vs the dashboard's Ready line | 17 s **before** Ready — poll your own health route |
| `deploy-check.sh` (create, read, update, validate, delete) | all six passed 12 s after Ready |
| Free plan lifetime | 60 min from the start of the deploy, then `URL Expired` |
| Plan change to Shared ($5/mo) | redeployed on its own, same URL, 52 s |

## Files

- `main.py` — the container: models, routes, the Parse-error mapping, one shared `httpx.AsyncClient`.
- `cloud/main.js` — the backend rule, deployed as Cloud Code (`beforeSave("Task")`).
- `Dockerfile` — `python:3.12-slim`, `uvicorn main:app --port 8080`.
- `requirements.txt` — fastapi 0.115.6, uvicorn 0.34.0, httpx 0.28.1.
- `deploy-check.sh` — the full round trip against any deployment URL.

## Deploy your own

1. **Create a free account.** Sign up at [https://www.back4app.com/signup](https://www.back4app.com/signup). One account gives you both halves: **Build your Backend** (the Task class and its rule) and **Containers** (where the Dockerfile runs).
2. **Backend:** New App → Build your Backend. On Overview copy the App ID and the REST API key. **Cloud Code → main.js**: paste `cloud/main.js`, Deploy, then edit and deploy again (the first deploy on a fresh backend ships nothing); prove the hook with a request.
3. **Container:** push this repo to GitHub, then **Containers → New App → Deploy from GitHub**. The form detects the Dockerfile. Set `PARSE_APP_ID` and `PARSE_REST_KEY` as environment variables and the health check to `/healthz`. Deploy.
4. Verify: `./deploy-check.sh https://<your-app>.b4a.run`

The platform's health check only needs the port to answer, so a wrong key is not caught at deploy time. `deploy-check.sh` is the real health check. On the free plan the container's URL lives 60 minutes per deploy; for a permanent URL change the plan (Shared starts at $5/month as of September 2026).

## Run locally

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp .env.example .env      # PARSE_APP_ID, PARSE_REST_KEY
set -a; . ./.env; set +a
.venv/bin/uvicorn main:app --port 8000
```

## What the platform gives you

Containers build the Dockerfile, run the image behind HTTPS on a public URL and redeploy on push. The backend is a managed Parse Server with a database, REST and GraphQL APIs, Cloud Code and a dashboard where every Task is a row you can inspect. Documentation: [https://www.back4app.com/docs-containers](https://www.back4app.com/docs-containers) · [https://www.back4app.com/docs](https://www.back4app.com/docs).

## License

MIT
