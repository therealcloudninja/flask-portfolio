# Flask Portfolio — My Scratch Page

A small portfolio website built with **Python, Flask, and SQLite**, containerized with **Docker**, deployed to **Kubernetes**, and delivered through a fully automated **CI/CD pipeline in Azure DevOps**.

🔗 **Live site (PythonAnywhere):** https://henryylim01.pythonanywhere.com
🔗 **Portfolio page:** https://henryylim01.pythonanywhere.com/portfolio/

Built as Mini Project A and B for the Cloud Support & DevOps Bootcamp at Generation Singapore, to practice Flask fundamentals, database-backed apps, authentication, and using Git as source control — later extended with a full CI/CD project: containerization, private registry hosting, Kubernetes deployment, and Azure Pipelines automation.

---

## What it does

- Displays a "Scratch Page" with a list of comments stored in a database
- Lets **logged-in users only** add new comments, enforced on both the frontend and backend
- Has a login/logout system with multiple user accounts, each with a securely hashed password
- Has a separate Portfolio/Introduction page linking back to the Scratch Page
- Ships as a Docker image, deployable to any Kubernetes cluster, with every commit automatically built, pushed, and deployed via CI/CD

---

## Tech stack

| Piece | Tool |
|---|---|
| Backend | Python 3.10, Flask |
| Auth | Flask-Login, Werkzeug password hashing |
| Database | SQLite, accessed via Flask-SQLAlchemy |
| Containerization | Docker |
| Container registry | Docker Hub (private repository) |
| Orchestration | Kubernetes (Minikube, local) |
| CI/CD | Azure DevOps Pipelines, self-hosted agent (WSL) |
| Hosting (original) | PythonAnywhere (free "Beginner" tier) |
| Version control | Git + GitHub |

---

## CI/CD & Kubernetes deployment

This app was extended with a full CI/CD project: **CI/CD for a Flask App**, covering containerization, private registry hosting, Kubernetes manifests, and Azure DevOps pipeline automation.

### Architecture

```
Commit pushed to GitHub
        │
        ▼
Azure DevOps detects the push (CI trigger)
        │
        ▼
Self-hosted agent (WSL, on my laptop) picks up the job
        │
        ├─ 1. docker build   → tags image with $(Build.BuildNumber)
        ├─ 2. docker push    → pushes to private Docker Hub repo
        ├─ 3. kubectl set image → rolling update on the Kubernetes Deployment
        └─ 4. cleanup        → clears the agent's workspace
        │
        ▼
Kubernetes (Minikube) pulls the new image (authenticated via imagePullSecret)
and rolls out a new pod, replacing the old one with zero pipeline downtime
```

### Docker

