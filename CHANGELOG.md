# Changelog

All notable changes to BadBotSearch.

## [0.2.2] - 2026-11-19

### Added
- `badbot scan [path]` positional folder scan: `badbot scan /home`, `badbot scan /Users/beatriz --quick` on macOS, `badbot scan --path /tmp` explicit flag kept for compat (previously positional arg was ignored and scanned `/` with 2.9M files)
- Prebuilt binaries layout: `bin/linux/badbot_v0.2.1` + `bin/osx/badbot_v0.2.1` (arm64 default), `bin/osx/badbot_v0.2.1_amd64` (Intel), `bin/osx/badbot_v0.2.1_arm64` (Apple Silicon) — cross-compiled with `CGO_ENABLED=0` + `modernc.org/sqlite`, root `badbot -> bin/linux/badbot_v0.2.1` symlink

### Changed
- `badbot scan` help updated (examples for macOS, `--quick`, positional path)
- `badbot update --hashes` now incremental via unified `https://apis.grandekos.com/malware/hashes/manifest.json` with `X-API-Key` entitlement (`trial`/`paid`, auto-downgrade if expired), state `configs/db/last_update.json` `{last_date,count,entitlement,tier}`, range `--from/--to` + hourly `2026-11-15-10` parsing fixed (`parseBadbotDate` handles `YYYY-MM-DD` and `YYYY-MM-DD-HH`)
- Hashes fetcher now uses progress bar (`github.com/schollz/progressbar/v3`) with width 30, count + ETA, clears on finish, replacing endless `... fetched 1900/2674` spam

### Fixed
- `invalid from date: parsing time "2026-11-15-10": extra text "-10"` when `last_update.json` contained hourly suffix — now both daily and hourly formats parse
- `VerifyDebianPackages` on macOS: `dpkg -V` missing now reports `[*] Skipped package verify: dpkg not available on darwin — Debian/Ubuntu only` via `IsSkippedOnThisOS()` instead of `[!] Package verify error`
- macOS Phase 1 validated: `badbot scan /Users/beatriz/` works (Beatriz iMac test: 2.9M-file full scan passed, fixed to respect target folder)

## [0.2.1] - 2026-10-29

### Added
- `badbot version` command (shows version, commit, build date)
- `badbot stats` command (IOCs loaded, hash DB, daily staging, cache, allowlist, quarantine)
- `badbot import` / `badbot --import` / `badbot -i` (import daily SHA256 JSONs from `/home/t3ch/Working/Hobies/vulndb/daily/*.json` into SQLite `hashes.db`, incremental dedup, 77k hashes tested)
- `badbot scan --hashdb` support (hash DB lookup via SQLite, single hash per file)
- Initial **macOS support** (Phase 1, experimental):
  - Core scanner (`scan`, `import`, `stats`, `quarantine`) already cross-platform (pure Go, SQLite)
  - Build: `GOOS=darwin GOARCH=arm64 go build -o badbot` works
  - Platform abstraction prepared: `internal/packages/linux.go` (current) + `internal/packages/darwin.go` placeholder
  - Phase 1 tested: hash DB + IOC scan works on macOS
  - Phase 2 planned (after testing): `check` OSX adapters (`brew` pkg verify, `otool -L sshd` for liblzma, `launchctl` + LaunchAgents/LaunchDaemons persistence)

### Changed
- `configs/config.yaml` documents multi-source merge + harddisk cache
- `internal/scanner` now respects explicit `/tmp` scans (only skips /tmp when scanning from `/`)

### Fixed
- Scanner `OR` -> `AND` logic for multi-criteria rules (fixes JIATAN-002 false positive on `/usr/share/aclocal/build-to-host.m4`)
- `PERSIST-002` false positives (40 services) replaced by `dpkg -S` orphan check + allowlist

## [0.2.0] - 2026-10-28

- `badbot check --interactive` + `badbot allowlist` (exact + glob)
- `badbot update` 3 modes: GitHub, plain HTTPS, HTTPS Basic Auth (with env var support, ETag caching, harddisk fallback)
- `internal/update` adapters for CISA KEV / NVD / OSV (CISA KEV tested: 1.8M -> 1760 rules, 304 cache)
- `internal/allowlist` + `internal/packages` orphan detection
- `internal/hashdb` SQLite hash DB groundwork

## [0.1.0] - 2026-10-28

- Initial MVP: Go + Cobra, Debian/Ubuntu focus
- `badbot check` (xz 5.6.0/5.6.1, sshd liblzma, persistence)
- `badbot scan` (IOC YAML path_regex/contains/sha256)
- `badbot quarantine` (vault)
- `configs/iocs.yaml` (JIATAN-001/002, PERSIST-001, SUPPLY-001)
