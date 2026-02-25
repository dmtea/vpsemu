---
name: vpsemu
version: 0.36
description: >
  Управление Incus-контейнерами которые имитируют VPS-серверы (Ubuntu 24.04).
  Используй этот скилл для создания, проверки, сброса и удаления тестовых
  машин под Ansible / IaC задачи. Поддерживает параллельную работу множества
  агентов через строгий неймспейс vpsemu-{AGENT_ID}-{TASK_NAME}.
  Агент никогда не трогает контейнеры с чужим AGENT_ID.
---

# Incus VPS Skill

## Неймспейс

```
vpsemu-{AGENT_ID}-{TASK_NAME}

Примеры:
  vpsemu-a3f9-deploy-nginx
  vpsemu-b7c1-setup-db
```

- `vpsemu` — статичный префикс, всегда
- `AGENT_ID` — 4-hex, генерируется **один раз при старте сессии**, живёт только в контексте
- `TASK_NAME` — короткое имя текущей задачи, агент назначает сам

---

## ⚡ Инициализация при загрузке скилла (обязательно)

Как только этот скилл загружен в контекст, агент **немедленно** выполняет:

```bash
AGENT_ID=$(openssl rand -hex 2)   # генерируется один раз — например: a3f9
```

Затем агент **запоминает `AGENT_ID` как критический факт сессии** и использует его во всех командах до конца сессии. При сжатии контекста — восстановить из памяти, не генерировать заново.

### Обнаружение форка

При форке сессии в opencode дочерняя сессия наследует весь контекст родителя включая `AGENT_ID`. Два параллельных форка получат одинаковый ID и будут трогать одни и те же контейнеры.

**При инициализации агент обязан проверить владение:**

```bash
# Проверить: есть ли контейнеры с текущим AGENT_ID которые агент НЕ создавал в этой сессии
EXISTING=$(incus list --format csv | awk -F',' '{print $1}' | grep "^vpsemu-${AGENT_ID}-")

if [ -n "$EXISTING" ]; then
  # AGENT_ID унаследован от форкнутой/родительской сессии — регенерируем
  AGENT_ID=$(openssl rand -hex 2)
  echo "FORK DETECTED: new AGENT_ID=${AGENT_ID}"
fi
```

Проверка безопасна: если агент сам создал эти контейнеры ранее в этой же сессии — он знает о них из контекста и узнает их. Если не узнаёт — они принадлежат родительской или соседней сессии → регенерировать.

---

## Bootstrap (выполняется владельцем один раз до первого запуска агента)

### 1. Установка Incus

**Ubuntu 24.04 (noble) — нативный пакет, внешний репозиторий не нужен:**
```bash
sudo apt install incus
```

**Ubuntu 22.04 (jammy) / Pop!_OS 22.04 — через репозиторий Zabbly:**
```bash
# Проверить что fingerprint ключа совпадает: 4EFC 5906 96CB 15B8 7C73 A3AD 82CC 8797 C838 DCFD
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

### 2. Общая настройка (для обеих версий)

```bash
sudo usermod -aG incus-admin $USER && newgrp incus-admin
incus admin init --minimal

# Изолированная сеть — NAT для интернета, DHCP отключён (IP задаются внутри контейнеров)
incus network create incusbr1 \
  ipv4.address=10.10.0.1/24 \
  ipv4.dhcp=false \
  ipv4.nat=true \
  ipv6.address=none

# Базовый профиль — чистый, без nictype/ipv4_filtering (оба конфликтуют со свойством "network:")
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

# Эталонный контейнер — images:ubuntu/24.04 это minimal образ (без cloud-init, быстрый)
incus launch images:ubuntu/24.04 vps-template --profile vps-base

# Ждём готовности сетевого стека (cloud-init отсутствует в minimal образе)
until incus exec vps-template -- true 2>/dev/null; do sleep 1; done
sleep 3

