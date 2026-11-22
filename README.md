# BadBotSearch

Spybot-like scanner for Linux (+ macOS experimental), focused on supply-chain threats like **Jia Tan / xz-utils (CVE-2024-3094)**.

Terminal only, written in Go with Cobra CLI. No GUI. Debian/Ubuntu first, macOS Phase 1 supported.

> Inspired by Spybot Search & Destroy for Windows - but rebuilt for Linux supply-chain attacks: social-engineered maintainer takeovers, obfuscated build files, backdoored tarballs.

## How It Works

Three layers like Spybot, adapted for Linux:

1. **Artifact Scanner** - Filesystem walk + IOC matching (path_regex, contains, sha256)
2. **Package Provenance Scanner** - `dpkg -V` verification, xz version check, sshd->liblzma linkage, LD_PRELOAD
3. **Persistence Detector** - systemd, cron, LD_PRELOAD with allowlist support (no false positives)

YARA planned for v0.2 (free BSD license, currently YAML engine covers 80%).

## Installation

```bash
git clone https://github.com/badbotsearch/badbot
cd BadBotSearch
go build -o badbot .
sudo install -m 755 badbot /usr/local/bin/badbot

# Install configs
sudo mkdir -p /etc/badbot
sudo cp configs/iocs.yaml /etc/badbot/iocs.yaml
sudo cp configs/config.yaml.example /etc/badbot/config.yaml
sudo cp configs/allowlist.yaml.example /etc/badbot/allowlist.yaml
# or keep local for development: configs/iocs.yaml, configs/allowlist.yaml work as fallback
```

**Requirements:** Go 1.24+. Linux: `dpkg`, `ldd`, `xz` (Debian/Ubuntu). macOS: `brew`, `otool` (Phase 2, optional - hash scan works without). No other dependencies - single static binary.

**Platforms:**
- **Linux (primary, tested):** Debian/Ubuntu - full `check`/`scan`/`import`/`stats` support
- **macOS (experimental, Phase 1 - 2026-10-29):** Core `scan --hashdb`, `import`, `stats`, `quarantine` work cross-platform (`GOOS=darwin GOARCH=arm64 go build`). Phase 2 (brew/launchctl adapters) planned after Phase 1 testing.

## Quick Start

```bash
# 1. Health check (xz version, sshd linkage, persistence)
sudo badbot check
# --- 1. xz / liblzma version ---
# [*] xz (XZ Utils) 5.8.1
# --- 2. sshd linkage check ---
# [+] sshd NOT linked to liblzma (safe config)
# --- 3. Persistence mechanisms ---
# [+] LD_PRELOAD not set (ok)
# [+] All 1 systemd services are owned by dpkg (ok)

# 2. Full scan
sudo badbot scan --quick                    # quick mode: skips /proc /sys /snap, checks only xz/liblzma via dpkg -V
sudo badbot scan --path /                   # full root scan
sudo badbot scan --path /etc --quick        # scan specific path

# 3. Quarantine
badbot quarantine add ./suspicious_file
badbot quarantine list
badbot quarantine restore <id>
```

## Commands

### `badbot check` - Quick Health Check

```bash
badbot check
badbot check --interactive                  # prompt Y/n to allowlist orphaned services
badbot check --interactive --allowlist /etc/badbot/allowlist.yaml
badbot check --allowlist ./my-allowlist.yaml
```

Checks:
- **xz version:** detects vulnerable 5.6.0 / 5.6.1 (Jia Tan CVE-2024-3094). Warns if vulnerable + sshd linked to liblzma = exploitable.
- **sshd linkage:** `ldd /usr/sbin/sshd | grep liblzma` - if linked, higher risk.
- **Persistence:** 
  - `/etc/ld.so.preload` existence (file + env var)
  - systemd: finds orphaned services in `/etc/systemd/system/*.service` not owned by any `dpkg -S` package. Only orphaned are reported (no false positives on legit services like `bluetooth.service` or `ssh.service`). Respects allowlist.

**Example output after fix (no false positives):**
```
[*] Running quick health check...
--- 1. xz / liblzma version ---
[*] xz (XZ Utils) 5.8.1
--- 2. sshd linkage check ---
[+] sshd NOT linked to liblzma (safe config)
--- 3. Persistence mechanisms ---
  - [+] LD_PRELOAD not set (ok)
  - [?] Orphaned service (not owned by dpkg): /etc/systemd/system/ollama.service - review: cat /etc/systemd/system/ollama.service | use 'badbot check --interactive' to allowlist
  - [!] Found 1 orphaned systemd services
```

### `badbot scan` - Filesystem Scan

```bash
badbot scan [--quick] [--path <dir>]
# --quick: skip /home, /proc, /sys, /dev, /run, /snap, /tmp and use fast dpkg -V (only xz-utils/liblzma5)
# --path:  custom path, default /
```

Scans via IOC rules (`/etc/badbot/iocs.yaml` or `configs/iocs.yaml`):

