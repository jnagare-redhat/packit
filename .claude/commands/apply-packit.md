# Apply Packit COPR Build to Containerized Host

Apply a Packit COPR build from a GitHub PR to a containerized Foreman/Satellite host.

## Arguments
- `$ARGUMENTS` — space-separated: `[--rollback] <PR_URL> <HOST>`
  - Apply: `/apply-packit <PR_URL> <HOST>`
  - Rollback: `/apply-packit --rollback <PR_URL> <HOST>`
  - List applied: `/apply-packit --list <HOST>`
  Example: `https://github.com/theforeman/foreman_rh_cloud/pull/1214 ip-10-0-199-88.example.com`

## Instructions

When this skill is invoked, follow these steps precisely:

### Step 0: Parse inputs, detect mode, and ask for credentials

1. Parse `$ARGUMENTS` for flags and positional args:
   - If `--rollback` is present → go to **Rollback Mode** (see below)
   - If `--list` is present → go to **List Mode** (see below)
   - Otherwise → continue with **Apply Mode** (Step 1 onward)
2. Extract the PR URL and host. If either is missing, ask the user.
3. From the PR URL, extract `<org>` (e.g. `theforeman` or `Katello`), `<repo>` (e.g. `foreman_rh_cloud`), and `<pr_number>`.
4. **Ask the user for the SSH password** for the remote host using AskUserQuestion. Never hardcode passwords.

---

## List Mode (`--list`)

When `--list <HOST>` is passed, show all currently applied Packit PRs on the host:

1. Ask the user for the SSH password.
2. SSH to the host and discover applied PRs:

```bash
# Find host-level Packit COPR repos
dnf copr list 2>/dev/null | grep packit

# Find container-level volume-mount overrides
ls /etc/containers/systemd/foreman.container.d/*.conf 2>/dev/null

# Find persisted gem directories
ls -d /opt/*-pr*/ 2>/dev/null
```

3. For each discovered PR, report:
   - PR identifier (repo + PR number)
   - Package name and NVR (from `rpm -q` for host packages, or from the override conf/directory name for container packages)
   - Install type (host, hammer, or container volume-mount)

---

## Rollback Mode (`--rollback`)

When `--rollback <PR_URL> <HOST>` is passed, undo a previously applied Packit PR:

### Step R0: Parse and classify

1. Extract `<org>`, `<repo>`, `<pr_number>` from the PR URL.
2. Ask the user for the SSH password.
3. Determine the package type (same classification as Step 2 in Apply Mode):
   - Host-level / Hammer CLI → go to **Step R1**
   - Foreman container (volume-mount) → go to **Step R2**

### Step R1: Rollback host-level or hammer packages

```bash
# Find the currently installed Packit NVR
CURRENT=$(rpm -q <package_name>)

# Find what version to downgrade to — query the base repo for the non-Packit version
# The base version is typically the one from the Satellite repo without "pr<number>" in the release
dnf list available --disablerepo='copr:*' <package_name> 2>/dev/null

# Downgrade to the base version
dnf downgrade -y <package_name>

# Remove the COPR repo
dnf copr remove -y packit/<org>-<repo>-<pr_number>
```

Verify with: `rpm -q <package_name>` — confirm it no longer contains `pr<number>` in the release.

### Step R2: Rollback foreman container packages (volume-mount)

```bash
# Remove all Quadlet drop-in override files for this PR
for dir in /etc/containers/systemd/*.container.d; do
  rm -f "$dir/<repo>-pr<pr_number>.conf"
done

# Remove the persisted files
rm -rf /opt/<repo>-pr<pr_number>/

# Remove any leftover COPR repo file inside the container (if it exists)
podman exec --user 0 foreman rm -f /etc/yum.repos.d/packit-<repo>-pr<pr_number>.repo 2>/dev/null

# Reload systemd and restart all affected services
systemctl daemon-reload
systemctl restart foreman-db-migrate.service
# Wait for foreman-db-migrate to finish
systemctl restart foreman.service
# Wait for foreman to become active
systemctl restart dynflow-sidekiq@orchestrator.service dynflow-sidekiq@worker.service dynflow-sidekiq@worker-hosts-queue.service
```

Wait for `foreman.service` to reach `active` status by polling `systemctl is-active foreman.service`.

### Step R3: Verify rollback

- For host-level packages: `rpm -q <package_name>` — confirm the base version is restored
- For container packages: `podman exec foreman ls <gem_path>` — confirm the original gem is back (no overlay)
- Confirm no override conf files remain: `ls /etc/containers/systemd/foreman.container.d/*<pr_number>* 2>/dev/null` should return nothing
- Confirm persisted directory is gone: `ls /opt/<repo>-pr<pr_number>/ 2>/dev/null` should return nothing

