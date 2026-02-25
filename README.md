# vpsemu

A skill for AI agents (opencode and compatible CLI agents) to create, manage and delete
Incus containers that emulate clean VPS servers — for testing Ansible playbooks and
IaC infrastructure.

---

## What and why

When developing Ansible/IaC projects you need an environment where you can quickly spin up
a clean server, run a playbook, verify it works — and reset. Renting a VPS for this is
expensive and slow. Docker doesn't cut it — systemd, SSH, network interfaces behave
differently from a real server.

Incus containers solve this: start in seconds, behave like a real VPS (full systemd, SSH,
static IP, root access), and cost nothing.

The skill gives the agent a ready set of commands + isolation rules so multiple agents
can work in parallel without interfering.

---

## Folder structure

```
vpsemu/
├── SKILL.md      — the skill itself, loaded by agent
├── SKILL.ru.md   — Russian version for reading/editing
├── README.md     — this file
└── README.ru.md  — Russian version of this file
```

---

## Requirements

- Ubuntu 22.04 (jammy) / Pop!_OS 22.04 or Ubuntu 24.04 (noble) as host
- `incus` — native via apt (24.04) or Zabbly repository (22.04), see Bootstrap in SKILL.md
- `sops` if using an encrypted secrets file
- `python3`, `python3-yaml` on host (for parsing SOPS output)
- Ansible project: expects structure `ansible/group_vars/all/secrets.enc.yml`

---

## Quick start

1. Run the Bootstrap section from `SKILL.md` — once on the host
2. Load `SKILL.md` into the agent (via skills folder or system prompt)
3. Agent auto-generates `AGENT_ID` on skill load and is ready

---

## Connecting to agent

opencode looks for skills in `<name>/SKILL.md` format. Two options:

**Project-level (recommended):**
```bash
mkdir -p .opencode/skills
git clone https://github.com/dmtea/vpsemu .opencode/skills/vpsemu
```

**Global (available in all projects):**
```bash
mkdir -p ~/.config/opencode/skills
git clone https://github.com/dmtea/vpsemu ~/.config/opencode/skills/vpsemu
```

On skill load the agent must immediately run `AGENT_ID` initialization —
described in the `⚡ Initialization` section inside `SKILL.md`.

---

## Container namespace

```
vpsemu-{AGENT_ID}-{TASK_NAME}

Examples:
  vpsemu-a3f9-deploy-nginx
  vpsemu-b7c1-setup-db
```

Each agent works only with its own containers. Parallel agents and tasks
never collide on names or IP addresses.

---

## Changelog

### v0.01 — initial version
Basic concept: Incus as a replacement for Vagrant for learning Ansible.
- Image: `ubuntu:22.04`
- Single container, manual SSH setup, root password access
- Isolated network `incusbr1` without DHCP, static IP set manually
- Base profile `vps-base`, template container `vps-template` with snapshot `clean-vps`
- Commands: create / SSH / stop / start / delete / snapshot / restore
- No isolation between agents — single user, single task

### v0.02 — multi-agent support via SESSION_ID
- Introduced namespace `{SESSION_ID}-{NAME}` to isolate containers between agents
- `SESSION_ID` generated at session start via `openssl rand -hex 3`
- Added deterministic IP allocation via MD5 hash of container name
  (range `10.10.0.10–249`) — eliminates collisions during parallel work
- Added commands: list own containers, delete all own
- Added security rules: agent does not touch containers with foreign prefix

### v0.03 — upgrade to Ubuntu 24.04
- Updated base image from `ubuntu:22.04` to `ubuntu:24.04`
- Added `python3-apt` to template dependencies (required for Ansible on 24.04)

### v0.10 — three-level namespace AGENT_ID + SESSION_ID + ROLE
- Namespace expanded to `{AGENT_ID}-{SESSION_ID}-{ROLE}` to support
  multiple agents, multiple tasks per agent, multiple containers per task
- Added static prefix `vpsemu-` for global identification of skill containers
- `AGENT_ID` — stable agent identifier, set at initialization
- `SESSION_ID` — session identifier, generated on each start
- `ROLE` — container role in task (web, db, cache...)
- Expanded security rules: agent does not touch containers with foreign `AGENT_ID`
- Added programmatic control example via REST API

### v0.20 — simplification: SESSION_ID removed, ROLE → TASK_NAME
- `SESSION_ID` removed as redundant — duplicated the meaning of `AGENT_ID`
- `ROLE` renamed to `TASK_NAME` — more accurately reflects the purpose
- Final namespace: `vpsemu-{AGENT_ID}-{TASK_NAME}`
- `AGENT_ID` — 4-hex, generated via `openssl rand -hex 2` at session start
- Skill renamed from `lxd-vps` to `vpsemu` (name + file)
- Section "One-time host setup" renamed to `Bootstrap (run by owner...)` to clearly
  distinguish what the human does vs what the agent does

### v0.21 — clarification of AGENT_ID storage
- Removed attempt to persist `AGENT_ID` to file `~/.vpsemu_agent_id`
- Reason: multiple agents can work from the same project folder simultaneously —
  a file on disk would cause collisions between them
- `AGENT_ID` lives exclusively in agent session context
- Added explicit requirement: on context compression agent restores `AGENT_ID`
  from memory, never regenerates

