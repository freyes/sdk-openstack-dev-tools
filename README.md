# OpenStack Dev Tools SDK for Workshop

A development environment for OpenStack projects within a [Workshop](https://github.com/canonical/workshop). This SDK provisions all the tools, packages, and configuration needed to work on four OpenStack development workflows: upstream service development, Ubuntu package maintenance, Sunbeam deployment engineering, and Charmed OpenStack charm authoring.

---

## Features

- **Python development stack** — Python 3, pip, venv, tox, stestr, bindep, pre-commit
- **OpenStack CLI** — `openstacksdk` and `python-openstackclient` pre-installed via pip
- **Ubuntu packaging** — `openstack-pkg-tools`, `ubuntu-dev-tools`, `devscripts`, `dpkg-dev`, `build-essential`
- **Charmed operator tooling** — Juju, Charmcraft, kubectl, MicroK8s via snap (best-effort)
- **Shared venv** — persists at `/home/workshop/.openstack-venv` across workshop updates via mount plug
- **Persistent caches** — tox, pip, and pre-commit caches survive refreshes
- **Shell integration** — PATH, bash completions, and aliases configured automatically
- **Health checks** — `workshopctl set-health` integration verifies all tools on launch

---

## What's Installed

### Apt packages

| Category | Packages |
|---|---|
| **Python** | `python3` `python3-dev` `python3-pip` `python3-venv` |
| **Upstream dev** | `tox` `stestr` `bindep` |
| **Packaging** | `openstack-pkg-tools` `devscripts` `dpkg-dev` `ubuntu-dev-tools` `build-essential` |
| **Cloud Archive** | `cloud-archive-utils` (from `ppa:ubuntu-cloud-archive/tools`) |
| **Build deps** | `libssl-dev` `libffi-dev` `libxml2-dev` `libxslt1-dev` |
| **Databases** | `mariadb-client` `postgresql-client` `memcached` |
| **Git** | `git` `git-review` |
| **Utilities** | `jq` `curl` `wget` `vim` `tmux` |

### Pip packages

| Package | Purpose |
|---|---|
| `openstacksdk` | Unified SDK for all OpenStack services |
| `python-openstackclient` | `openstack` CLI command |

### Snap packages (best-effort — requires `vm: true`)

| Snap | Flag | Purpose |
|---|---|---|
| `juju` | `--classic` | Charmed Operator Lifecycle Manager |
| `charmcraft` | `--classic` | Build and publish charms |
| `kubectl` | `--classic` | Kubernetes CLI |
| `microk8s` | `--classic` | Lightweight Kubernetes |

All snap installs are **best-effort** — they warn but do not fail if snapd is unavailable (e.g., in a container backend).

---

## Reference Workshop Definition

```yaml
# workshop.yaml
name: openstack-dev-tools
base: ubuntu@24.04
sdks:
  - name: openstack-dev-tools
  - name: orange-dev-tools
  - name: direnvrc
  - name: claude-code
  - name: opencode

actions:
  clone-nova: |
    git clone https://opendev.org/openstack/nova openstack/nova
  setup-upstream: |
    openstack-dev-setup upstream openstack/nova
  clone-charm: |
    git clone https://github.com/openstack-charmers/charm-nova charms/nova
  build-charm: |
    cd charms/nova && charmcraft pack
```

> **Note:** The `orange-dev-tools` SDK provides the full Debian packaging stack (`sbuild`, `dput`, `mini-dinstall`, `git-ubuntu`). Include it for Ubuntu OpenStack package maintenance workflows.

---

## Usage Guide

### Prerequisites

- **VM backend required** for snap-based tools (Juju, Charmcraft, MicroK8s). Use `vm: true` in your workshop definition.
- **`orange-dev-tools` SDK** for Debian packaging workflows (`sbuild`, `dput`, `mini-dinstall`, `git-ubuntu`).

### Workflow 1 — Upstream OpenStack Development

Develop against individual OpenStack service repos (Nova, Neutron, Keystone, Glance, Swift, Cinder, etc.).

```bash
# Clone an upstream service repository
workshop run clone-nova openstack/nova

# Bootstrap the development environment (venv, pre-commit, tox)
workshop run setup-upstream openstack/nova

# Enter the workshop and start developing
workshop shell
cd openstack/nova

# List available tox environments
tox -l

# Run unit tests
tox -e py3

# Run style checks
tox -e pep8
```

### Workflow 2 — Ubuntu OpenStack Packaging

Maintain Debian/Ubuntu packages for OpenStack services. Requires `orange-dev-tools` SDK.

```bash
workshop shell

# Clone an Ubuntu source package
git ubuntu clone nova
cd nova
git ubuntu export-orig

# Quick local build
dpkg-buildpackage -us -uc

# Clean chroot build (requires create-sbuild-chroots first)
create-sbuild-chroots --all
sbuild --dist=$(lsb_release -cs)

# Upload to local apt repository
dput local ../nova_*.changes
```

### Workflow 3 — Sunbeam OpenStack Development

Develop on Canonical's OpenStack-on-Kubernetes distribution.

```bash
workshop shell

# Clone Sunbeam Python monorepo
git clone https://github.com/openstack/sunbeam sunbeam/sunbeam
cd sunbeam/sunbeam

# Run tests
tox -e functional-feature

# Clone and build charms
git clone https://github.com/openstack/sunbeam-charms sunbeam/charms
cd sunbeam/charms

# Build a charm
charmcraft pack
```

### Workflow 4 — Charmed OpenStack Development

Author and maintain Juju charms for OpenStack services.

```bash
workshop run clone-charm charms/nova
workshop shell
cd charms/nova

# Build the charm
charmcraft pack

# Run unit tests
tox -e unit

# Run integration tests
tox -e integration
```

---

## Configuration

| File | Purpose |
|---|---|
| `/etc/profile.d/openstack-dev-tools.sh` | System-wide PATH: adds `$SDK/bin` |
| `~/.profile` | User PATH: adds `$SDK/bin` |
| `~/.bash_aliases` | Aliases: `osc` → `openstack`, `os` → `openstack` |
| `/home/workshop/.openstack-venv` | Shared Python virtual environment (persisted) |
| `/home/workshop/.cache/openstack-dev` | tox/pip/pre-commit caches (persisted) |
| `/home/workshop/.local/share/juju` | Juju credentials and model data (persisted) |
| `/home/workshop/openstack/` | Workspace for upstream service repos |
| `/home/workshop/charms/` | Workspace for charm repos |

---

## Plugs

### `openstack-venv`
- **Interface:** `mount`
- **Workshop target:** `/home/workshop/.openstack-venv`
- **Purpose:** Persists the shared Python virtual environment across workshop updates and refreshes.

### `dev-cache`
- **Interface:** `mount`
- **Workshop target:** `/home/workshop/.cache/openstack-dev`
- **Purpose:** Persists tox, pip, and pre-commit caches to avoid re-downloading on every launch.

### `juju-data`
- **Interface:** `mount`
- **Workshop target:** `/home/workshop/.local/share/juju`
- **Purpose:** Persists Juju client data — credentials, controller connections, and model state — across workshop refreshes.

---

## How It Works

### Lifecycle hooks

| Hook | Runs as | When | Purpose |
|---|---|---|---|
| `hooks/setup-base` | root | Workshop launch + refresh, before user session | Installs apt/pip/snap packages, writes `/etc/profile.d/openstack-dev-tools.sh`, creates workspace directories |
| `hooks/setup-project` | workshop user | After setup-base, launch + refresh | Creates shared Python venv, configures shell aliases, writes user PATH |
| `hooks/check-health` | root | After all setup hooks complete | Verifies required tools exist and reports status via `workshopctl set-health` |

### Helper script

| Script | Purpose |
|---|---|
| `bin/openstack-dev-setup` | Bootstraps a project directory for a specific workflow: `openstack-dev-setup upstream <dir>` installs tox + pre-commit; `openstack-dev-setup charm <dir>` installs tox only |

---

## Troubleshooting

### Snaps not available (container backend)

The snap-based tools (Juju, Charmcraft, kubectl, MicroK8s) install via snap but require a VM backend. If you're running in a container:

```yaml
# workshop.yaml — add vm: true
name: openstack-dev-tools
base: ubuntu@24.04
vm: true
sdks:
  - name: openstack-dev-tools
```

Without `vm: true`, the `setup-base` hook will log warnings for each snap install failure, but the rest of the environment works normally.

### openstack-pkg-tools not found

This package is in the `universe` component. If you see `E: Unable to locate package openstack-pkg-tools`, ensure your base image has universe enabled:

```bash
sudo apt-get update
sudo add-apt-repository universe
sudo apt-get install -y openstack-pkg-tools
```

### tox environments fail with missing system packages

Run `tox -e bindep` in the project directory. It will list missing OS-level dependencies. Install them with `sudo apt-get install <packages>`.

---

## Documentation and Guidance

| Resource | URL |
|---|---|
| OpenStack Contributor Guide | https://docs.openstack.org/contributors/ |
| Ubuntu OpenStack Packaging | https://wiki.ubuntu.com/OpenStack/CorePackages |
| Charmed OpenStack | https://charmhub.io/openstack |
| Sunbeam Documentation | https://docs.openstack.org/sunbeam/ |
| orange-dev-tools SDK | https://github.com/freyes/sdk-orange-dev-tools |
| Workshop Documentation | https://ubuntu.com/workshop/docs/ |
| SDKcraft Documentation | https://ubuntu.com/workshop/docs/reference/cli/sdkcraft/ |

---

## Community and Support

- Report issues or request features on the [GitHub repository](https://github.com/freyes/sdk-openstack-dev-tools)
- Follow the [Ubuntu Code of Conduct](https://ubuntu.com/community/ethos/code-of-conduct)

---

## License and Copyright

Copyright 2026 Felipe Reyes &lt;freyes@tty.cl&gt;.

Licensed under the GPL-3.0 license.