### Step R4: Report rollback results

Summarize:
- PR that was rolled back
- Package name and what version it was restored to
- Services that were restarted

---

## Apply Mode

### Step 1: Determine the COPR repo and RPM package name

1. The Packit COPR repo follows the pattern: `packit/<org>-<repo>-<pr_number>`
   - COPR base URL: `https://download.copr.fedorainfracloud.org/results/packit/<org>-<repo>-<pr_number>/rhel-9-x86_64/`
   - Note: `<org>` and `<repo>` are case-sensitive and must match the GitHub URL exactly (e.g. `Katello/katello` produces `packit/Katello-katello-<number>`, `theforeman/foreman_rh_cloud` produces `packit/theforeman-foreman_rh_cloud-<number>`).
2. Fetch the COPR directory listing to find the latest build directory (highest build ID number prefix).
3. Fetch `results.json` from the latest build directory to get the exact NVR (name-version-release) for the `noarch` RPM.
4. From `results.json`, extract the RPM package name (the `name` field) and the full NVR.

### Step 2: Determine installation target

Classify the package to decide where it gets installed:

- **Host-level packages** (installed directly on the host):
  - Package names that do NOT start with `rubygem-` (e.g. `foremanctl`, `yggdrasil-worker-forwarder`)
  - Install with: `dnf copr enable` + `dnf install`

- **Foreman container packages** (installed inside the `foreman` container):
  - Package names starting with `rubygem-` that are Foreman plugins (e.g. `rubygem-foreman_rh_cloud`, `rubygem-katello`, `rubygem-foreman_remote_execution`, `rubygem-foreman_virt_who_configure`)
  - These need the **volume-mount persistence pattern** (see Step 4)

- **Hammer CLI packages** (installed directly on the host):
  - Package names starting with `rubygem-hammer_cli` (e.g. `rubygem-hammer_cli_foreman`, `rubygem-hammer_cli_katello`)
  - Install with: `dnf copr enable` + `dnf install` on the host
  - Note: Hammer CLI PRs often accompany server-side PRs. For example, a Katello API change adding new fields typically has a companion hammer-cli-katello PR to display those fields in the CLI output. Ask the user if there is a companion PR if the changes seem incomplete (e.g. new API fields without CLI display).

### Step 3: Install host-level or hammer packages

For packages that install directly on the host:

```bash
sshpass -p '<PASSWORD>' ssh -o StrictHostKeyChecking=no root@<HOST> \
  "dnf copr enable -y packit/<org>-<repo>-<pr_number> rhel-9-x86_64 && \
   dnf install -y <FULL_NVR>.<arch>"
```

Verify with: `rpm -q <package_name>`

Skip to Step 6 after verification.

### Step 4: Install foreman container packages (volume-mount pattern)

For `rubygem-*` Foreman plugin packages, a direct `dnf install` inside the container is lost on restart because these are systemd-managed Quadlet containers that recreate from the base image. Use volume mounts to persist:

#### 4a. Try installing via dnf first

```bash
# Add COPR repo
podman exec --user 0 foreman bash -c 'cat > /etc/yum.repos.d/packit-<repo>-pr<number>.repo << EOF
[packit-<org>-<repo>-<pr_number>]
name=Packit <repo> PR <pr_number>
baseurl=https://download.copr.fedorainfracloud.org/results/packit/<org>-<repo>-<pr_number>/rhel-9-x86_64/
enabled=1
gpgcheck=0
module_hotfixes=1
EOF'

# Install/upgrade — disable rhel-* repos if they fail with 403
podman exec --user 0 foreman dnf upgrade -y --disablerepo='rhel-*' <package_name>
```

**If dnf fails with dependency errors** (version mismatch), fall back to Step 4b.

#### 4b. Handle version mismatches (RPM extraction + overlay)

This commonly happens when the PR targets a newer major version than what's installed in the container (e.g. PR builds `rubygem-katello-5.1.0` but container has `katello-5.0.0`, or PR requires `foreman >= 5.1` but container has `foreman 5.0`).

Instead of forcing the install, download the RPM on the host, extract it, and overlay only the changed files onto the existing gem:

```bash
# Download the RPM to the host (bypasses container dependency resolution)
mkdir -p /opt/<repo>-pr<number>
cd /opt/<repo>-pr<number>
dnf download --disablerepo='*' \
  --repofrompath=packit-<repo>,'https://download.copr.fedorainfracloud.org/results/packit/<org>-<repo>-<pr_number>/rhel-9-x86_64/' \
  <FULL_NVR>.<arch>

# Extract RPM contents
mkdir -p extracted
cd extracted
rpm2cpio ../<rpm_filename> | cpio -idmv
```