- **path_regex:** regex on file path, e.g. `bad-3-corrupt_lzma2\.xz`
- **contains:** file contains string (only for files <2MB)
- **sha256:** hash match
- **AND logic:** if rule defines multiple criteria (e.g. `path_regex` + `contains`), ALL must match to trigger. Single-criterion rules behave as before.

Output:
```
[*] BadBotSearch scan starting...
[+] Loaded 4 IOC definitions
[*] Checking for xz/liblzma backdoor (CVE-2024-3094)...
[+] XZ/liblzma check passed
[*] Verifying Debian package integrity (dpkg -V)...
[+] No modified packages found
[*] Scanning filesystem: /etc (quick=true)
=== Scan Complete ===
Scanned: 1846 files
Hits: 0
[+] No threats found
```

### `badbot allowlist` - Trust Services

Fixes false positives. When `check` finds orphaned services (e.g. `ollama.service` installed outside apt), you can trust them.

```bash
# List
badbot allowlist list
badbot allowlist list --allowlist /etc/badbot/allowlist.yaml

# Add exact path (exact match)
badbot allowlist add /etc/systemd/system/ollama.service

# Add glob pattern (e.g. all dev services)
badbot allowlist add "/etc/systemd/system/custom-*.service" --exclude

# Remove
badbot allowlist remove /etc/systemd/system/ollama.service

# Interactive (recommended)
badbot check --interactive
# [?] /etc/systemd/system/ollama.service
#     Allowlist this service? (y/N/exclude): y
#     [+] Added /etc/systemd/system/ollama.service to allowlist
```

**Allowlist file:** `/etc/badbot/allowlist.yaml` (system, preferred if writable) or `configs/allowlist.yaml` (local fallback, used in dev). Structure:

```yaml
# BadBotSearch allowlist - services you trust, won't be flagged again
# Managed via: badbot allowlist list|add|remove and badbot check --interactive
services:
  - /etc/systemd/system/ollama.service
exclude:
  - /etc/systemd/system/custom-*.service
```

`exclude` supports glob via `filepath.Match`, e.g. `custom-*.service`. Once allowlisted, `badbot check` shows:
```
[+] All orphaned services are allowlisted (1 allowed) - clean
```

You can manually edit the YAML too.

### `badbot update` - Update IOCs (3 modes)

All via HTTPS, no paywall. Config searched: `/etc/badbot/config.yaml`, `configs/config.yaml`, `./config.yaml`.

```bash
# 1. GitHub (public community, default)
badbot update
badbot update --github https://raw.githubusercontent.com/badbotsearch/iocs/main/iocs.yaml
# Sets Accept: application/vnd.github.v3.raw

# 2. Plain HTTPS (no auth) - your own server
badbot update --url https://example.com/iocs.yaml
badbot update --url http://127.0.0.1:8000/iocs.yaml --dest /tmp/iocs.yaml

# 3. HTTPS with Basic Auth (user+pass)
badbot update --url https://private.example.com/iocs.yaml --user admin --pass secret123
badbot update --url https://private.example.com/iocs.yaml --user admin --pass secret123 --dest /etc/badbot/iocs.yaml

# Via env vars in config.yaml (more secure, don't hardcode password)
export BADBOT_USER=admin BADBOT_PASS=secret123
# config.yaml:
# sources:
#   - name: private
#     type: https-auth
#     url: https://private.example.com/iocs.yaml
#     username: ${BADBOT_USER}
#     password: ${BADBOT_PASS}
badbot update

# Override dest
badbot update --dest /tmp/test.yaml
```

**Config file example** (`configs/config.yaml.example`):
```yaml
dest: /etc/badbot/iocs.yaml
sources:
  - name: official
    type: github
    url: https://raw.githubusercontent.com/badbotsearch/iocs/main/iocs.yaml
  # - name: custom
  #   type: https
  #   url: https://example.com/iocs.yaml
  # - name: private
  #   type: https-auth
  #   url: https://private.example.com/iocs.yaml
  #   username: ${BADBOT_USER}
  #   password: ${BADBOT_PASS}
```

Multiple sources supported - last successful fetch wins (merge planned). Validation: must contain `rules:` and be valid YAML, otherwise fetch fails.

### `badbot quarantine` - Isolate Files

```bash
badbot quarantine add ./suspicious_binary
badbot quarantine list
badbot quarantine restore <id>
```

Vault: `/var/lib/badbot/quarantine` (needs root) or `./quarantine` fallback (no root). Saves original path + timestamp as JSON meta (`<id>_<filename>.json`). Original file is removed after copy.

## IOC Definitions (`iocs.yaml`)

Located at `/etc/badbot/iocs.yaml` (system) or `configs/iocs.yaml` (fallback). Hot-reloadable without recompile.

**Current rules (v0.1):**

