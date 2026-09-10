# beenergy-infra

Infrastructure automation for BeEnergy.

This repository currently contains an Ansible role that installs and configures
**dnsmasq** as a caching DNS forwarder on Debian/Ubuntu hosts.

---

## What this does

dnsmasq is a lightweight DNS forwarder and cache. It does not perform recursive
resolution itself: it forwards queries to upstream resolvers and caches the
answers, which makes repeat lookups near-instant. It also serves local hostnames
and supports wildcard and conditional forwarding.

This role:

1. **Frees port 53.** On Ubuntu, `systemd-resolved` binds port 53 on `127.0.0.53`
   and `127.0.0.54` (UDP and TCP). dnsmasq cannot start while those sockets are
   held. The role disables the stub listener, waits for the port to be released,
   and refuses to continue if any other process still holds it.
2. **Installs dnsmasq** and ensures `/etc/dnsmasq.d/` exists.
3. **Writes configuration** to `/etc/dnsmasq.d/10-ansible.conf` from a Jinja2
   template, validated with `dnsmasq --test` before the file is written.
4. **Points the system resolver** at dnsmasq via `/etc/resolv.conf`.
5. **Verifies the result** — service state, port ownership, absence of DHCP,
   and actual name resolution.

**DHCP is intentionally disabled.** dnsmasq only starts its DHCP service when a
`dhcp-range` is defined. None is defined here, and the verification step asserts
that nothing is listening on port 67.

---

## Requirements

| Component | Version |
|---|---|
| Ansible | core 2.14 or later |
| Target OS | Debian or Ubuntu |
| Privileges | `sudo` access on the target host |

Tested on Ubuntu Server 26.04 LTS with ansible-core 2.20.1.

---

## Quick start

```bash
git clone git@github.com:lopparg/beenergy-infra.git
cd beenergy-infra

# Check what would change, without changing anything
ansible-playbook playbooks/site.yml --check --diff -K

# Apply
ansible-playbook playbooks/site.yml -K
```

`-K` prompts for the sudo password. See the sudo-rs note below if you are on
Ubuntu 25.10 or later.

### Verify manually

```bash
sudo ss -tulpn | grep ':53'          # dnsmasq should be the only listener
systemctl is-active dnsmasq
dig @127.0.0.1 github.com +short
dig @127.0.0.1 github.com | grep "Query time"   # second call should be 0 msec
sudo ss -ulpn | grep ':67'           # should return nothing (no DHCP)
```

---

## Repository layout

```
.
├── ansible.cfg                      # Inventory path, roles path, output format
├── inventory/hosts.ini              # Target hosts
├── group_vars/dns_servers.yml       # Group-level overrides (committed)
├── local.example.yml                # Template for local overrides (copy to local.yml)
├── playbooks/site.yml               # Entry point
└── roles/dnsmasq/
    ├── defaults/main.yml            # All tunable variables
    ├── tasks/
    │   ├── main.yml                 # Orchestration
    │   ├── preflight.yml            # Port 53 conflict resolution
    │   ├── install.yml              # Package installation
    │   ├── configure.yml            # Config, service, resolv.conf
    │   └── verify.yml               # Post-deployment assertions
    ├── handlers/main.yml            # Service restarts
    ├── templates/dnsmasq.conf.j2    # Config template
    └── meta/main.yml                # Role metadata
```

---

## Configuration

Every setting lives in `roles/dnsmasq/defaults/main.yml`. Nothing is hard-coded
in the tasks or the template.

| Variable | Default | Purpose |
|---|---|---|
| `dnsmasq_upstream_servers` | `[1.1.1.1, 8.8.8.8]` | Where queries are forwarded |
| `dnsmasq_listen_addresses` | `[127.0.0.1]` | Addresses dnsmasq binds to |
| `dnsmasq_local_domain` | `beenergy.internal` | Internal domain suffix |
| `dnsmasq_static_hosts` | one test record | Static name-to-IP mappings |
| `dnsmasq_cache_size` | `1000` | Cached entries |
| `dnsmasq_enable_dhcp` | `false` | Documents that DHCP is off |
| `dnsmasq_no_resolv` | `true` | Prevents a query loop (see below) |
| `dnsmasq_manage_systemd_resolved` | `true` | Disable the stub listener |
| `dnsmasq_manage_resolv_conf` | `true` | Rewrite `/etc/resolv.conf` |
| `dnsmasq_log_queries` | `false` | Verbose query logging |
| `dnsmasq_verify_enabled` | `true` | Run post-deployment assertions |

### Overriding for your own environment

The defaults use public resolvers so that anyone cloning this repository gets
the same result. To point at your own network instead:

```bash
cp local.example.yml local.yml
nano local.yml
ansible-playbook playbooks/site.yml -e @local.yml -K
```

`local.yml` is gitignored, so your environment never lands in the repository.

Alternatively, pass values directly:

```bash
ansible-playbook playbooks/site.yml -e '{"dnsmasq_upstream_servers":["192.168.1.1"]}' -K
```

Variable precedence, weakest to strongest:

```
roles/dnsmasq/defaults/ < group_vars/ < host_vars/ < --extra-vars
```

