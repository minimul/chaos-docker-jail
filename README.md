# chaos-docker-jail

Run [`seuros/chaos`](https://github.com/seuros/chaos) inside a Docker container with a minimal wrapper script.

## What this does

`chaos-docker-jail`:

- builds a Docker image containing the Chaos CLI
- mounts your current working directory into the container
- persists Chaos state in `~/.chaos-home`
- reuses Claude auth state from `~/.claude-home`
- exposes a compose-scoped `docker` CLI inside the container
- runs `chaos` from inside the container

The image is rebuilt automatically when the wrapper script changes, or manually with `--rebuild`.

## Requirements

- Docker installed and working on the host
- Bash

## Files

- `chaos-docker-jail` — main launcher script
- `compose.yml` — sample Compose stack for scope testing
- `test-compose-scope.sh` — host-side verification script

## Usage

```bash
./chaos-docker-jail [OPTIONS] [-- CHAOS_ARGS...]
```

### Options

- `-e, --env KEY=VAL` — pass an environment variable into the container
- `-r, --rebuild` — force rebuild of the Docker image
- `--no-cache` — rebuild without using Docker cache
- `-s, --shell` — start a bash shell instead of `chaos`
- `--safe` — do not pass `--dangerously-bypass-approvals-and-sandbox`
- `-h, --help` — show help

### Environment

- `CHAOS_REF` — git ref to build from `seuros/chaos` (default: `master`)

## Examples

Run Chaos:

```bash
./chaos-docker-jail
```

Pass arguments through to Chaos:

```bash
./chaos-docker-jail -- --help
```

Force a rebuild:

```bash
./chaos-docker-jail --rebuild
```

Rebuild with no Docker cache:

```bash
./chaos-docker-jail --no-cache
```

Start an interactive shell in the container:

```bash
./chaos-docker-jail --shell
```

Build Chaos from a different git ref:

```bash
CHAOS_REF=main ./chaos-docker-jail --rebuild
```

Pass environment variables into the container:

```bash
./chaos-docker-jail -e FOO=bar -e BAZ=qux
```

## Host directories used

The script creates and mounts:

- `~/.claude-home/.claude`
- `~/.claude-home/.claude.json`
- `~/.chaos-home/.chaos`
- `~/.chaos-home/.codex`

It also bind-mounts the current directory at the same path inside the container.

## Docker access inside the jail

When launched from a directory with a running Docker Compose project, the container gets:

- the host Docker socket mounted in
- the Docker CLI and Compose plugin installed
- a filtering `docker` shim at `/usr/local/bin/docker`

The shim only allows access to containers whose Compose label matches the launch directory:

- allowed: `docker ps`, `docker compose ...`, and container-scoped commands for in-scope containers
- blocked: host-wide Docker operations like `docker run`, `docker build`, `docker image`, `docker network`, etc.

This is a convenience guard rail, not a hard security boundary.

## Testing compose-scoped container access

This repo includes a sample `compose.yml` and a verification script.

Start the test stack and verify that only the local Compose containers are visible inside the jail:

```bash
chmod +x test-compose-scope.sh
./test-compose-scope.sh
```

What it checks:

- the local Compose stack is up
- `docker ps -a` inside `chaos-docker-jail` matches only containers from this directory's Compose project
- an out-of-scope container cannot be inspected from inside the jail

## Notes

- By default, the script runs Chaos with `--dangerously-bypass-approvals-and-sandbox`.
- Use `--safe` if you want to disable that default behavior.
- `--no-cache` implies a rebuild and passes `--no-cache` to `docker build`.