| ID | Name | Severity | Criteria |
|----|------|----------|----------|
| JIATAN-001 | xz backdoor test blob | critical | `path_regex: bad-3-corrupt_lzma2\.xz` |
| JIATAN-002 | xz build-to-host.m4 backdoor | critical | `path_regex: (xz\|liblzma).*build-to-host\.m4` + `contains: eval` (AND, only in xz tree) |
| PERSIST-001 | LD_PRELOAD file | high | `path_regex: ^/etc/ld\.so\.preload$` |
| SUPPLY-001 | Hidden binary in tests/ | high | `path_regex: tests/.*\.xz` |

**Why no PERSIST-002 false positives?** Old broad regex `/etc/systemd/system/.*\.service` was removed (flagged every legit service). Now handled in code via `dpkg -S` ownership check + allowlist. `/usr/share/aclocal/build-to-host.m4` (gettext) no longer flagged because JIATAN-002 now requires xz context.

**Add custom rule:**
```yaml
rules:
  - id: CUSTOM-001
    name: "My backdoor sig"
    severity: critical
    sha256: "abc123..."
  - id: CUSTOM-002
    name: "Suspicious string"
    severity: high
    contains: "evil_payload"
    path_regex: "suspicious_dir/.*"
```

Update via `badbot update` to get latest community feed.

## Project Structure

```
BadBotSearch/
├── main.go
├── cmd/
│   ├── root.go           # cobra root
│   ├── scan.go           # scan + dpkg -V + xz check
│   ├── check.go          # health check + interactive allowlist
│   ├── quarantine.go     # vault
│   ├── update.go         # 3-mode updater (github/https/https-auth)
│   └── allowlist.go      # allowlist CRUD
├── internal/
│   ├── ioc/ioc.go        # YAML loader + regex compile
│   ├── scanner/scanner.go# WalkDir + AND logic + hash/contains
│   ├── packages/debian.go# GetXZVersion, IsSSHDLinkedToLZMA, CheckPersistenceWithAllowlist
│   ├── quarantine/quarantine.go # VaultDir logic
│   ├── update/update.go  # Fetch with BasicAuth, Save, LoadOrDefault
│   └── allowlist/allowlist.go # Allowlist struct, IsAllowed, Add/Remove
└── configs/
    ├── iocs.yaml
    ├── iocs.yaml / iocs.yaml.example (moved to /etc on install)
    ├── config.yaml / config.yaml.example
    └── allowlist.yaml / allowlist.yaml.example
```

## Platform Details

**Linux (Debian/Ubuntu, primary):**
- **Package verify:** `dpkg -V` (quick mode: `dpkg -V xz-utils liblzma5` only). Parses output lines.
- **xz version:** `xz --version` -> first line check for `5.6.0`/`5.6.1`.
- **sshd linkage:** `ldd /usr/sbin/sshd | grep liblzma` (+ fallback `which sshd`).
- **Persistence:** `find /etc/systemd/system -type f -name '*.service'` + `dpkg -S` per file + `ls -l /etc/ld.so.preload` + `echo $LD_PRELOAD`.

**macOS (experimental, Phase 1):**
- **Phase 1 (now, tested):** Hash DB + IOC scan only. Build: `GOOS=darwin GOARCH=arm64 go build -o badbot` (and `GOOS=darwin GOARCH=amd64` for Intel). Works: `import`, `scan --hashdb`, `stats`, `quarantine`.
- **Phase 2 (planned after Phase 1 testing):** `check` OSX adapters: `brew list --versions` package verify, `otool -L $(which sshd) | grep lzma`, persistence via `launchctl list` + `/Library/LaunchDaemons` + `/Library/LaunchAgents` + `~/Library/LaunchAgents` + `DYLD_INSERT_LIBRARIES`. See `CHANGELOG.md` 0.2.1.

Arch/Fedora support via pacman/rpm adapters planned.

## Development

```bash
go version  # 1.24.4 tested
go mod tidy
go build -o badbot .
./badbot --help
./badbot check
./badbot scan --path ./configs --quick
./badbot allowlist list
./badbot update --url http://localhost:8765/configs/iocs.yaml --dest /tmp/test.yaml
```

**Testing update locally:**
```bash
python3 -m http.server 8765 &
./badbot update --url http://127.0.0.1:8765/configs/iocs.yaml --dest /tmp/iocs.yaml
```

## Roadmap

- v0.1: IOC YAML engine + debian verifier + quarantine + 3-mode update + allowlist
- v0.2: YARA support (free libyara), `immunize` via `apt-mark hold xz-utils`, JSON report `scan --json`
- v0.2.1 (current): `import`/`stats`/`version`, hash DB (77k hashes), macOS Phase 1 (cross-platform scan)
- v0.3: Supply-chain heuristic analyzer (git history, maintainer takeover detection for next Jia Tan) + macOS Phase 2 (brew/launchctl)
- Future: Arch (pacman), Fedora (rpm), npm/pip/cargo typosquat checks, TUI

See `CHANGELOG.md` for details.

## License & Disclaimer

For educational/defensive use. Don't quarantine system files blindly - review hits first. Use `allowlist` for trusted custom services.

```