### Running selected stages

```bash
ansible-playbook playbooks/site.yml --tags preflight -K
ansible-playbook playbooks/site.yml --tags config -K
ansible-playbook playbooks/site.yml --tags verify -K
```

---

## Design decisions

### Drop-in config instead of editing `/etc/dnsmasq.conf`

Configuration is written to `/etc/dnsmasq.d/10-ansible.conf` rather than
replacing the distribution's main config file. `/etc/dnsmasq.conf` already
contains `conf-dir=/etc/dnsmasq.d/,*.conf`, so the drop-in is picked up
automatically. This keeps package upgrades from clobbering our settings and
makes the Ansible-managed surface explicit.

### `no-resolv` is required, not optional

The role points `/etc/resolv.conf` at `127.0.0.1`. If dnsmasq also read that
file to discover upstream servers, it would treat itself as its own upstream and
loop forever. `no-resolv` plus explicit `server=` entries prevents this.

### Config is validated before it is written

The `template` task uses `validate: "dnsmasq --test --conf-file=%s"`. Ansible
writes the rendered file to a temporary location, runs the syntax check, and
only moves it into place if the check passes. A malformed template can never
take the service down.

### Ordering around `/etc/resolv.conf`

dnsmasq must be running and answering on port 53 *before* the resolver is
repointed at it. `wait_for` enforces this. The file is overwritten in place with
`follow: false` rather than deleted and recreated, so the host is never left
without a resolver if the run fails partway through.

### The port 53 assertion is scoped to non-dnsmasq processes

An earlier version asserted that port 53 must be completely free. That passed on
a first run but failed on every subsequent run, because dnsmasq itself is the
listener by then — the playbook flagged its own service as a conflict. The
assertion now checks that no process *other than dnsmasq* holds the port, which
is both correct and idempotent.

### `.internal`, not `.local`

`.local` is reserved for mDNS/Bonjour by RFC 6762 and can collide with Avahi on
Linux desktops. `beenergy.internal` avoids that.

### `become` is not enabled globally

`ansible.cfg` sets `become = false`. Each task that needs root declares it
explicitly. This makes the privilege surface visible rather than implicit.

---

## Known issues

### sudo-rs on Ubuntu 25.10 and later

Ubuntu 25.10 switched the default `sudo` to **sudo-rs**, a memory-safe Rust
reimplementation. It rejects the way Ansible feeds the become password over
stdin and fails with:

```
sudo: interactive authentication is required
Task failed: Premature end of stream waiting for become success.
```

The original sudo remains installed as `/usr/bin/sudo.ws`. `playbooks/site.yml`
contains a `pre_task` that checks for that binary and sets `ansible_become_exe`
only when it exists, so the playbook works on both sudo-rs and traditional-sudo
systems without any extra flags.

To handle it manually instead:

```bash
ansible-playbook playbooks/site.yml -e 'ansible_become_exe=/usr/bin/sudo.ws' -K
```

### Recovering if DNS breaks

If a run fails partway through and the host loses name resolution:

```bash
sudo rm -f /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

---

## Troubleshooting

**dnsmasq will not start: `Address already in use`**

Something else holds port 53. Find it:

```bash
sudo ss -tulpn | grep ':53'
```

If it is `systemd-resolve`, the preflight stage should have handled it. Check
that `/etc/systemd/resolved.conf` contains `DNSStubListener=no` and restart
`systemd-resolved`.

**Queries time out**

Verify dnsmasq is listening and that upstream servers are reachable:

```bash
systemctl status dnsmasq
sudo journalctl -u dnsmasq -n 50
dig @1.1.1.1 github.com +short
```

**Cache does not appear to work**

Run the same query twice and compare `Query time`. The second should be 0 msec.
If not, check `cache-size` in `/etc/dnsmasq.d/10-ansible.conf`.

**Enable query logging while debugging**

```bash
ansible-playbook playbooks/site.yml -e 'dnsmasq_log_queries=true' -K
sudo tail -f /var/log/dnsmasq.log
```

---

## Testing environment

Developed and verified on a local VMware Workstation VM:

- Ubuntu Server 26.04 LTS
- 2 GB RAM, 2 vCPU, 20 GB disk
- NAT networking

The playbook targets `localhost` with `ansible_connection=local`, so the VM acts
as both control node and managed node. To deploy to a remote host instead,
replace the entry in `inventory/hosts.ini`:

```ini
[dns_servers]
dns01 ansible_host=10.0.0.10 ansible_user=ubuntu
```

---

## Versioning

This project follows [Semantic Versioning](https://semver.org/). See
[CHANGELOG.md](CHANGELOG.md) for the release history.

---

## Contributing

Work happens on feature branches, never directly on `main`. Commit messages
follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat(dnsmasq): add support for conditional forwarding
fix(preflight): scope port 53 assertion to non-dnsmasq processes
docs(readme): document sudo-rs workaround
```

Before opening a pull request:

```bash
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml --check --diff -K
ansible-playbook playbooks/site.yml -K          # run twice
ansible-playbook playbooks/site.yml -K          # second run must report changed=0
```
