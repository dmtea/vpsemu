---
name: vpsemu
version: 0.36
description: >
  Manage Incus containers that emulate VPS servers (Ubuntu 24.04).
  Use this skill to create, check, reset and delete test machines
  for Ansible / IaC tasks. Supports parallel agents via strict
  namespace vpsemu-{AGENT_ID}-{TASK_NAME}. Never touch containers
  with a foreign AGENT_ID.
---

# vpsemu

## Namespace

```
vpsemu-{AGENT_ID}-{TASK_NAME}

Examples:
  vpsemu-a3f9-deploy-nginx
  vpsemu-b7c1-setup-db
```

- `vpsemu` — static prefix, always
- `AGENT_ID` — 4-hex, generated **once at session start**, lives in context only
- `TASK_NAME` — short task name, assigned by agent

---

## ⚡ Initialization (required on skill load)

As soon as this skill is loaded, agent **immediately** runs:

```bash
AGENT_ID=$(openssl rand -hex 2)   # e.g.: a3f9
```

Agent **remembers `AGENT_ID` as a critical session fact**. On context compression — restore from memory, never regenerate.

### Fork detection

opencode session fork copies full parent context including `AGENT_ID`. Two parallel forks would share the same ID and collide on containers.

**On initialization, agent must verify ownership:**

```bash
# Check if any containers with current AGENT_ID exist but were NOT created in this session
EXISTING=$(incus list --format csv | awk -F',' '{print $1}' | grep "^vpsemu-${AGENT_ID}-")

if [ -n "$EXISTING" ]; then
  # AGENT_ID was inherited from a forked/parent session — regenerate
  AGENT_ID=$(openssl rand -hex 2)
  echo "FORK DETECTED: new AGENT_ID=${AGENT_ID}"
fi
```

This check is safe: if the agent itself created those containers earlier in the same session, it knows about them from context — and will recognize them. If it does NOT recognize them, they belong to a parent/sibling session → regenerate.

---

## Bootstrap (run once by owner before first agent launch)

### 1. Install Incus

**Ubuntu 24.04 (noble) — native package, no external repo needed:**
```bash
sudo apt install incus
```

**Ubuntu 22.04 (jammy) / Pop!_OS 22.04 — via Zabbly repository:**
```bash
# Verify key fingerprint matches: 4EFC 5906 96CB 15B8 7C73 A3AD 82CC 8797 C838 DCFD
curl -fsSL https://pkgs.zabbly.com/key.asc | gpg --show-keys --fingerprint

sudo mkdir -p /etc/apt/keyrings
sudo curl -fsSL https://pkgs.zabbly.com/key.asc \
  -o /etc/apt/keyrings/zabbly.asc

sudo sh -c 'cat <<EOF > /etc/apt/sources.list.d/zabbly-incus-stable.sources
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo ${VERSION_CODENAME})
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/zabbly.asc
EOF'

sudo apt update && sudo apt install -y incus
```

### 2. Common setup (both versions)

```bash
sudo usermod -aG incus-admin $USER && newgrp incus-admin
incus admin init --minimal

# Isolated network — NAT for internet, DHCP disabled (IPs are set inside containers)
incus network create incusbr1 \
  ipv4.address=10.10.0.1/24 \
  ipv4.dhcp=false \
  ipv4.nat=true \
  ipv6.address=none

# Base profile — clean, no nictype/ipv4_filtering (both cause issues with "network:" property)
incus profile create vps-base
incus profile edit vps-base << 'EOF'
config:
  security.nesting: "false"
devices:
  eth0:
    name: eth0
    network: incusbr1
    type: nic
  root:
    path: /
    pool: default
    size: 10GB
    type: disk
name: vps-base
EOF

# Template container — images:ubuntu/24.04 is a minimal image (no cloud-init, fast)
incus launch images:ubuntu/24.04 vps-template --profile vps-base

# Wait for network stack to be ready (no cloud-init in minimal image)
until incus exec vps-template -- true 2>/dev/null; do sleep 1; done
sleep 3

# Set static IP inside container (Incus ipv4.address only filters, doesn't configure)
incus exec vps-template -- bash -c "
  ip addr add 10.10.0.2/24 dev eth0
  ip route add default via 10.10.0.1
  rm -f /etc/resolv.conf
  echo 'nameserver 8.8.8.8' > /etc/resolv.conf
"

# Verify connectivity before installing packages
incus exec vps-template -- ping -c 1 8.8.8.8 || { echo 'ERROR: no internet in container'; exit 1; }

# Install packages and configure SSH
incus exec vps-template -- bash -c "
  apt-get update -qq
  apt-get install -y openssh-server python3 python3-apt -qq
  apt-get clean
  printf "PermitRootLogin yes\nPasswordAuthentication yes\n" \
    > /etc/ssh/sshd_config.d/99-vpsemu.conf
  systemctl restart ssh
"

# Snapshot stores clean OS without password and without IP — both are set by agent after start
incus snapshot create vps-template clean-vps
incus stop vps-template
```