# Задать статичный IP внутри контейнера (Incus ipv4.address только фильтрует, не конфигурирует)
incus exec vps-template -- bash -c "
  ip addr add 10.10.0.2/24 dev eth0
  ip route add default via 10.10.0.1
  echo 'nameserver 8.8.8.8' > /etc/resolv.conf
"

# Проверить сеть перед установкой пакетов
incus exec vps-template -- ping -c 1 8.8.8.8 || { echo 'ERROR: no internet in container'; exit 1; }

# Установить пакеты и настроить SSH
incus exec vps-template -- bash -c "
  apt-get update -qq
  apt-get install -y openssh-server python3 python3-apt -qq
  apt-get clean
  printf "PermitRootLogin yes\nPasswordAuthentication yes\n" \
    > /etc/ssh/sshd_config.d/99-vpsemu.conf
  systemctl restart ssh
"

# Снапшот хранит чистую ОС без пароля и без IP — оба задаются агентом после каждого старта
incus snapshot create vps-template clean-vps
incus stop vps-template
```

> **Важно:** Снапшот НЕ содержит статичного IP — контейнер стартует без IP.
> Агент задаёт IP + пароль сразу после каждого `incus start` или `incus restore`.

---

## Переменные перед каждой командой

```bash
# AGENT_ID берётся из контекста сессии (уже сгенерирован при загрузке скилла)
TASK_NAME="deploy-nginx"               # агент назначает сам по смыслу задачи
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)

NAME="vpsemu-${AGENT_ID}-${TASK_NAME}"
PREFIX="vpsemu-${AGENT_ID}"
```

### Получить ROOT_PASS

Агент выполняет следующую логику **перед созданием контейнера и перед сбросом**.

Стандартный путь к файлу секретов: `ansible/group_vars/all/secrets.enc.yml` (относительно корня проекта).

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
    # файл есть, но ключа нет
    echo "WARN: key '${key}' not found in ${secrets_file}" >&2
  fi

  # файла нет или ключа нет — агент ОСТАНАВЛИВАЕТСЯ и спрашивает пользователя в диалоге
  # затем устанавливает: ROOT_PASS="<введённый пароль>"
  return 1
}

if ! ROOT_PASS=$(_get_root_pass "${PROJECT_ROOT}"); then
  # агент спрашивает пользователя и присваивает ROOT_PASS вручную
  echo "ACTION_REQUIRED: enter root password for container"
fi
```

**Поведение агента:**

| Ситуация | Действие |
|---|---|
| `secrets.enc.yml` найден, ключ `vault_root_password` есть | использовать значение из SOPS |
| `secrets.enc.yml` найден, ключ отсутствует | предупредить пользователя, спросить пароль в диалоге |
| `secrets.enc.yml` не найден | спросить пользователя: указать другой путь или ввести пароль напрямую |
| Пользователь ввёл пароль в диалоге | сохранить в `ROOT_PASS` в контексте, не логировать, не писать на диск |

### IP-аллокация — детерминированная, без коллизий

Одно имя всегда даёт один IP. Параллельные агенты не пересекаются:

```bash
IP_LAST=$(python3 -c "
import hashlib
h = int(hashlib.md5('${NAME}'.encode()).hexdigest(), 16)
print(10 + (h % 240))  # диапазон 10.10.0.10 – 10.10.0.249
")
IP="10.10.0.${IP_LAST}"
```

---

## Команды

### Проверить существование

```bash
incus list --format csv | awk -F',' '{print $1}' | grep -q "^${NAME}$" \
  && echo "EXISTS" || echo "NOT_FOUND"
```

### Создать контейнер

