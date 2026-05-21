# Copilot Instructions for RaspiBlitz

Short guide for Copilot sessions working in RaspiBlitz: build/test/lint commands, architecture, and repository-specific conventions.

## Build, test, and lint commands

### Building images

- **Makefile targets** (recommended for building): Use from repo root
  - `make arm64-rpi-lean-image` — build for RaspberryPi 4/5 (ARM64)
  - `make amd64-lean-desktop-uefi-image` — build for x86-64 with desktop
  - `make amd64-lean-server-legacyboot-image` — build for x86-64 without desktop
  - These require `sudo`, `packer`, and system tools; consult `ci/README.md` for full setup

- **Direct script invocation** (useful for debugging):
  - `cd ci/arm64-rpi && sudo bash packer.build.arm64-rpi.local.sh --pack lean`
  - `cd ci/amd64 && sudo bash packer.build.amd64-debian.sh --pack lean --boot uefi --desktop gnome`

### Testing

- **Shell/config script tests** (bats-based):
  - `cd test && sudo bats ./bonus.postgresql-15.bats` — run single test file with sudo (required)
  - `bats --verbose-run ./bonus.postgresql-15.bats` — add --verbose-run flag for detailed output
  - Test files live in `test/*.bats` and target specific config scripts (e.g., `bonus.postgresql*.sh`)

- **Python unit tests**:
  - `cd home.admin/BlitzPy && pytest -q` — run all tests quietly
  - `cd home.admin/BlitzPy && pytest path/to/test_file.py::test_name` — run single test
  - Same pattern for BlitzTUI: `cd home.admin/BlitzTUI && pytest`
  - Both packages use pytest configuration in `setup.cfg`

### Linting

- **ShellCheck** (enforced in CI via `.github/workflows/test-shellcheck.yml`):
  - `shellcheck home.admin/config.scripts/*.sh` — check all config scripts
  - `shellcheck -S error path/to/script.sh` — check with error severity
  - Disable specific checks sparingly: `# shellcheck disable=SC1234` with SC code included

## High-level architecture

### Purpose & scope
RaspiBlitz builds Bitcoin/Lightning Fullnode OS images and provides device-side utilities for configuration. The repo contains bash/python scripts, CI automation, and tooling. Companion repos handle Web UI (raspiblitz-web) and API (blitz_api).

### Key directories

- **`ci/`** — Image build pipelines (packer + bash)
  - `arm64-rpi/` — Raspberry Pi 4/5 builds
  - `amd64/` — x86-64 builds (UEFI/legacy, with/without desktop)
  - Outputs: raw/qcow2 images + SHA256 checksums

- **`home.admin/`** — Device-side scripts and utilities (mirrors `/home/admin` on running device)
  - `config.scripts/` — Modular scripts for feature configuration (see naming patterns below)
  - `BlitzPy/` — Python utilities package (blitzpy module)
  - `BlitzTUI/` — Terminal UI for device operations (PyQt5-based)
  - `setup.scripts/` — Initial OS setup helpers
  - Other menu/utility scripts (`00*.sh`, `99*.sh`)

- **`test/`** — Bats integration tests (currently PostgreSQL bonus script tests)
  - Mirrors CI patterns; run locally with sudo to match CI environment

- **`.github/workflows/`** — CI/CD orchestration
  - `test-shellcheck.yml` — Lint all shell scripts (runs on PRs/dev)
  - `test-bats.yml` — Run integration tests for PostgreSQL configs
  - `*-image.yml` — Build ARM64/AMD64 images on push/workflow_dispatch

### Script organization & naming

**config.scripts/** contains modular feature installers following these patterns:

- `blitz.*.sh` — Core Blitz functionality (blitz.conf, blitz.data, blitz.systemd, blitz.ssh, etc.)
- `bitcoin.*.sh` — Bitcoin Core setup (install, update, check, testnet, monitor)
- `lnd.*.sh` — LND (Lightning Network Daemon) setup and management
- `cl.*.sh` — Core Lightning setup and management
- `cl-plugin.*.sh` — Core Lightning plugins
- `bonus.*.sh` — Optional features (lnbits, rtl, specter, btcpayserver, postgresql, etc.)
- `network.*.sh` — Network configuration (chain, aliases, txindex, reindex, wallet)
- `internet.*.sh` — Internet/connectivity (wifi, tailscale, zerotier, dyndomain, letsencrypt, dns)
- `tor.*.sh` — Tor integration (install, network, onion-service)

Each script is self-contained and often has:
- Install/update modes (triggered by system on boot)
- Menu integration (for interactive configuration)
- Status checking/health monitoring

## Key conventions and patterns

### Authoritative paths & device assumptions
- `home.admin/` scripts are designed to run on the RaspberryPi device with `/home/admin` as their runtime directory
- When modifying scripts, verify they work on both ARM64 (RPi) and x86-64 (desktop VM) if applicable
- Config scripts assume systemd services for most components

### Changes & testing workflow
1. Modify config.scripts or other home.admin content
2. If touching config.scripts, add corresponding bats test in `test/` (or update existing one)
3. Run linting locally: `shellcheck home.admin/config.scripts/your-script.sh`
4. Run integration tests locally with sudo: `cd test && sudo bats ./bonus.*.bats`
5. Push to PR; CI will run ShellCheck and Bats tests automatically

### CI mirrors local development
- The commands run in `.github/workflows/` (ShellCheck, Bats) should match local test runs
- If CI fails, reproduce locally first: same command, same sudo context

### Python package patterns
- Both BlitzPy and BlitzTUI use `setup.cfg` for pytest configuration
- Tests live alongside source code (check each package's structure)
- Use `pytest -q` for quiet output; add specific test paths to run subsets

### Image building
- Makefile targets are thin wrappers—use them for standard builds
- For debugging packer issues, invoke packer scripts directly from `ci/`
- Build artifacts (images + checksums) go to `ci/{platform}/builds/`

## Documentation sources

- **README.md** — Project overview, links to docs/API/WebUI
- **CONTRIBUTING.md** — Community development workflow, PR review criteria, philosophy
- **ci/README.md** — Detailed image building, packer setup, flashing instructions
- **test/README.md** — Bats test running instructions
- **.github/workflows** — Live CI patterns for testing and building
- **CHANGES.md** — Release changelog

## No additional AI assistant configs

No CLAUDE.md, .cursorrules, AGENTS.md, or similar files exist in this repo.