> **Note:** The snapshot does NOT contain a static IP — the container boots without IP.
> The agent sets IP + password immediately after every `incus start` or `incus snapshot restore`.
> Also, `/etc/resolv.conf` must be removed before writing (it may be a protected symlink in minimal images).

---

## Variables

```bash
# AGENT_ID is taken from session context (generated on skill load)
TASK_NAME="deploy-nginx"               # agent assigns based on task
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)

NAME="vpsemu-${AGENT_ID}-${TASK_NAME}"
PREFIX="vpsemu-${AGENT_ID}"
```

### Get ROOT_PASS

Agent runs this logic **before creating or resetting a container**.
Default secrets path: `ansible/group_vars/all/secrets.enc.yml`

```bash
_get_root_pass() {
  local project_root="${1}"
  local secrets_file="${project_root}/ansible/group_vars/all/secrets.enc.yml"
  local key="vault_root_password"

  if [ -f "${secrets_file}" ]; then
    local pass
    pass=$(sops -d "${secrets_file}" 2>/dev/null \
      | python3 -c "import sys,yaml; d=yaml.safe_load(sys.stdin); print(d.get('${key}',''))" 2>/dev/null)
    if [ -n "${pass}" ]; then
      echo "${pass}"
      return
    fi
    echo "WARN: key '${key}' not found in ${secrets_file}" >&2
  fi

  # No file or no key — agent asks user in dialog, then sets ROOT_PASS manually
  return 1
}

if ! ROOT_PASS=$(_get_root_pass "${PROJECT_ROOT}"); then
  echo "ACTION_REQUIRED: enter root password for container"
fi
```

| Situation | Action |
|---|---|
| `secrets.enc.yml` found, key exists | use value from SOPS |
| `secrets.enc.yml` found, key missing | warn user, ask in dialog |
| `secrets.enc.yml` not found | ask user: provide path or enter password directly |
| User provided password in dialog | store in `ROOT_PASS` in context, never log |

### IP allocation — deterministic, no collisions

Same name always gives same IP. Parallel agents never collide:

```bash
IP_LAST=$(python3 -c "
import hashlib
h = int(hashlib.md5('${NAME}'.encode()).hexdigest(), 16)
print(10 + (h % 240))  # range 10.10.0.10 – 10.10.0.249
")
IP="10.10.0.${IP_LAST}"
```

---

## Commands

### Check existence

```bash
incus list --format csv | awk -F',' '{print $1}' | grep -q "^${NAME}$" \
  && echo "EXISTS" || echo "NOT_FOUND"
```

### Create container

```bash
IP_LAST=$(python3 -c "import hashlib; h=int(hashlib.md5('${NAME}'.encode()).hexdigest(),16); print(10+(h%240))")
IP="10.10.0.${IP_LAST}"

incus copy vps-template/clean-vps ${NAME}
incus start ${NAME}

# Wait for container to accept commands
until incus exec ${NAME} -- true 2>/dev/null; do sleep 1; done

# Set static IP inside container
incus exec ${NAME} -- bash -c "
  ip addr add ${IP}/24 dev eth0
  ip route add default via 10.10.0.1
  rm -f /etc/resolv.conf
  echo 'nameserver 8.8.8.8' > /etc/resolv.conf
"

# Set password
incus exec ${NAME} -- bash -c "echo 'root:${ROOT_PASS}' | chpasswd"

# Wait for SSH
until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 root@${IP} true 2>/dev/null
do sleep 1; done

echo "READY name=${NAME} ip=${IP} user=root"
```