Then create an overlay by copying the EXISTING gem from the container and applying the PR's changed files on top:

```bash
# Copy the existing gem as the base (preserves correct version path)
EXISTING_GEM_PATH=$(podman exec foreman ls /usr/share/gems/gems/ | grep '<gem_name_without_rubygem_prefix>')
podman cp foreman:/usr/share/gems/gems/$EXISTING_GEM_PATH /opt/<repo>-pr<number>/overlay

# Copy changed files from the extracted RPM over the base
# The extracted gem will be at: extracted/usr/share/gems/gems/<gem_name>-<new_version>/
# Overlay the relevant files onto: /opt/<repo>-pr<number>/overlay/
# Only overlay application code files (controllers, views, models, lib, config), NOT gemspec or version files
```

Important: When the PR version differs from the installed version, assets/webpack directories inside the extracted RPM are often **symlinks** pointing to the new version path (e.g. `-> /usr/share/gems/gems/katello-5.1.0.pre.master/public/assets/katello`). These symlinks will be broken when mounted. Always use the overlay approach (copy base + apply changes) rather than trying to mount the extracted assets directly.

#### 4c. Copy installed files to the host for persistence

**If dnf install succeeded (no version mismatch):**

```bash
# Determine gem name and version from the RPM file list
GEM_DIR=$(podman exec foreman rpm -ql <package_name> | grep '/usr/share/gems/gems/' | head -1 | cut -d'/' -f1-6)

mkdir -p /opt/<repo>-pr<number>

# Copy the gem directory, gemspec, bundler config, assets, and webpack
podman cp foreman:$GEM_DIR /opt/<repo>-pr<number>/gem
podman cp foreman:/usr/share/gems/specifications/<gem_name>.gemspec /opt/<repo>-pr<number>/ 2>/dev/null
podman cp foreman:/usr/share/foreman/bundler.d/<gem_name_underscored>.rb /opt/<repo>-pr<number>/ 2>/dev/null
podman cp foreman:/usr/share/foreman/public/assets/<gem_name_underscored> /opt/<repo>-pr<number>/assets 2>/dev/null
podman cp foreman:/usr/share/foreman/public/webpack/<gem_name_underscored> /opt/<repo>-pr<number>/webpack 2>/dev/null
```

**If using the overlay approach (version mismatch):** The overlay directory from Step 4b is already prepared at `/opt/<repo>-pr<number>/overlay`.

#### 4d. Create Quadlet volume-mount overrides

Create override conf files for all containers that use `foreman.image` under `/etc/containers/systemd/`:
- `foreman.container.d/`
- `foreman-db-migrate.container.d/`
- `dynflow-sidekiq@.container.d/`
- `dynflow-sidekiq@orchestrator.container.d/`
- `dynflow-sidekiq@worker.container.d/`
- `dynflow-sidekiq@worker-hosts-queue.container.d/`
- `foreman-recurring@daily.container.d/`
- `foreman-recurring@hourly.container.d/`
- `foreman-recurring@monthly.container.d/`
- `foreman-recurring@weekly.container.d/`

Each override file (`<dir>/<repo>-pr<number>.conf`) should contain:

**If dnf install succeeded (same version, full gem copied):**
```ini
[Container]
Volume=/opt/<repo>-pr<number>/gem:<container_gem_path>:ro,z
Volume=/opt/<repo>-pr<number>/<gemspec_file>:<container_gemspec_path>:ro,z
Volume=/opt/<repo>-pr<number>/assets:<container_assets_path>:ro,z
Volume=/opt/<repo>-pr<number>/webpack:<container_webpack_path>:ro,z
```

**If using overlay (version mismatch, overlay over existing gem path):**
```ini
[Container]
Volume=/opt/<repo>-pr<number>/overlay:<existing_container_gem_path>:ro,z
```

Only add Volume lines for files/directories that actually exist.

#### 4e. Reload and restart services

```bash
systemctl daemon-reload
systemctl restart foreman-db-migrate.service   # runs db:migrate + db:seed with new code
# Wait for foreman-db-migrate to finish (it's a oneshot service)
systemctl restart foreman.service
# Wait for foreman to become active (may take 60-120s for seed + puma boot)
systemctl restart dynflow-sidekiq@orchestrator.service dynflow-sidekiq@worker.service dynflow-sidekiq@worker-hosts-queue.service
```

Wait for foreman.service to reach `active` status by polling `systemctl is-active foreman.service`.

### Step 5: Verify

