# Packit COPR Applier for Foreman/Satellite

Claude Code skill for applying Packit COPR builds from GitHub PRs to containerized Foreman/Satellite hosts.

## What this does

The `/apply-packit` slash command automates applying RPM packages built by [Packit](https://packit.dev/) from GitHub pull requests onto a remote containerized Foreman/Satellite host. It handles the full lifecycle: finding the COPR build, determining install target, installing the package, persisting through container restarts, and verifying.

## Usage

```
/apply-packit <GITHUB_PR_URL> <HOST>
```

Example:
```
/apply-packit https://github.com/theforeman/foreman_rh_cloud/pull/1214 ip-10-0-199-88.example.com
```

The skill will ask for the SSH root password before proceeding.

## Prerequisites

The following tools must be available on the machine running Claude Code:
- `sshpass` — for non-interactive SSH password authentication
- `ssh` — for connecting to the remote host
- `curl` or `WebFetch` — for fetching COPR build metadata

The remote host must:
- Be a containerized Foreman/Satellite deployment using systemd Quadlet
- Have `podman` managing the containers
- Have containers managed by systemd units under `/etc/containers/systemd/`
- Have `dnf` available (both on host and inside containers)
- Allow root SSH access

## How it works

### Package classification

The skill classifies packages by name to determine where and how to install:

| Package pattern | Install target | Method |
|----------------|---------------|--------|
| No `rubygem-` prefix (e.g. `foremanctl`) | Host | `dnf install` directly |
| `rubygem-hammer_cli*` | Host | `dnf install` directly |
| `rubygem-*` plugins (e.g. `rubygem-katello`) | Foreman container | Volume-mount persistence |

### Persistence for container packages

Containerized Foreman uses systemd Quadlet, which recreates containers from the base image on every restart. Packages installed via `dnf` inside a container are lost. The skill uses volume mounts to persist changes:

1. Installs/extracts the RPM to get gem files
2. Copies gem files to a host directory (`/opt/<repo>-pr<number>/`)
3. Creates Quadlet drop-in override files (`.conf`) in each container's `.container.d/` directory
4. The drop-in files add `Volume=` directives that bind-mount the gem files into the container
5. Changes survive container restarts because the files live on the host filesystem

### Version mismatch handling

When a PR targets a newer version than what's installed (e.g. PR builds katello 5.1 but container has 5.0), `dnf` will fail with dependency errors. The skill handles this by:

1. Downloading the RPM directly to the host
2. Extracting it with `rpm2cpio`
3. Copying the existing gem from the container as a base
4. Overlaying only the changed source files from the PR

### Companion PRs

Server-side changes (new API fields) often have companion hammer CLI PRs. The skill knows the common pairings:

- `Katello/katello` <-> `Katello/hammer-cli-katello`
- `theforeman/foreman_rh_cloud` <-> `theforeman/hammer-cli-foreman-rh-cloud`
- `theforeman/foreman_remote_execution` <-> `theforeman/hammer_cli_foreman_remote_execution`
- `theforeman/foreman` <-> `theforeman/hammer-cli-foreman`

## Things to avoid

- **Never use `podman restart`** for Quadlet-managed containers — use `systemctl restart`
- **Never use `podman commit`** — it captures stale PID files and process state
- **Never hardcode passwords** — always prompt the user
- **Never use `--disablerepo='rhel-*'`** unless RHEL repos return 403 errors inside the container