- `Dockerfile` builds the app on `python:3.10-slim`, installs dependencies, and runs `flask_app.py` on port 5000
- Images are tagged with the Azure Pipelines build number (e.g. `20260829.5`) — **never `:latest`** (see [Why not `:latest`](#why-not-latest) below)
- Pushed to a **private** Docker Hub repository (`henryylim01/flask-app`)

### Kubernetes

Manifests live in [`k8s/`](./k8s):

- **`deployment.yaml`** — defines the Deployment: 1 replica, pulls the private image, injects `FLASK_SECRET_KEY` from a Kubernetes Secret, mounts `comments.db` to a **PersistentVolume** so data survives pod restarts, and authenticates to the private registry via `imagePullSecrets`
- **`service.yaml`** — exposes the app via a `NodePort` Service
- **`pv.yaml`** — a `hostPath` PersistentVolume + PersistentVolumeClaim backing the SQLite database, so the database isn't lost every time a pod is recreated

Two Kubernetes Secrets are used, for two different purposes:
| Secret | Purpose |
|---|---|
| `flask-app-secret` | App-level — injected into the container as `FLASK_SECRET_KEY` |
| `flask-app-registry-secret` | Cluster-level — a `docker-registry` Secret Kubernetes uses to **authenticate to Docker Hub** before it can even pull the private image |

### Azure Pipelines

`azure-pipelines.yml` defines a 4-step pipeline (build → push → deploy → cleanup) that runs on a **self-hosted agent** (this laptop, via WSL) rather than a Microsoft-hosted one — faster startup, and avoids consuming bootcamp Azure credits, since no VM is provisioned per run.

### Why not `:latest`

`:latest` isn't a real version — it's just a movable label pointing at whatever was pushed most recently, so the exact same manifest can silently pull different code at different times, breaks safe rollback, and forces `imagePullPolicy: Always` on every restart. Instead, every build is tagged with a unique, immutable identifier (`$(Build.BuildNumber)`) so a given tag always means the exact same image, forever — enabling reproducible deploys and instant rollback via `kubectl set image`.

### Persisting the database across pod restarts

By default, a container's filesystem is thrown away and rebuilt fresh from the image every time a pod is recreated (a Minikube restart, or a normal rolling deploy) — which meant `comments.db` was wiped every time. This was fixed with a `hostPath` PersistentVolume: the database file is mounted from the Minikube node's own disk (`/data/flask-app-db`) into the container at `/app/comments.db` via a `subPath` mount, so it now survives both pod recreation and cluster restarts.

---

## Mini Project A — Portfolio site + comments

- Built a Flask app with a `Comment` model and two routes: `/` (view comments) and `/add` (post a new comment)
- Used SQLite instead of MySQL, since free PythonAnywhere accounts no longer include MySQL access:
  ```python
  SQLALCHEMY_DATABASE_URI = "sqlite:///" + os.path.join(project_dir, "comments.db")
  ```
- Created the database tables via a one-off script (`create_db.py`), rather than the interactive Python shell — more reliable than typing multi-line code directly into the REPL
- Deployed via PythonAnywhere's **Manual configuration**, with a custom WSGI file pointing at `flask_app.py`
- Added name + LinkedIn link to the page (Bonus Task A)

## Mini Project B — Login, security, and a database-backed user system

### Task A — Login page
- Added `/login/` (GET + POST) and a matching `login_page.html` template, styled with Bootstrap
- Integrated **Flask-Login** to manage user sessions:
  - A `User` class (`UserMixin`) representing a logged-in user
  - A `login_manager.user_loader` function so Flask-Login can look up a user from their session
  - `app.secret_key` used internally to sign session cookies (loaded from the `FLASK_SECRET_KEY` environment variable — see **Security** below)
- Added `/logout/`, protected with `@login_required`

### Task B — Real, server-side security
Initially, the comment form was only *hidden* from logged-out users on the frontend — the `/add` route itself had no protection. This was demonstrated by:
1. Logging in, then opening "Log out" in a second tab (logging out there only)
2. Submitting a comment from the original, still-open tab

The comment still got saved, proving that hiding UI elements isn't real security — a request can always be sent directly to the server, bypassing the browser entirely.

**Fix:** added a server-side check as the first line of the `/add` route:
```python
@app.route("/add", methods=["POST"])
def add():
    if not current_user.is_authenticated:
        return redirect(url_for("index"))
    ...
```
This blocks the request at the server regardless of what the frontend does or doesn't show. Retesting the same two-tab scenario confirmed the comment is no longer saved when the request comes from a logged-out session.

### Bonus Task B — Users moved into the database
Originally, users were hardcoded in a Python dictionary (`all_users = {...}`) directly in `flask_app.py` — functional, but not realistic or extensible. This was replaced with:
- A `User` **database model** (`db.Model` + `UserMixin`), storing `username` and a hashed `password_hash`
- `load_user()` and the `login()` view updated to query the database instead of the dictionary
- A seed script (`create_users.py`) to create the `users` table and populate initial accounts, including a `tester` account

Verified directly via the SQLite CLI:
```bash
sqlite3 comments.db
SELECT * FROM users;
```
— confirming passwords are stored as hashes, never in plaintext. (`create_users.py` originally seeded fixed accounts with hardcoded passwords; it's since been rewritten to prompt for each username/password interactively so no plaintext credentials live in the source — see **Security** below.)

### Bonus Task A — Separate Portfolio/Introduction page
Added `/portfolio/` and `portfolio.html`, a standalone introduction page distinct from the comments Scratch Page, with a name, short bio, LinkedIn link, and a project summary. Linked both ways so visitors can move between the Scratch Page and the Portfolio page.

---

## Getting started

### Running locally (no Docker)

```bash
git clone https://github.com/henryylim01/flask-portfolio.git
cd flask-portfolio
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env        # reference for which vars you need; edit as a reminder
export FLASK_SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
# ^ the app reads this at runtime — set it in every new terminal session

python3 create_db.py         # creates comments.db
python3 create_users.py      # prompts you to add login accounts

python3 flask_app.py         # runs on http://localhost:5000
```

### Running via Docker

```bash
docker build -t flask-app:local .
docker run -p 5000:5000 \
  -e FLASK_SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))") \
  -e FLASK_DEBUG=false \
  flask-app:local
```
(You'll still need to `create_db.py`/`create_users.py` inside the container the first time — see `k8s/deployment.yaml` for the equivalent Kubernetes setup.)

### Deploying to Kubernetes

```bash
kubectl apply -f k8s/pv.yaml
kubectl create secret generic flask-app-secret \
  --from-literal=FLASK_SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
kubectl create secret docker-registry flask-app-registry-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-dockerhub-username> \
  --docker-password=<your-dockerhub-access-token>
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

# first-time only, or after a fresh PersistentVolume:
kubectl exec -it <pod-name> -- python3 create_db.py
kubectl exec -it <pod-name> -- python3 create_users.py
```

### CI/CD (Azure DevOps)

Pushing to the branch watched by `trigger:` in `azure-pipelines.yml` automatically builds, pushes, and deploys — no manual steps required, as long as a self-hosted agent (`myagent/run.sh`) is online and Minikube is running.

---

## Project structure

```
flask-portfolio/
├── flask_app.py          # Flask routes, database models, auth logic
├── create_db.py           # One-off script to create the comments table
├── create_users.py         # Interactive script to create the users table + accounts
├── Dockerfile                 # Builds the app image
├── azure-pipelines.yml         # CI/CD pipeline: build → push → deploy → cleanup
├── k8s/
│   ├── deployment.yaml            # Deployment: image, env, Secrets, PV mount
│   ├── service.yaml                # NodePort Service
│   └── pv.yaml                      # PersistentVolume + PersistentVolumeClaim
├── .env.example              # Template for required environment variables
├── requirements.txt          # Python dependencies
├── templates/
│   ├── index.html              # Scratch Page (comments)
│   ├── login_page.html          # Login form
│   └── portfolio.html            # Introduction/Portfolio page
└── .gitignore

Note: comments.db is not committed — locally it's gitignored and generated
via create_db.py/create_users.py; in Kubernetes it lives on a PersistentVolume,
outside the container image and outside Git entirely.
```

---

## What I learned

**From Mini Project A:**
- How Flask splits logic (routes) from presentation (Jinja2 templates)
- How Flask-SQLAlchemy maps Python classes to database tables
- Why free-tier constraints sometimes require adapting a tutorial (MySQL → SQLite)
- Git basics: cloning, committing, pushing, authenticating with a Personal Access Token

**From Mini Project B:**
- The difference between **frontend security** (hiding UI elements — makes a site *look* secure) and **backend security** (server-side checks — makes a site *actually* secure)
- How session-based authentication works with Flask-Login, including cookies and the role of `secret_key`
- Why passwords should always be stored as salted hashes (`generate_password_hash` / `check_password_hash`), never in plaintext
- How to migrate application data (users) from hardcoded Python structures into a proper database table
- Practical debugging on PythonAnywhere: reading error logs, diagnosing a `ModuleNotFoundError` for a missing package (`flask-login`), and fixing virtualenv dependency gaps

**From the CI/CD project:**
- How Docker containers nest inside a Minikube node (itself a single Docker container), and why a container's filesystem is ephemeral — rebuilt fresh from the image on every restart
- Why image tags matter: `:latest` is a moving target, not a version — reproducible deploys and safe rollback require immutable, unique tags per build
- The difference between a Deployment **manifest file** (a local YAML) and the Deployment **object** persisted in the cluster's etcd — they can drift apart once you run imperative commands like `kubectl set image`
- Why a pod's local storage doesn't survive recreation, and how a `hostPath` PersistentVolume + `subPath` mount fixes that — including a real debugging session where Kubernetes auto-created a *directory* instead of a *file* at the mount path, breaking SQLite until the file was pre-created manually
- The difference between an **app-level Secret** (env vars the app reads) and a **registry-level Secret** (`imagePullSecret`, used by Kubernetes itself before your app even starts, to authenticate a private image pull)
- Precisely which network hop actually needs bridging in a WSL2 + Docker + Minikube setup — WSL2 and Docker's bridge network are already connected, but a specific unpublished port isn't, which is what `minikube service --url` tunnels through
- Setting up and operating a **self-hosted Azure DevOps agent**: PAT-based registration, agent pools, and why self-hosted agents avoid the Microsoft-hosted per-minute VM billing (at the cost of maintaining the machine yourself)

---

## Security

This repo went through a security review and cleanup after the original bootcamp submission. Fixes applied:

- **Secret key** — no longer hardcoded in source; loaded from the `FLASK_SECRET_KEY` environment variable at startup (the app now fails fast if it's not set, instead of silently using a weak default). Copy `.env.example` to `.env` and fill in a real value generated with `python3 -c "import secrets; print(secrets.token_hex(32))"`.
- **Debug mode** — off by default; only enabled if `FLASK_DEBUG=true` is explicitly set.
- **CSRF protection** — added via Flask-WTF to both the login and comment forms.
- **No committed database or credentials** — `comments.db` and `__pycache__` are gitignored and generated locally instead of checked in. `create_users.py` now prompts for usernames/passwords interactively rather than storing them as plaintext in source.
- **Git history rewritten** — the repo's commit history was rewritten with `git-filter-repo` to permanently remove an old hardcoded secret key and plaintext seed passwords that had been committed early in the project.
- **Private container registry** — the Docker Hub repository is private, not public; pulling the image requires authentication.
- **Scoped registry credentials** — Kubernetes authenticates to Docker Hub using a **read-only** access token (via `imagePullSecret`), not the full Docker Hub account password, and not a broadly-scoped token.

If you're setting this up fresh, see **Getting started** above.

---

## Possible next steps

- Style the portfolio page to match the Bootstrap look of the login page
- Store timestamps and the posting user's name on each comment (linking `Comment` to `User`)
- Add basic input validation/error handling around login attempts (e.g. rate limiting)
- Bake `create_db.py`/`create_users.py` into the image's startup so a fresh PersistentVolume auto-initializes instead of requiring a manual `kubectl exec`
- Move from a `hostPath` PersistentVolume (single-node only) to a cloud-backed PV if this is ever deployed to a real multi-node cluster (e.g. AKS)