#### 5a. Verify package installation
- For host-level packages: `rpm -q <package_name>`
- For container packages (dnf install): `podman exec foreman rpm -q <package_name>` to confirm, and `podman exec foreman ls <container_gem_path>` to confirm the volume mount
- For container packages (overlay): `podman exec foreman ls <overlay_mount_path>/<changed_file>` to confirm the changed files are present
- Report the installed NVR and where it was installed

#### 5b. Verify all services are running

Check that all critical services are active after the restart:

```bash
# Core services
systemctl is-active foreman.service
systemctl is-active dynflow-sidekiq@orchestrator.service
systemctl is-active dynflow-sidekiq@worker.service
systemctl is-active dynflow-sidekiq@worker-hosts-queue.service

# Check foreman-db-migrate completed successfully (oneshot — should be inactive/dead with success)
systemctl is-failed foreman-db-migrate.service  # should return "inactive", NOT "failed"

# Verify foreman container is healthy — hit the ping API
curl -sk https://localhost/api/v2/ping | python3 -m json.tool
```

If any service is not active, report the failure and show `systemctl status <service>` and `journalctl -u <service> --no-pager -n 50` for the failed service.

The `/api/v2/ping` response shows the status of all backend services (database, Pulp, Candlepin, etc.). Report any service that is not "ok".

### Step 6: Report results

Summarize:
- PR URL and package NVR installed
- Installation target (host or foreman container)
- Persistence method used (direct install, volume-mount, or overlay)
- Rollback instructions:
  - Host-level: `dnf downgrade -y <original_nvr>` + `dnf copr remove packit/<org>-<repo>-<pr_number>`
  - Container: Remove override conf files from all `*.container.d/` dirs, remove `/opt/<repo>-pr<number>/`, then `systemctl daemon-reload && systemctl restart foreman.service dynflow-sidekiq@orchestrator.service dynflow-sidekiq@worker.service dynflow-sidekiq@worker-hosts-queue.service`

## Important notes

- All SSH commands use `sshpass -p '<PASSWORD>' ssh -o StrictHostKeyChecking=no root@<HOST>` — never hardcode the password.
- When `dnf` inside the container fails with 403 errors on `rhel-*` repos, add `--disablerepo='rhel-*'`.
- Never use `podman restart` for systemd-managed Quadlet containers — it recreates them from the base image and loses all in-container changes. Always use `systemctl restart`.
- Never use `podman commit` to persist changes — the committed image captures stale PID files (`/tmp/rails.pid`) and running process state that breaks container startup. The container entrypoint runs `bin/rails db:migrate && bin/rails db:seed && bin/rails server --pid /tmp/rails.pid`, and a committed PID file causes "A server is already running" errors on next start.
- The volume-mount pattern survives container restarts because the files live on the host filesystem and are bind-mounted into new containers by systemd Quadlet.
- When applying multiple PRs, each PR gets its own override conf file in each `*.container.d/` directory. They stack — systemd merges all conf files in the drop-in directory.

## Version mismatch patterns

Common version mismatch scenarios and how to handle them:

| Scenario | Example | Resolution |
|----------|---------|------------|
| PR targets newer major version | PR builds katello 5.1, container has 5.0 | Use overlay: copy existing 5.0 gem, apply PR's changed files on top |
| PR requires newer foreman | PR needs `foreman >= 5.1`, container has 5.0 | Use overlay approach (same as above) |
| PR requires newer dependency | PR needs `scoped_search >= 5.0` | Use overlay approach |
| Same major version | PR builds foreman_rh_cloud 14.3.0, container has 14.3.0 | dnf upgrade works directly |

Key principle: when versions match, `dnf upgrade` inside the container works and you copy the full gem for volume mounting. When versions don't match, download the RPM on the host, extract it, and overlay only the changed source files onto a copy of the existing gem.

## Companion PRs

Some PRs require companion changes in other repos to be fully functional:

| Server-side repo | CLI companion repo | Example |
|------------------|--------------------|---------|
| `Katello/katello` | `Katello/hammer-cli-katello` | New API fields need hammer CLI columns |
| `theforeman/foreman_rh_cloud` | `theforeman/hammer-cli-foreman-rh-cloud` | New RH Cloud features need CLI support |
| `theforeman/foreman_remote_execution` | `theforeman/hammer_cli_foreman_remote_execution` | REX features need CLI support |
| `theforeman/foreman` | `theforeman/hammer-cli-foreman` | Core Foreman changes need CLI support |

If only a server-side PR is provided and the changes include new API endpoints or response fields, ask the user if there is a companion hammer CLI PR that should also be applied. The server-side changes (new API fields) will work without the CLI PR, but the new data won't be visible in `hammer` output until the CLI PR is also applied.
