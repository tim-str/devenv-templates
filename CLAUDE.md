# devenv-templates

Personal dev environment manager for macOS. Manages isolated devcontainers per project with SSH access from VS Code and IntelliJ IDEA.

## What this repo contains

- `bin/devenv` — CLI script managing container lifecycle
- `templates/` — devcontainer templates (copy-on-use per project)
- `docs/` — documentation (TBD)

## Runtime directories (not in this repo)

- `~/.devenv/projects/<project>/config.env` — per-project config (ports, repo URL, workspace path)
- `~/.devenv/projects/<project>/secrets.env` — tokens, never committed
- `~/dev/<project>/` — devcontainer config for the project (scaffolded from template)
- `~/projects/<project>/` — bind-mounted project workspace (cloned repo lives here)

## Stack

- Docker Desktop — container runtime
- `@devcontainers/cli` — builds and runs devcontainer specs
- `mcr.microsoft.com/devcontainers/base:ubuntu` — base image (provides `vscode` user)
- devcontainer features — tool installation (python, git, uv, etc.)

## Key design decisions

- **Copy-on-use templates** — `devenv new` copies the template; projects diverge independently
- **Bind mount** — `~/projects/<project>` on host maps to `/workspace` in container; survives rebuilds
- **SSH access** — public key baked in at build time via Docker `ARG SSH_PUBKEY`; sshd started via `postStartCommand`
- **`postStartCommand`** handles all init — sshd, git credentials, initial clone — because `devcontainer` overrides the image ENTRYPOINT
  - **Exception — `django-bff` template** splits the lifecycle the way the spec intends: `postCreateCommand` provisions the codebase + deps (git config → clone → `uv sync`, so `/workspace/.venv` is built from `uv.lock`), while `postStartCommand` only boots sshd. Because `devenv stop` does `docker rm`, each `start` recreates the container and re-runs `postCreateCommand`, reconciling deps to the lockfile. Dep install is non-fatal (won't block sshd) and only fires once a `pyproject.toml`/`uv.lock` (or `requirements.txt`) exists in the repo.
- **Host env vars** injected via `${localEnv:VAR}` in `devcontainer.json` — `GIT_TOKEN`, `GIT_USER_EMAIL`, `GIT_USER_NAME` must be set on host

## Required host env vars

```bash
export GIT_TOKEN=ghp_...          # GitHub classic token, repo scope
export GIT_USER_EMAIL=...
export GIT_USER_NAME=...
```

## Bootstrap on a new machine

```bash
git clone https://github.com/tim-str/devenv-templates ~/.devenv
echo 'export PATH="$HOME/.devenv/bin:$PATH"' >> ~/.bash_profile
# install deps
npm install -g @devcontainers/cli
```

## devenv commands

```bash
devenv new <project> --template <t>   # scaffold project from template
devenv start <project>                # build + start container, update ~/.ssh/config
devenv stop <project>                 # stop container, clean ~/.ssh/config
devenv ssh <project>                  # open SSH session
devenv list                           # show all projects + running status
devenv templates                      # list available templates
```