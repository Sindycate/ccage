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
# Automatically enables domain-gated networking
ccage -y ~/projects/myapp

# Explicit network gating (prompts for each new domain)
ccage --net gate ~/projects/myapp

# No network at all
ccage --net off ~/projects/myapp

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

## Network gating

With `--net gate`, all outbound HTTP/HTTPS from the container routes through a host-side proxy that prompts you before allowing access to new domains.

**How it works:**
1. A Python proxy starts on the host and binds to a random port
2. The container gets `HTTP_PROXY`/`HTTPS_PROXY` env vars pointing to it
3. When Claude Code (or any tool) tries to reach a new domain, a macOS dialog pops up
4. You choose: **Allow (project)**, **Allow (always)**, or **Deny**
5. The connection is held open during the prompt — no failed first request

**Pre-allowed domains:** AWS infrastructure (`*.amazonaws.com`, `*.amazontrust.com`, `*.cloudfront.net`) is always allowed since Claude Code needs Bedrock API access.

**Allowlist storage:**
- Global (all projects): `~/.claude/netgate/global.json`
- Per-project: `~/.claude/netgate/project-{hash}.json`
- Manually edit these files to add/remove domains

**Yolo + gating:** `ccage -y` defaults to `--net gate`. Override with `ccage -y --net open` if you want full network access.

## Limitations

- macOS-only (`md5 -q` in the launcher, `osascript` for network gate dialogs)
- Network gating only covers HTTP/HTTPS via proxy env vars — raw TCP, SSH, and DNS bypass the proxy
- The mounted repo is still fully writable — but it's git-tracked, so worst case you `git checkout .`
