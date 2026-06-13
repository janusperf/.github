# Janus: Run the whole stack

How to bring up every part of Janus, locally and in production, step by step.

---

## 1. The pieces (and their ports)

| Component      | Path         | Tech                      | Local URL                | Talks to                          |
| -------------- | ------------ | ------------------------- | ------------------------ | --------------------------------- |
| **Studio API** | `studio/api` | FastAPI (Python 3.13)     | `http://127.0.0.1:8000`  | Supabase, Redis, S3/MinIO, LLMs   |
| **Studio Web** | `studio/web` | Vue 3 + Vite              | `http://localhost:5173`  | Studio API                        |
| **Agent**      | `agent`      | FastAPI + asyncio         | `http://127.0.0.1:8765`  | Studio API, the device            |
| **Desktop**    | `desktop`    | Electron + Vue 3 + Vite   | `http://localhost:3000`  | Agent                             |
| **CLI**        | `cli`        | Node + Ink (TS)           | terminal                 | Agent, Studio API                 |
| **Landing**    | `landing`    | Vue 3 + Vite              | own Vite port            | (static marketing site)           |
| **domain**     | `domain`     | shared Python package     | (installed into venvs)   | imported by Agent + Studio API    |

Local infra (Docker, from `studio/docker-compose.yml`):

| Service     | Port(s)        | Notes                                  |
| ----------- | -------------- | -------------------------------------- |
| **Redis**   | `6379`         | cache / outbox                         |
| **MinIO**   | `9000` / `9001`| S3-compatible storage / web console    |
| **Mailpit** | `1025` / `8025`| local SMTP / web inbox for emails      |

### How requests flow

```
Desktop ─┐
         ├─▶ Agent (127.0.0.1:8765) ─▶ Studio API (127.0.0.1:8000) ─▶ Supabase (Postgres + Auth)
CLI  ────┘                                                          ├─▶ Redis
                                                                    └─▶ S3 / MinIO
Studio Web ───────────────────────────▶ Studio API (127.0.0.1:8000)
```

> Postgres and Auth are **not** in Docker. Even locally they come from a **Supabase** project (use a separate dev project). Docker only gives you Redis, MinIO and Mailpit.

---

## 2. Prerequisites (install once)

- **Python 3.13+**
- **Node.js 20+** (covers Studio Web, Desktop, CLI and Landing)
- **Docker** (for Redis, MinIO, Mailpit)
- A **Supabase** project for Postgres + Auth (one for dev, one for prod)
- A **Gemini API key** (default LLM provider). OpenAI / Anthropic keys are optional.
- For real device testing: **Android** needs `adb` (Desktop fetches it); **iOS** needs macOS + Xcode.

---

## 3. Local: first-time setup (once per component)

Do this once. After that, `./resources/scripts/dev-all.sh` brings everything up together (section 4).

### 3.1 Infra (Docker)

```bash
cd studio
docker-compose up -d        # redis + minio + mailpit
```

MinIO console: http://localhost:9001 (user/pass `minioadmin` / `minioadmin`). Mailpit inbox: http://localhost:8025.

### 3.2 Studio API