### v0.30 — SOPS integration for ROOT_PASS
- Removed hardcoded `ROOT_PASS="rootpass"`
- Added `_get_root_pass` function with three-level fallback:
  1. read `vault_root_password` from `ansible/group_vars/all/secrets.enc.yml` via `sops -d`
  2. file exists but key missing → warn user, ask in dialog
  3. file not found → ask user: provide another path or enter password directly
- Added `PROJECT_ROOT` variable via `git rev-parse --show-toplevel`
- Added security rule #6: `ROOT_PASS` never logged, never written to disk,
  in fallback passed only via `-e` at ansible invocation time
- Removed `pass=${ROOT_PASS}` from final `echo "READY..."` (log leak)

### v0.31 — fix password logic in template
- Found architectural bug: password was set in bootstrap as hardcode,
  inherited by snapshot, and did not match real password from SOPS
- Fix: password completely removed from template and snapshot
- Password is now set by agent via `incus exec ... chpasswd` right after
  `incus start` — both on create and after every `incus restore`
- Snapshot now stores clean OS without password, independent of secret rotation

### v0.32 — Ubuntu 24.04 technical fixes
- Fixed SSH setup: Ubuntu 24.04 `sshd_config` uses `Include sshd_config.d/*.conf`,
  so `sed` on the main file is unreliable. Now creates
  `/etc/ssh/sshd_config.d/99-vpsemu.conf` with explicit directives
- Fixed FALLBACK in `_get_root_pass`: function was returning string `"FALLBACK"`
  which got assigned to `ROOT_PASS` — now returns code `1` and agent
  stops to request password from user
- Replaced `sleep 3` with `until systemctl is-system-running` in two places
  (create and restore) — deterministic wait for systemd readiness
  instead of fixed delay

### v0.33 — fork session detection
- Added fork detection on initialization: opencode fork copies full parent context
  including `AGENT_ID`, which would cause two parallel forks to collide on containers
- On skill load agent checks if containers with current `AGENT_ID` exist but are
  unknown to this session — if yes, `AGENT_ID` is regenerated
- Safe by design: containers created in the current session are known from context;
  unrecognized containers indicate a fork → regenerate

### v0.34 — switch from LXD (snap) to Incus (apt)
- Replaced LXD with Incus — community fork of LXD, available via apt on Ubuntu 22.04/24.04
- Reason: LXD removed from standard apt repos in Ubuntu 24.04, snap-only; Incus is actively
  maintained by the community and installs cleanly via Zabbly repository
- Ubuntu 24.04: Incus available natively via `apt install incus` — no external repo needed
- Ubuntu 22.04 / Pop!_OS 22.04: install via Zabbly apt repository using `VERSION_CODENAME`
- Fixed Zabbly key filename: `zabbly.asc` (not `.gpg`) per official docs
- Added key fingerprint verification step per official Zabbly instructions
- snap install replaced with apt (native or Zabbly)
- Init command changed from `lxd init --minimal` to `incus admin init --minimal`
- Group changed from `lxd` to `incus-admin`
- All `lxc` CLI commands replaced with `incus` — API and behavior identical
- Compatible with Ubuntu 22.04 (jammy), 24.04 (noble) and derivatives incl. Pop!_OS 22.04

### v0.35 — bootstrap fixes from real run
- Fixed `curl` key download missing `sudo` — caused write permission error to `/etc/apt/keyrings/`
- Fixed container image: `ubuntu:24.04` remote doesn't exist by default → use `images:ubuntu/24.04`
- Fixed static IP assignment: `--config devices.eth0.ipv4.address` at launch doesn't work →
  use `incus config device override` after launch
- Fixed network: added `ipv4.nat=true` — without it containers have no internet access (apt fails)
- Fixed profile: added `security.ipv4_filtering=true` to eth0 device — required for static IP
  when DHCP is disabled on network
- Removed `cloud-init status --wait` — minimal `images:` image has no cloud-init, replaced with `sleep 5`
- Fixed snapshot command: `incus snapshot` → `incus snapshot create` (new Incus CLI syntax)
- SSH setup moved after `apt-get install openssh-server` — openssh not pre-installed in minimal image

### v0.36 — IP-inside-container architecture (from real run analysis)
- Root cause found: Incus `ipv4.address` in device is a FILTER, not IP configuration —
  container never gets IP from it regardless of settings
- Removed `nictype: bridged` — conflicts with `network:` property (validation error)
- Removed `security.ipv4_filtering` from profile — blocks all traffic when no `ipv4.address`
  is set in device at profile level; not useful without per-container IP override
- Reverted to `images:ubuntu/24.04` — minimal image, faster, no cloud-init dependency
- **New IP strategy**: set IP inside container via `ip addr add` + `ip route` after every
  `incus start` or `incus restore` — deterministic, cloud-init-independent, works on copy
- Replaced `sleep 5` with `until incus exec -- true` — reliable readiness check
- Added connectivity check after IP setup (`ping -c 1 8.8.8.8`) — fast fail with clear error
- Snapshot now explicitly contains neither IP nor password — both set fresh by agent each time
- Added Bootstrap cleanup script — full Incus state reset for re-bootstrap from scratch
