# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2026-09-10

### Added

- Ansible role `dnsmasq` that installs and configures dnsmasq as a caching
  DNS forwarder.
- Preflight stage that detects processes holding port 53, disables the
  systemd-resolved DNS stub listener, waits for the port to be released,
  and refuses to continue if another process still holds it.
- Jinja2 template producing a drop-in configuration at
  `/etc/dnsmasq.d/10-ansible.conf`, validated with `dnsmasq --test`
  before it is written.
- Verification stage asserting that the service is active and enabled,
  that dnsmasq is the only process bound to port 53, that no DHCP server
  is listening on port 67, and that both public and internal names resolve.
- `local.example.yml` for environment-specific overrides.
- Documentation covering setup, usage, design decisions and troubleshooting.

### Changed

- `/etc/resolv.conf` is replaced with a static file pointing at 127.0.0.1.
  The previous symlink to systemd-resolved's stub file is overwritten in
  place rather than deleted first, so the host never loses DNS mid-run.

### Security

- The DHCP server is intentionally disabled. No `dhcp-range` is defined,
  so dnsmasq never starts its DHCP service.
- `bogus-priv`, `domain-needed` and `stop-dns-rebind` are enabled to avoid
  leaking private-range queries upstream and to reject rebinding answers.

[Unreleased]: https://github.com/lopparg/beenergy-infra/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/lopparg/beenergy-infra/releases/tag/v0.1.0
