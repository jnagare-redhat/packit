# apply-packit

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash command that applies [Packit](https://packit.dev/) COPR builds from GitHub PRs to containerized Foreman/Satellite hosts.

## What it does

When you have a GitHub pull request with a Packit COPR build and a containerized Foreman/Satellite host, this skill automates the entire deployment:

1. Finds the latest Packit COPR build for the PR
2. Determines the correct install target (host, container, or hammer CLI)
3. Installs the RPM with proper persistence
4. Handles version mismatches between PR and installed packages
5. Restarts the necessary services
6. Verifies the installation

## Setup

### Option 1: Clone this repo

```bash
git clone https://github.com/jnagare-redhat/packit.git
cd packit
claude   # start Claude Code from this directory
```

### Option 2: Copy the skill into your existing project

```bash
mkdir -p /path/to/your/project/.claude/commands
cp .claude/commands/apply-packit.md /path/to/your/project/.claude/commands/
```

### Option 3: Copy to your user-level commands (available in all projects)

```bash
mkdir -p ~/.claude/commands
cp .claude/commands/apply-packit.md ~/.claude/commands/
```

### Prerequisites

**On the machine running Claude Code:**
- `sshpass` — install with `dnf install sshpass` or `apt install sshpass`
- `ssh` — standard SSH client

**On the remote host:**
- Containerized Foreman/Satellite deployment using systemd Quadlet
- `podman` managing the containers
- `dnf` available (both on host and inside containers)
- Root SSH access

## Usage

In Claude Code, run:

```
/apply-packit <GITHUB_PR_URL> <HOSTNAME>
```

### Examples

**Apply a Foreman plugin PR:**
```
/apply-packit https://github.com/theforeman/foreman_rh_cloud/pull/1214 satellite.example.com
```

**Apply a Katello PR:**
```
/apply-packit https://github.com/Katello/katello/pull/11822 satellite.example.com
```

**Apply a Hammer CLI PR:**
```
/apply-packit https://github.com/Katello/hammer-cli-katello/pull/1033 satellite.example.com
```

**Apply a host-level package PR:**
```
/apply-packit https://github.com/theforeman/foremanctl/pull/569 satellite.example.com
```

The skill will prompt you for the SSH root password before connecting.

## How it works

### Package classification

The skill determines install method based on the RPM package name:

| Package pattern | Where it installs | How |
|---|---|---|
| No `rubygem-` prefix (e.g. `foremanctl`) | Directly on host | `dnf copr enable` + `dnf install` |
| `rubygem-hammer_cli*` | Directly on host | `dnf copr enable` + `dnf install` |
| `rubygem-*` plugins (e.g. `rubygem-katello`) | Foreman container | Volume-mount persistence pattern |

### Why volume mounts?

Containerized Foreman/Satellite uses **systemd Quadlet**, which recreates containers from the base image on every restart. Any `dnf install` inside the container is lost. The skill works around this by:

1. Installing or extracting the RPM to get the gem files
2. Copying them to a persistent host directory (`/opt/<repo>-pr<number>/`)
3. Creating Quadlet drop-in `.conf` files that add `Volume=` bind mounts
4. systemd merges the drop-in configs and mounts the files into every new container

This applies to all containers using the `foreman.image`:
- `foreman` (main Rails app)
- `foreman-db-migrate` (runs migrations and seeds)
- `dynflow-sidekiq@*` (background job workers)
- `foreman-recurring@*` (scheduled tasks)

### Version mismatch handling

When a PR targets a newer version than what's installed (e.g. PR builds `rubygem-katello-5.1.0` but the container has `5.0.0`), `dnf` will fail with dependency errors. The skill handles this with the **overlay pattern**:

1. Downloads the RPM directly to the host (bypassing container dependency resolution)
2. Extracts it with `rpm2cpio`
3. Copies the existing gem from the container as a base
4. Overlays only the PR's changed source files on top
5. Mounts the overlay at the existing gem path

### Companion PRs

Server-side API changes often need a companion Hammer CLI PR to be visible in `hammer` output. Known pairings:

| Server-side repo | CLI companion |
|---|---|
| `Katello/katello` | `Katello/hammer-cli-katello` |
| `theforeman/foreman_rh_cloud` | `theforeman/hammer-cli-foreman-rh-cloud` |
| `theforeman/foreman_remote_execution` | `theforeman/hammer_cli_foreman_remote_execution` |
| `theforeman/foreman` | `theforeman/hammer-cli-foreman` |

The skill will ask about companion PRs when it detects new API fields.

## Rollback

The skill prints rollback instructions after every install. In general:

**Host-level packages:**
```bash
dnf downgrade -y <original_package_nvr>
dnf copr remove packit/<org>-<repo>-<pr_number>
```

**Container packages:**
```bash
# Remove drop-in override files
rm /etc/containers/systemd/*.container.d/<repo>-pr<number>.conf

# Remove persisted files
rm -rf /opt/<repo>-pr<number>/

# Reload and restart
systemctl daemon-reload
systemctl restart foreman.service dynflow-sidekiq@orchestrator.service \
  dynflow-sidekiq@worker.service dynflow-sidekiq@worker-hosts-queue.service
```

## Things to avoid

- **Never use `podman restart`** on Quadlet containers — it recreates from the base image and loses changes. Use `systemctl restart`.
- **Never use `podman commit`** — it captures stale PID files and running process state that breaks container startup.
- **Never hardcode passwords** — the skill always prompts.

## Applying multiple PRs

Each PR gets its own drop-in conf file (e.g. `foreman_rh_cloud-pr1214.conf`, `katello-pr11822.conf`). They stack — systemd merges all `.conf` files in a `.container.d/` directory. Run `/apply-packit` once per PR.

## License

MIT
