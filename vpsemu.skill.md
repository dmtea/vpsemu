---
name: vpsemu
version: 0.33
description: >
  Manage LXD containers that emulate VPS servers (Ubuntu 24.04).
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
EXISTING=$(lxc list --format csv | awk -F',' '{print $1}' | grep "^vpsemu-${AGENT_ID}-")

if [ -n "$EXISTING" ]; then
  # AGENT_ID was inherited from a forked/parent session — regenerate
  AGENT_ID=$(openssl rand -hex 2)
  echo "FORK DETECTED: new AGENT_ID=${AGENT_ID}"
fi
```

This check is safe: if the agent itself created those containers earlier in the same session, it knows about them from context — and will recognize them. If it does NOT recognize them, they belong to a parent/sibling session → regenerate.

---

## Bootstrap (run once by owner before first agent launch)

```bash
sudo snap install lxd
sudo usermod -aG lxd $USER && newgrp lxd
lxd init --minimal

# Isolated network, no DHCP
lxc network create lxdbr1 \
  ipv4.address=10.10.0.1/24 \
  ipv4.dhcp=false \
  ipv6.address=none

# Base profile
lxc profile create vps-base
lxc profile edit vps-base << 'EOF'
config:
  security.nesting: "false"
devices:
  eth0:
    name: eth0
    network: lxdbr1
    type: nic
  root:
    path: /
    pool: default
    size: 10GB
    type: disk
name: vps-base
EOF

# Template container
lxc launch ubuntu:24.04 vps-template --profile vps-base \
  --config devices.eth0.ipv4.address=10.10.0.2
lxc exec vps-template -- cloud-init status --wait

# Configure SSH — password is NOT set here
lxc exec vps-template -- bash -c "
  echo -e 'PermitRootLogin yes\nPasswordAuthentication yes' \
    > /etc/ssh/sshd_config.d/99-vpsemu.conf
  systemctl restart ssh
  apt-get update -qq && apt-get install -y python3 python3-apt -qq && apt-get clean
"

# Snapshot stores clean OS without password — password is always set by agent after start
lxc snapshot vps-template clean-vps
lxc stop vps-template
```

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
lxc list --format csv | awk -F',' '{print $1}' | grep -q "^${NAME}$" \
  && echo "EXISTS" || echo "NOT_FOUND"
```

### Create container

```bash
IP_LAST=$(python3 -c "import hashlib; h=int(hashlib.md5('${NAME}'.encode()).hexdigest(),16); print(10+(h%240))")
IP="10.10.0.${IP_LAST}"

lxc copy vps-template/clean-vps ${NAME} \
  --config devices.eth0.ipv4.address=${IP}
lxc start ${NAME}

until lxc exec ${NAME} -- systemctl is-system-running --quiet 2>/dev/null; do sleep 1; done
lxc exec ${NAME} -- bash -c "echo 'root:${ROOT_PASS}' | chpasswd"

until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 root@${IP} true 2>/dev/null
do sleep 1; done

echo "READY name=${NAME} ip=${IP} user=root"
```

### Get IP

```bash
lxc list ${NAME} -c 4 --format csv | cut -d' ' -f1
```

### Status

```bash
lxc list ${NAME} --format csv | awk -F',' '{print $2}'
# → RUNNING | STOPPED | FROZEN | (empty = does not exist)
```

### Stop / Start

```bash
lxc stop  ${NAME}
lxc start ${NAME}
```

### Reset to clean state

```bash
lxc stop ${NAME} --force 2>/dev/null || true
lxc restore ${NAME} clean-vps
lxc start ${NAME}

until lxc exec ${NAME} -- systemctl is-system-running --quiet 2>/dev/null; do sleep 1; done
lxc exec ${NAME} -- bash -c "echo 'root:${ROOT_PASS}' | chpasswd"

until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 root@${IP} true 2>/dev/null
do sleep 1; done
echo "RESET_DONE name=${NAME} ip=${IP}"
```

### Delete

```bash
lxc delete ${NAME} --force
```

### List own containers

```bash
lxc list --format csv | awk -F',' '{print $1}' | grep "^${PREFIX}-"
```

### Delete all own containers

```bash
lxc list --format csv | awk -F',' '{print $1}' | grep "^${PREFIX}-" \
  | xargs -r -I{} lxc delete {} --force
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
3. Never run `lxc delete` without explicit full `${NAME}`.
4. Never modify `vps-template` — it is a read-only source for `lxc copy`.
5. Agent does not read or modify containers with a foreign `AGENT_ID`.
6. `ROOT_PASS` is never logged, never written to disk, never appears in container names — passed only at the moment of use.