```bash
IP_LAST=$(python3 -c "import hashlib; h=int(hashlib.md5('${NAME}'.encode()).hexdigest(),16); print(10+(h%240))")
IP="10.10.0.${IP_LAST}"

incus copy vps-template/clean-vps ${NAME}
incus start ${NAME}

# Ждём готовности контейнера
until incus exec ${NAME} -- true 2>/dev/null; do sleep 1; done

# Задать статичный IP внутри контейнера
incus exec ${NAME} -- bash -c "
  ip addr add ${IP}/24 dev eth0
  ip route add default via 10.10.0.1
  echo 'nameserver 8.8.8.8' > /etc/resolv.conf
"

# Установить пароль
incus exec ${NAME} -- bash -c "echo 'root:${ROOT_PASS}' | chpasswd"

# Ждём SSH
until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 root@${IP} true 2>/dev/null
do sleep 1; done

echo "READY name=${NAME} ip=${IP} user=root"
```

### Получить IP

```bash
incus list ${NAME} -c 4 --format csv | cut -d' ' -f1
```

### Статус

```bash
incus list ${NAME} --format csv | awk -F',' '{print $2}'
# → RUNNING | STOPPED | FROZEN | (пусто = не существует)
```

### Стоп / Старт

```bash
incus stop  ${NAME}
incus start ${NAME}
```

### Сбросить к чистому состоянию

```bash
incus stop ${NAME} --force 2>/dev/null || true
incus restore ${NAME} clean-vps
incus start ${NAME}

# Ждём готовности контейнера
until incus exec ${NAME} -- true 2>/dev/null; do sleep 1; done

# Перезадать IP и пароль (снапшот не содержит ни IP ни пароля)
incus exec ${NAME} -- bash -c "
  ip addr add ${IP}/24 dev eth0
  ip route add default via 10.10.0.1
  echo 'nameserver 8.8.8.8' > /etc/resolv.conf
"
incus exec ${NAME} -- bash -c "echo 'root:${ROOT_PASS}' | chpasswd"

until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=2 root@${IP} true 2>/dev/null
do sleep 1; done
echo "RESET_DONE name=${NAME} ip=${IP}"
```

### Удалить

```bash
incus delete ${NAME} --force
```

### Список своих контейнеров

```bash
incus list --format csv | awk -F',' '{print $1}' | grep "^${PREFIX}-"
```

### Удалить все свои контейнеры (конец задачи / сессии)

```bash
incus list --format csv | awk -F',' '{print $1}' | grep "^${PREFIX}-" \
  | xargs -r -I{} incus delete {} --force
```

---

## Ansible inventory

`ROOT_PASS` берётся из SOPS (см. выше) и подставляется агентом при генерации inventory.

```ini
; один контейнер
[vps]
${IP} ansible_user=root ansible_password=${ROOT_PASS} ansible_ssh_common_args='-o StrictHostKeyChecking=no'

; несколько контейнеров одной задачи
[web]
${IP_WEB} ansible_user=root ansible_password=${ROOT_PASS} ansible_ssh_common_args='-o StrictHostKeyChecking=no'
[db]
${IP_DB}  ansible_user=root ansible_password=${ROOT_PASS} ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Если пароль получен через диалог (fallback) — передавать только через переменную окружения, не писать в файл:

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory \
  -e "ansible_password=${ROOT_PASS}" playbook.yml
```

---

## Правила безопасности (строго обязательны)

1. `AGENT_ID` генерируется **один раз при старте сессии** — никогда не переназначается и не пишется на диск.
2. Перед любой мутацией (`stop`, `delete`, `restore`) проверить что `${NAME}` начинается с `vpsemu-${AGENT_ID}-`.
3. Никогда не использовать `incus delete` без полного явного `${NAME}`.
4. Никогда не трогать `vps-template` — только источник для `incus copy`.
5. Агент не читает и не изменяет контейнеры с чужим `AGENT_ID`.
6. `ROOT_PASS` никогда не логируется, не пишется в файлы на диске, не появляется в именах контейнеров — передаётся только в момент использования.

---

## Bootstrap cleanup (только для владельца — полный сброс состояния Incus)

```bash
#!/bin/bash
# Запускать если нужно полностью переделать bootstrap с нуля

set -e

# Остановить и удалить все vpsemu контейнеры
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

echo "Очистка завершена. Осталось контейнеров: $(incus list --format csv | wc -l)"
```
