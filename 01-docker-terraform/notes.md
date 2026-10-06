# 01-docker-terraform — notes
## Setup (Mac, Apple Silicon, 8 GB RAM)

**What broke and how I fixed it**
- `brew: command not found` after installing Homebrew → hadn't run the
  "Next steps" PATH commands it printed; ran them, then `brew` worked
- `(base)` in the prompt = conda auto-activating; turned it off with
  `conda config --set auto_activate_base false` so uv manages Python
- My repo had the same name as the course repo, so cloning failed
  → renamed mine to `de-zoomcamp`. Course repo = read-only, mine = work
- Pasting commands with `# comments` created junk files/folders
  → zsh ignores comments only with `setopt interactive_comments`
- "failed to connect to the docker API" → Docker Desktop wasn't open;
  `open -a Docker`

**Key facts**
- Docker memory: 4 GB of my 8 GB; check usage with `docker system df`
- GCP keys live in `~/.gcp/`, never inside a repo
- Check the prompt before running commands: `%` = my Mac, `#` = inside a container

## Lesson 1 – Docker intro

**What I learned**
- Image = template; container = a running copy of it
- Every `docker run` makes a NEW container; stopped ones remain (`docker ps -a`)
- Containers are stateless: changes don't carry into a new container
- `--rm` deletes the container on exit; skip it when you need the logs for debugging
- `-v mac-folder:container-folder` shares a folder, so data survives the container
  (this is how Postgres keeps its data in lesson 4)
- `--entrypoint=bash` overrides what the image runs by default
- Installs can hang on prompts (tzdata); `DEBIAN_FRONTEND=noninteractive` avoids that

## Lesson 2 – Virtual environments with uv

**What I learned**
- `pip install` puts packages globally, so projects clash on versions
- `uv init` creates `pyproject.toml` (dependencies) and `.python-version`
- `uv add pandas` installs into the project's `.venv` only
- `uv run python ...` uses the project's Python (3.13), not the Mac's (3.9.6)
- `uv.lock` pins exact versions, so builds are reproducible
- Keep binary outputs out of Git: `*.parquet` in `.gitignore`
- `sys.argv` passes command-line arguments into a script

## Lesson 3 – Dockerizing the pipeline

**What I learned**
- A Dockerfile is the recipe; `docker build` makes the image; `docker run` starts a container from it
- Copy dependency files before code so Docker caches the pandas install

**What broke and how I fixed it**
- `uv run` failed in Docker: "Expected a Python module at src/pipeline/__init__.py"
- Cause: newer uv set up the project as a package
- Fix: added `package = false` under `[tool.uv]` in pyproject.toml, ran `uv lock`

## Lesson 4 – Postgres in Docker

**What I learned**
- Postgres runs fully in a container, with nothing installed on the Mac
- `-e` sets environment variables (user, password, database name)
- `-p 5432:5432` = Mac port : container port, so Mac tools can reach the database
- Named volume (`ny_taxi_postgres_data:/var/lib/postgresql`): Docker stores the
  data, so it survives `--rm`. Proved it: stopped Postgres, restarted it, and
  my test table was still there
- Named volume vs bind mount: named = managed by Docker (better on Mac);
  bind = a specific Mac folder (permission/speed issues on Mac)
- Need two terminals: one runs Postgres (shows logs), one for working
- `uv add --dev` = tools for development only (pgcli), kept separate from
  the pipeline's real dependencies
- pgcli: `\dt` lists tables, `\q` quits
- Data lives in the volume, never in Git; the repo only holds code and config

**What broke and how I fixed it**
- pgcli crashed: "ImportError: no pq wrapper available"
  - Cause: Mac doesn't ship libpq (the Postgres client library)
  - Fix: `uv add --dev "psycopg[binary]"`, which bundles libpq
- zsh needs quotes around square brackets: `"psycopg[binary]"`
- In VS Code, Cmd+T opens symbol search, not a terminal; new terminal =
  "+" in the terminal panel or Ctrl+Shift+`
- If "port is already allocated": something else is using 5432; map
  `-p 5433:5432` and connect with `-p 5433`