### Get IP

```bash
incus list ${NAME} -c 4 --format csv | cut -d' ' -f1
```

### Status

```bash
incus list ${NAME} --format csv | awk -F',' '{print $2}'
# → RUNNING | STOPPED | FROZEN | (empty = does not exist)
```

### Stop / Start

```bash
incus stop  ${NAME}
incus start ${NAME}
```

### Reset to clean state

```bash
incus stop ${NAME} --force 2>/dev/null || true
incus snapshot restore ${NAME} clean-vps
incus start ${NAME}

# Wait for container to accept commands
until incus exec ${NAME} -- true 2>/dev/null; do sleep 1; done

# Re-set IP and password (snapshot has no IP or password)
incus exec ${NAME} -- bash -c "
  ip addr add ${IP}/24 dev eth0
  ip route add default via 10.10.0.1
  rm -f /etc/resolv.conf
  echo 'nameserver 8.8.8.8' > /etc/resolv.conf
"
incus exec ${NAME} -- bash -c "echo 'root:${ROOT_PASS}' | chpasswd"

until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 root@${IP} true 2>/dev/null
do sleep 1; done
echo "RESET_DONE name=${NAME} ip=${IP}"
```

### Delete

```bash
incus delete ${NAME} --force
```

### List own containers

```bash
incus list --format csv | awk -F',' '{print $1}' | grep "^${PREFIX}-"
```

### Delete all own containers

```bash
incus list --format csv | awk -F',' '{print $1}' | grep "^${PREFIX}-" \
  | xargs -r -I{} incus delete {} --force
```

---

## Ansible inventory

```ini
; single container
[vps]
${IP} ansible_user=root ansible_password=${ROOT_PASS} ansible_ssh_common_args='-o StrictHostKeyChecking=no'

; multiple containers per task
[web]
${IP_WEB} ansible_user=root ansible_password=${ROOT_PASS} ansible_ssh_common_args='-o StrictHostKeyChecking=no'
[db]
${IP_DB}  ansible_user=root ansible_password=${ROOT_PASS} ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

If password was obtained via dialog (fallback) — pass via env only, never write to file:

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory \
  -e "ansible_password=${ROOT_PASS}" playbook.yml
```

---

## Security rules (strictly required)

1. `AGENT_ID` is generated **once at session start** — never reassigned, never written to disk.
2. Before any mutation (`stop`, `delete`, `restore`) — verify `${NAME}` starts with `vpsemu-${AGENT_ID}-`.
3. Never run `incus delete` without explicit full `${NAME}`.
4. Never modify `vps-template` — it is a read-only source for `incus copy`.
5. Agent does not read or modify containers with a foreign `AGENT_ID`.
6. `ROOT_PASS` is never logged, never written to disk, never appears in container names — passed only at the moment of use.

---

## Bootstrap cleanup (owner only — full reset of Incus state)

```bash
#!/bin/bash
# Run if you need to completely redo bootstrap from scratch

set -e

# Stop and delete all vpsemu containers
for c in $(incus list --format csv | awk -F',' '{print $1}' | grep "^vpsemu-" || true); do
  incus delete "$c" --force 2>/dev/null || true
done

incus delete vps-template --force 2>/dev/null || true
incus profile delete vps-base 2>/dev/null || true
incus profile device remove default eth0 2>/dev/null || true
incus network delete incusbr1 2>/dev/null || true
incus network delete incusbr0 2>/dev/null || true

for img in $(incus image list --format csv | awk -F',' '{print $2}' || true); do
  [ -n "$img" ] && incus image delete "$img" 2>/dev/null || true
done

incus profile device remove default root 2>/dev/null || true
incus storage delete default 2>/dev/null || true

echo "Cleanup done. Remaining containers: $(incus list --format csv | wc -l)"
```