```bash
cd studio/api
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt      # also installs ../../domain editable
pip install -r requirements-llms.txt
cp .env.example .env                      # then fill SUPABASE_* and GEMINI_API_KEY
alembic upgrade head                      # create the schema
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Edit `studio/api/.env`: set `SUPABASE_URL`, `SUPABASE_AUTH_BASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_JWT_SECRET` (from your Supabase dev project) and `GEMINI_API_KEY`. The Redis / MinIO / Mailpit defaults already match Docker.

### 3.3 Agent

```bash
cd agent
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt      # pulls ../domain editable
cp .env.example .env                      # JANUS_STUDIO_API_URL already points at :8000
uvicorn service:app --reload --port 8765
```

### 3.4 Studio Web

```bash
cd studio/web
cp .env.example .env                      # VITE_API_BASE_URL=/api (Vite proxies to :8000)
npm install
npm run dev                               # http://localhost:5173
```

### 3.5 Desktop

```bash
cd desktop
npm install
cp .env.example .env                      # points at the agent (:8765) and Studio Web (:5173)
npm run fetch:scrcpy                       # Android screen recorder
npm run fetch:toolchain                    # Appium, JDK, Maven, Node, etc.
npm run dev                                # launches the Electron app (Vite on :3000)
```

### 3.6 CLI

```bash
cd cli
npm install
cp .env.example .env                       # points at local Studio API + Web
npm run dev                                 # opens the Ink TUI
```

### 3.7 Landing (optional, marketing site)

```bash
cd landing
npm install
cp .env.example .env
npm run dev
```

---

## 4. Local: the fast way (daily)

Once each component is set up (venvs created, `npm install` done), start everything with one command from the repo root:

```bash
./resources/scripts/dev-all.sh
```

It frees ports `8000 8765 5173 3000`, starts the Docker infra, runs `alembic upgrade head`, makes sure the shared `domain` package is installed in both venvs, then launches **Studio API + Agent + Studio Web + Desktop** together. Stop it with `Ctrl-C`.

- Skip the Docker + migrations step: `SKIP_DB=1 ./scripts/dev-all.sh`
- The CLI and Landing are not started by this script; run them separately when needed.

---

## 5. Production

Each component ships independently. Use a separate Supabase project, an Upstash Redis, and an S3 (or MinIO) bucket for prod.

### Studio API → Railway (or Kubernetes)

Railway builds `studio/api/Dockerfile` and reads `studio/api/railway.toml` (runs `alembic upgrade head` before each release, healthcheck on `/health`).

1. New service from this repo, **Root Directory** = `studio/api`.
2. Add a `GH_DOMAIN_TOKEN` (GitHub token with read access to the private `domain` repo) so the build can install `studio-domain`.
3. Add a Redis plugin and set `REDIS_URL = ${{Redis.REDIS_URL}}`.
4. Set the rest of the variables from `studio/api/.env.example` (DB + Auth from Supabase, `S3_*`, `GEMINI_API_KEY`, `CORS_ORIGINS` = your Studio Web URL). Do **not** set `PORT` (Railway sets it).

Kubernetes alternative: starter manifests live in `deploy/k8s/` (see its README for the Secret/ConfigMap bootstrap).

### Studio Web → Vercel

```bash
cd studio/web && npm run build      # static output in dist/
```

Deploy `dist/` to Vercel and set `VITE_API_BASE_URL=https://<your-api-host>/api`. In Supabase Auth, add the Studio Web URL and the OAuth redirect URLs.

### Landing → static host

```bash
cd landing && npm run build         # deploy dist/ to any static host
```

### Desktop → installers

Agents and the device toolchain are bundled into the binary:

```bash
cd desktop
npm run electron:build:mac          # or :win / :linux
```

### CLI → packaged binary (CI/CD)

The headless CLI embeds a frozen agent so runners need no Python. Build the agent for the target arch, embed, and pack:

```bash
(cd agent && ./freeze.sh linux)  && (cd cli && npm run package:df:linux)   # Android runners
(cd agent && ./freeze.sh darwin) && (cd cli && npm run package:df:mac)     # iOS runners (macOS)
```

In CI, authenticate with a Studio token and run a config:

```bash
janus init --token $JANUS_SERVICE_TOKEN
janus run --config janus.config.yml
```

---

## 6. Common gotchas

- **`dev-all.sh` does nothing useful on a fresh checkout.** It assumes the venvs and `node_modules` already exist. Do section 3 first.
- **Studio API can't connect to the DB.** The DB is Supabase, not Docker. Check `SUPABASE_URL`; for dev prefer the transaction pooler on port `6543` with `pgbouncer=true` (see `studio/README.md`).
- **Auth/login fails.** `SUPABASE_JWT_SECRET` must match your Supabase project, and the redirect URLs must be configured in Supabase Auth.
- **CORS errors from Desktop/Web.** `CORS_ORIGINS` in `studio/api/.env` must include `http://localhost:3000` and `http://localhost:5173` (the defaults already do).
- **Port already in use.** `dev-all.sh` frees `8000/8765/5173/3000` on start; otherwise `lsof -ti tcp:<port> | xargs kill`.

For deeper per-component detail, see each subproject's own README: [`agent/README.md`](agent/README.md), [`studio/README.md`](studio/README.md), [`desktop/README.md`](desktop/README.md), [`cli/README.md`](cli/README.md), and the specs under [`docs/`](docs/).
