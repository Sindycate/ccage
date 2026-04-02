# ccage

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code) in a Docker container so it can only touch the repo you point it at — nothing else on your machine.

Born after a sub-agent deleted ~200GB of files on a MacBook. Never again.

## What it does

- Runs Claude Code inside a hardened Docker container (all capabilities dropped, no privilege escalation)
- Only the target repo is mounted read-write — the sole blast radius
- AWS credentials and Claude settings are mounted read-only
- Per-repo persistent state via Docker volumes (sessions, onboarding survive restarts)
- Reuses your host `~/.claude/settings.json` automatically

## Requirements

- macOS with [Colima](https://github.com/abiosoft/colima) (or Docker Desktop)
- Docker + Docker Compose
- AWS Bedrock credentials in `~/.aws/credentials`

Start Colima with enough memory (Claude Code needs 4GB+):

```bash
colima start --cpu 4 --memory 8 --disk 100
```

## Install

```bash
git clone git@github.com:Sindycate/ccage.git ~/claude-container
cd ~/claude-container
docker compose build
chmod +x ccage
ln -sf ~/claude-container/ccage ~/.local/bin/ccage
```

## Usage

```bash
# Run Claude Code against a repo
ccage ~/projects/myapp

# Yolo mode — skip all permission prompts (safe because containerized)
ccage -y ~/projects/myapp

# Pass any claude args through
ccage ~/projects/myapp --resume
ccage ~/projects/myapp -p "fix the failing tests"

# Multiple repos in parallel (separate terminals)
ccage ~/repo-a   # terminal 1
ccage ~/repo-b   # terminal 2
```

## How it works

`ccage` is a small bash script that runs `docker run` with:

| Mount | Path in container | Access |
|-------|-------------------|--------|
| Your repo | `/workspace` | **read-write** |
| `~/.aws/credentials` | `/home/claude/.aws/credentials` | read-only |
| `~/.claude` | `/host-claude` | read-only |
| Docker volume (per-repo) | `/home/claude/.claude` | read-write |

Everything else — your home directory, OS config, other repos — is not accessible to the container.

On each start, `entrypoint.sh` copies your host `settings.json` into the container's writable volume and symlinks `CLAUDE.md` and `agents/` if present. This means changes to your host Claude settings propagate automatically.

## Updating Claude Code

```bash
cd ~/claude-container
docker compose build --no-cache
```

## Managing state

```bash
# List active containers
docker ps --filter "name=claude-"

# List per-repo state volumes
docker volume ls --filter "name=claude-state-"

# Reset state for a repo
docker volume rm claude-state-<name>
```

## Limitations

- macOS-only (`md5 -q` in the launcher script — use `md5sum | cut -c1-8` on Linux)
- Network is open (Claude Code needs Bedrock API access) — no outbound firewall
- The mounted repo is still fully writable — but it's git-tracked, so worst case you `git checkout .`
