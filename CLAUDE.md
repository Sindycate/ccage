# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

A Docker-based isolation wrapper for running Claude Code in a container on macOS with Colima. Limits the blast radius so Claude Code can only write to the mounted repo, not the rest of the host filesystem. Uses AWS Bedrock for authentication.

## Build and Run

```bash
# Build/rebuild the image
docker compose build

# Rebuild from scratch (e.g., to update Claude Code version)
docker compose build --no-cache

# Run Claude Code against a repo
cage ~/path/to/repo

# Pass args through to claude
cage ~/path/to/repo --resume
cage ~/path/to/repo -p "do something"

# Yolo mode — skip all permission prompts (safe because containerized)
# Yolo defaults to --net gate (domain-gated networking)
cage -y ~/path/to/repo

# Explicit network gating without yolo
cage --net gate ~/path/to/repo

# No network at all
cage --net off ~/path/to/repo
```

## Architecture

**`cage`** (host-side launcher, symlinked to `~/.local/bin/`):
- Takes a repo path, derives a unique container name + Docker volume via md5 hash of the full path
- Loads git/SSH config from `~/.claude/cage.conf` (global) then `<repo>/.cage.conf` (per-project override)
- Runs `docker run` with security hardening (cap_drop ALL, no-new-privileges) and mounts:
  - Repo at `/workspace` (read-write) — the only writable host path
  - `~/.aws/credentials` read-only for Bedrock auth
  - `~/.claude` read-only at `/host-claude` for settings reuse
  - Per-repo named Docker volume at `/home/claude/.claude` for persistent state
  - SSH key read-only at `/home/claude/.ssh/id` for git push (if `SSH_KEY` configured)
  - `~/.ssh/known_hosts` read-only (if exists)
- Uses `md5 -q` (macOS-specific) for hashing — not portable to Linux

**`entrypoint.sh`** (runs inside container on every start):
- Symlinks `~/.claude.json` into the volume so onboarding state persists across `--rm` restarts
- Copies `settings.json` from host read-only mount into writable volume
- Symlinks `CLAUDE.md` and `agents/` from host if present
- Sets `git safe.directory` to handle UID mismatch between host and container
- Sets `user.name`/`user.email` from env vars (passed from `cage.conf`)
- Writes `~/.ssh/config` with SSH host alias if `SSH_HOST` is set

**`Dockerfile`**: Ubuntu 24.04, installs Claude Code via official installer, runs as non-root `claude` user. `jq` is required by the statusLine command in the host's `settings.json`.

**`docker-compose.yml`**: Build-only helper — tags the image as `claude-code:latest`. Not used for running containers (that's `cage`'s job).

**`netgate-proxy.py`** (host-side, runs when `--net gate` is active):
- Python3 forward proxy that gates outbound HTTP/HTTPS by domain
- Handles HTTPS via CONNECT method (sees hostname without TLS decryption)
- Holds unknown domains' connections open while showing a macOS `osascript` dialog
- Saves user decisions to allowlist files in `~/.claude/netgate/`
- Pre-allows `*.amazonaws.com` and related AWS domains via `netgate/defaults.json`
- Concurrent requests to the same unknown domain show only one dialog (deduplication via threading.Event)

**`netgate/defaults.json`**: Pre-allowed domain patterns (AWS infrastructure). Loaded on every proxy start.

## Key Constraints

- Host `~/.claude` is mounted **read-only** — entrypoint must copy/symlink, never write back
- `~/.claude.json` lives at `$HOME/.claude.json` (outside `$HOME/.claude/`), so the entrypoint symlinks it into the volume
- AWS auth uses long-lived credentials in `~/.aws/credentials` under the `claude-full` profile, via `zalando-aws-cli` on the host
- The `md5 -q` command in cage is macOS-only; use `md5sum | cut -c1-8` for Linux
- Network gating (`--net gate`) only covers HTTP/HTTPS traffic routed via proxy env vars. Raw TCP/SSH/DNS bypass the proxy (including `git push` over SSH)
- Git push requires `cage.conf` with `SSH_KEY` pointing to an unencrypted private key (passphrase-protected keys need ssh-agent, which is not available in the container)
- Allowlists: global at `~/.claude/netgate/global.json`, per-project at `~/.claude/netgate/project-{hash}.json`
- When `--net gate` is active, cage does NOT use `exec docker run` (needs shell alive for proxy cleanup)
