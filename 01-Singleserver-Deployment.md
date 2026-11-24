# Carbonio Single-Node Deployment Guide

### Ubuntu Server & Red Hat Enterprise Linux — Ansible-based installation

| | |
|---|---|
| **Prepared by** | Aazib Ali |
| **Version / Date** | 1.0 — 13 Nov 2026 |
| **Tested on** | Ubuntu Server 24.04 LTS, Ansible core 2.16.3, `zxbot.carbonio_install` 26.9.0 |

> **Scope.** One server runs every Carbonio component. This guide covers the complete path: server checks → OS preparation → Ansible → inventory → SSH → install → validation → troubleshooting. Theory is kept to one or two lines per step; every step has a command and an expected output.

---

## 0. How to Read This Guide

### 0.1 Command and output markers

Every command block is wrapped in start/end markers so you always know where a command begins and ends. Copy only the lines **between** the markers.

```bash
# ───────── COMMAND START ─────────
echo "this is a command"
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
this is a command
```

### 0.2 Tags used in headings

| Tag | Meaning |
|---|---|
| 🟦 **TARGET** | Run on the Carbonio server being built |
| 🟩 **CONTROL** | Run on the Ansible control node (can be the same machine as the target) |
| 🟠 **UBUNTU** | Ubuntu-only step |
| 🔴 **RHEL** | RHEL-only step |
| ⚪ **BOTH** | Same on Ubuntu and RHEL |

### 0.3 Placeholders

Replace every placeholder before running a command. **Never paste a command that still contains `UPPER_CASE` placeholders.**

| Placeholder | Meaning | Example format |
|---|---|---|
| `SERVER_FQDN` | Fully qualified hostname of the Carbonio server | `mail.example.com` |
| `SHORT_HOSTNAME` | Short hostname | `mail` |
| `SERVER_IP` | Static IP of the server | `192.0.2.10` |
| `PREFIX` | Subnet prefix length | `24` |
| `GATEWAY_IP` | Network gateway | `192.0.2.1` |
| `DNS_SERVER_IP` | Approved DNS server | `192.0.2.53` |
| `NETWORK_INTERFACE` | NIC name (`ip addr show`) | `ens160` |
| `TIME_ZONE` | Approved time zone | `Asia/Karachi` |
| `MAIL_DOMAIN` | Default mail domain | `example.com` |
| `WEBMAIL_FQDN` | Hostname users open in the browser | `mail.example.com` |
| `ANSIBLE_USER` | Deployment account (sudo-enabled) | `deploy` |

### 0.4 Workflow at a glance

| Phase | Where | Goal | Gate to pass |
|---|---|---|---|
| 1 | TARGET | Hardware verified | Meets minimum specs |
| 2 | TARGET | OS prepared | Pre-flight script all `PASS` |
| 3 | CONTROL | Ansible + collections ready | Collections listed |
| 4 | CONTROL | Inventory written | `ansible-inventory --graph` correct |
| 5 | CONTROL | SSH + sudo working | `ping` = `pong`, `id -u` = `0` |
| 6 | TARGET | Final gate + snapshot | Approval recorded |
| 7 | CONTROL | Install | `failed=0` |
| 8 | TARGET | Validate | No failed units |

---

## 1. What a Single-Node Deployment Is

All Carbonio roles (PostgreSQL, LDAP, Consul, Postfix/ClamAV, NGINX, Mailbox) run on **one** server and share its CPU, RAM and disk. It can only be installed with Ansible, never role-by-role.

| Included | Optional on the same node | Not supported on a single node |
|---|---|---|
| Mail, Calendar, Contacts, Backup | Files, Preview | Tasks, Docs, Chats, Video Server (need separate nodes), Replica Directory, Syslog server, Monitoring/Prometheus |

> **⚠ Decision gate — architecture.** The single-server installer applies tuning that **makes later scale-out to multi-node impossible**. Get written approval from the responsible technical authority *before* Phase 7.

---

## 2. Phase 1 — Hardware Verification &nbsp; 🟦 TARGET ⚪ BOTH

| Resource | Minimum (official single-node) | Ace Micro production recommendation |
|---|---|---|
| CPU | 8 vCPU | 16 vCPU |
| RAM | 24 GB | 24 GB or more |
| OS disk | 100 GB | 100 GB or more |
| Mailbox / backup storage | Extra, sized per users | Dedicated volumes (`/opt/zextras`, `/opt/zextras/backup`) |
| Architecture | x86_64 | x86_64 |

> Lab VMs below the minimum (e.g. 12 GB RAM) can install but are prone to slow starts and service timeouts. Never use them for production.

```bash
# ───────── COMMAND START ─────────
nproc
free -h
lsblk
df -h /
uname -m
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
8                       <- nproc: 8 or more
Mem:   23Gi ...         <- free -h: total about 23-24Gi on a 24 GB VM
sda 100G ...            <- lsblk: disk 100G or more
/dev/mapper/...  98G    <- df -h /: root filesystem about 100G
x86_64                  <- uname -m
```

---

## 3. Phase 2 — Server Preparation &nbsp; 🟦 TARGET

### 3.1 Operating system &nbsp; ⚪ BOTH

Supported: **Ubuntu Server 22.04 / 24.04 LTS** (choose *Ubuntu Server*, not *minimized*) and **RHEL 8 / 9**. Use a clean install.

```bash
# ───────── COMMAND START ─────────
cat /etc/os-release
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
PRETTY_NAME="Ubuntu 24.04.x LTS"      (or "Red Hat Enterprise Linux 9.x")
```

### 3.2 Network — fixed IP &nbsp; ⚪ BOTH

The IP must never change after reboot. Use a static IP **or** a DHCP reservation. Check the current state first:

```bash
# ───────── COMMAND START ─────────
ip addr show
ip route
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
2: NETWORK_INTERFACE: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
    inet SERVER_IP/PREFIX ...
default via GATEWAY_IP dev NETWORK_INTERFACE ...
```

> **⚠ Do network changes from the console, not over SSH.** A wrong value locks you out.

**🟠 UBUNTU — Netplan** (edit the file in `/etc/netplan/`):

```bash
# ───────── COMMAND START ─────────
sudo nano /etc/netplan/50-cloud-init.yaml
# ───────── COMMAND END ───────────
```

```yaml
network:
  version: 2
  ethernets:
    NETWORK_INTERFACE:
      dhcp4: false
      addresses: [SERVER_IP/PREFIX]
      routes:
        - to: default
          via: GATEWAY_IP
      nameservers:
        addresses: [DNS_SERVER_IP]
```

```bash
# ───────── COMMAND START ─────────
sudo netplan try
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Do you want to keep these settings?
Press ENTER before the timeout to accept the new configuration
→ press ENTER
```

**🔴 RHEL — NetworkManager:**

```bash
# ───────── COMMAND START ─────────
nmcli con show
sudo nmcli con mod "CONNECTION_NAME" ipv4.method manual \
  ipv4.addresses SERVER_IP/PREFIX ipv4.gateway GATEWAY_IP ipv4.dns DNS_SERVER_IP
sudo nmcli con up "CONNECTION_NAME"
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Connection successfully activated (D-Bus active path: ...)
```

### 3.3 Hostname and FQDN &nbsp; ⚪ BOTH

```bash
# ───────── COMMAND START ─────────
sudo hostnamectl set-hostname SERVER_FQDN
hostname
hostname -f
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
SERVER_FQDN
SERVER_FQDN
```

### 3.4 `/etc/hosts` — the #1 cause of failed installs &nbsp; ⚪ BOTH

Carbonio aborts if `localhost` appears more than once (typically on the IPv6 `::1` line), or if the FQDN maps to a loopback address. Replace the whole file with the template below.

```bash
# ───────── COMMAND START ─────────
sudo cp /etc/hosts /etc/hosts.bak
sudo nano /etc/hosts
# ───────── COMMAND END ───────────
```

```text
── FILE CONTENT (/etc/hosts) ──
127.0.0.1   localhost

SERVER_IP   SERVER_FQDN   SHORT_HOSTNAME

# IPv6 — note: NO "localhost" on the ::1 line
::1         ip6-localhost ip6-loopback
fe00::0     ip6-localnet
ff00::0     ip6-mcastprefix
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters
```

> Remove any `127.0.1.1 <hostname>` line (Ubuntu adds one by default) so the server name only maps to `SERVER_IP`.

```bash
# ───────── COMMAND START ─────────
cat -n /etc/hosts
getent hosts SERVER_FQDN
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
SERVER_IP SERVER_FQDN SHORT_HOSTNAME
```

If the installer later fails with `Multiple localhost definitions found in /etc/hosts`, see [Troubleshooting §10](#10-troubleshooting-matrix).

### 3.5 DNS &nbsp; ⚪ BOTH

The FQDN must resolve to `SERVER_IP`. For production, the DNS administrator must also publish these records for `MAIL_DOMAIN`:

| Record | Purpose |
|---|---|
| A | `SERVER_FQDN` → `SERVER_IP` |
| MX | Mail server for `MAIL_DOMAIN` |
| PTR | Reverse lookup `SERVER_IP` → `SERVER_FQDN` |
| TXT | SPF / DKIM / DMARC |

```bash
# ───────── COMMAND START ─────────
cat /etc/resolv.conf
# 🟠 UBUNTU:  sudo apt install -y dnsutils
# 🔴 RHEL:    sudo dnf install -y bind-utils
dig +short A SERVER_FQDN
dig +short MX MAIL_DOMAIN
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
SERVER_IP                          <- A record
10 SERVER_FQDN.                    <- MX record (production only)
```

> Lab without DNS: the `/etc/hosts` entry from §3.4 is enough for the FQDN; MX/PTR checks apply to production only.

### 3.6 Time zone, NTP and locale

**🟠 UBUNTU**

```bash
# ───────── COMMAND START ─────────
sudo timedatectl set-timezone TIME_ZONE
sudo timedatectl set-ntp true
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8
# ───────── COMMAND END ───────────
```

**🔴 RHEL**

```bash
# ───────── COMMAND START ─────────
sudo timedatectl set-timezone TIME_ZONE
sudo dnf install -y chrony glibc-langpack-en
sudo systemctl enable --now chronyd
sudo localectl set-locale LANG=en_US.UTF-8
chronyc tracking
# ───────── COMMAND END ───────────
```

**Verify (both)** — log out and back in so the locale applies:

```bash
# ───────── COMMAND START ─────────
timedatectl
locale | head -n 3
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
       Time zone: TIME_ZONE (...)
System clock synchronized: yes
              NTP service: active
LANG=en_US.UTF-8
LANGUAGE=
LC_CTYPE="en_US.UTF-8"
```

### 3.7 Required packages &nbsp; (includes the `acl` package)

Python 3 and Perl are required. **`acl` (provides `setfacl`) is also mandatory** when Ansible connects as a non-root user — without it the install fails at its first privileged task.

**🟠 UBUNTU**

```bash
# ───────── COMMAND START ─────────
sudo apt update
sudo apt install -y python3 perl acl openssh-server
sudo systemctl enable --now ssh
# ───────── COMMAND END ───────────
```

**🔴 RHEL**

```bash
# ───────── COMMAND START ─────────
sudo dnf install -y python3 perl acl openssh-server
sudo systemctl enable --now sshd
# ───────── COMMAND END ───────────
```

**Verify (both):**

```bash
# ───────── COMMAND START ─────────
python3 --version
perl -v | head -n 2
command -v setfacl || echo "setfacl is missing"
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Python 3.x.x
This is perl 5, version NN ...
/usr/bin/setfacl
```

### 3.8 RHEL-only: repositories and SELinux &nbsp; 🔴 RHEL

RHEL needs extra repositories and SELinux set to **permissive or disabled**.

**RHEL 9**

```bash
# ───────── COMMAND START ─────────
sudo subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms
sudo subscription-manager repos --enable=codeready-builder-for-rhel-9-x86_64-rpms
sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
# ───────── COMMAND END ───────────
```

**RHEL 8**

```bash
# ───────── COMMAND START ─────────
sudo subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms
sudo subscription-manager repos --enable=rhel-8-for-x86_64-appstream-rpms
sudo subscription-manager repos --enable=codeready-builder-for-rhel-8-x86_64-rpms
sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
# ───────── COMMAND END ───────────
```

**SELinux (RHEL 8 and 9)**

```bash
# ───────── COMMAND START ─────────
sudo sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config
sudo setenforce 0
sestatus
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Current mode:                   permissive
Mode from config file:          permissive
```

### 3.9 Conflicting services &nbsp; ⚪ BOTH

Carbonio brings its own mail, database and web stack. On a clean server nothing else should own those ports.

```bash
# ───────── COMMAND START ─────────
sudo ss -tulpn
# 🟠 UBUNTU:
dpkg -l | grep -Ei 'postfix|exim|mysql|mariadb|apache2|nginx|webmin|cockpit'
# 🔴 RHEL:
rpm -qa | grep -Ei 'postfix|exim|mysql|mariadb|httpd|nginx|webmin|cockpit'
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Only SSH (22) and the DNS stub (127.0.0.53/54:53) listening.
The package search prints nothing.
```

> Do not remove existing packages until their purpose is reviewed and approved.

### 3.10 Firewall &nbsp; ⚪ BOTH

Do not leave the firewall disabled in production. The single node needs **no internal ports**, only the external ones. Confirm the exact list for your Carbonio version in the official *External Connections* page; the usual set is below.

| Port(s) | Service |
|---|---|
| 22 | SSH (administration) |
| 25, 465, 587 | SMTP (MTA) |
| 80, 443 | Web / Proxy |
| 110, 995 / 143, 993 | POP3(S) / IMAP(S) via Proxy |

**🟠 UBUNTU — UFW** (allow SSH first, always)

```bash
# ───────── COMMAND START ─────────
sudo ufw allow 22/tcp
sudo ufw allow 25/tcp
sudo ufw allow 465/tcp
sudo ufw allow 587/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 110/tcp
sudo ufw allow 995/tcp
sudo ufw allow 143/tcp
sudo ufw allow 993/tcp
sudo ufw enable
sudo ufw status verbose
# ───────── COMMAND END ───────────
```

**🔴 RHEL — firewalld**

```bash
# ───────── COMMAND START ─────────
sudo systemctl enable --now firewalld
for p in 22 25 465 587 80 443 110 995 143 993; do sudo firewall-cmd --permanent --add-port=${p}/tcp; done
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Status: active                      (ufw)
22/tcp 25/tcp 465/tcp 587/tcp 80/tcp 443/tcp 110/tcp 995/tcp 143/tcp 993/tcp   (firewalld)
```

> Lab VMs may keep the firewall inactive (`Status: inactive`); record that decision in the deployment record.

### 3.11 The `zextras` user must not exist &nbsp; ⚪ BOTH

The installer creates this account. Never create it manually.

```bash
# ───────── COMMAND START ─────────
id zextras
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
id: 'zextras': no such user
```

> After a **failed or partial** install this user will exist. That is normal — do not delete it; simply re-run the playbook (§9.4).

---

## 4. Pre-Flight Check Script &nbsp; 🟦 TARGET ⚪ BOTH

One script that checks everything from Phase 1–2. Save it, run it, and do not continue until every line is `PASS`.

```bash
# ───────── COMMAND START ─────────
nano ~/carbonio-preflight.sh
# ───────── COMMAND END ───────────
```

```bash
#!/usr/bin/env bash
# carbonio-preflight.sh — run on the TARGET node
# Usage: bash carbonio-preflight.sh SERVER_FQDN
FQDN="${1:?Usage: bash carbonio-preflight.sh <SERVER_FQDN>}"
R() { if [ "$2" -eq 0 ]; then echo "[PASS] $1"; else echo "[FAIL] $1"; fi; }

cpu=$(nproc)
ram=$(awk '/MemTotal/ {printf "%d", $2/1024/1024}' /proc/meminfo)
disk=$(df -BG --output=size / | tail -n1 | tr -dc '0-9')
ip=$(getent ahostsv4 "$FQDN" | awk 'NR==1 {print $1}')

[ "$(uname -m)" = "x86_64" ];            R "Architecture is x86_64" $?
[ "$cpu" -ge 8 ];                        R "CPU >= 8 vCPU (found $cpu)" $?
[ "$ram" -ge 23 ];                       R "RAM >= 24 GB (found ~${ram} GiB)" $?
[ "$disk" -ge 90 ];                      R "Root filesystem ~100 GB (found ${disk} GB)" $?
[[ -n "$ip" && "$ip" != 127.* ]];        R "FQDN resolves to non-loopback IPv4 (got: ${ip:-none})" $?
[ "$(hostname -f 2>/dev/null)" = "$FQDN" ]; R "hostname -f returns $FQDN" $?
[ "$(grep -vE '^[[:space:]]*#' /etc/hosts | grep -Ec '(^|[[:space:]])localhost([[:space:]]|$)')" -eq 1 ]
                                         R "/etc/hosts has exactly one 'localhost' entry" $?
[ "$(timedatectl show -p NTPSynchronized --value)" = "yes" ]; R "Clock is NTP-synchronised" $?
locale | grep -q '^LANG=en_US.UTF-8';    R "Locale is en_US.UTF-8" $?
command -v python3 >/dev/null;           R "python3 installed" $?
command -v perl >/dev/null;              R "perl installed" $?
command -v setfacl >/dev/null;           R "acl installed (setfacl present)" $?
! id zextras >/dev/null 2>&1;            R "user 'zextras' absent (fresh install only)" $?
! ss -tln | grep -Eq ':(25|80|443|110|143|465|587|993|995)[[:space:]]'
                                         R "Carbonio ports are free (fresh install only)" $?
if command -v dpkg >/dev/null 2>&1; then pk=$(dpkg -l | awk '/^ii/ {print $2}'); else pk=$(rpm -qa); fi
! echo "$pk" | grep -Eq '^(postfix|exim|mysql|mariadb|apache2|httpd|nginx|webmin|cockpit)'
                                         R "No conflicting mail/db/web packages" $?
if command -v getenforce >/dev/null 2>&1; then
  [ "$(getenforce)" != "Enforcing" ];    R "SELinux is not Enforcing" $?
fi
```

```bash
# ───────── COMMAND START ─────────
bash ~/carbonio-preflight.sh SERVER_FQDN
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
[PASS] Architecture is x86_64
[PASS] CPU >= 8 vCPU (found 8)
[PASS] RAM >= 24 GB (found ~23 GiB)
[PASS] Root filesystem ~100 GB (found 98 GB)
[PASS] FQDN resolves to non-loopback IPv4 (got: SERVER_IP)
[PASS] hostname -f returns SERVER_FQDN
[PASS] /etc/hosts has exactly one 'localhost' entry
[PASS] Clock is NTP-synchronised
[PASS] Locale is en_US.UTF-8
[PASS] python3 installed
[PASS] perl installed
[PASS] acl installed (setfacl present)
[PASS] user 'zextras' absent (fresh install only)
[PASS] Carbonio ports are free (fresh install only)
[PASS] No conflicting mail/db/web packages
[PASS] SELinux is not Enforcing          <- RHEL only
```

> **🛑 Gate:** any `[FAIL]` → fix it first. Do not start Phase 3.

---

## 5. Phase 3 — Ansible Control Node &nbsp; 🟩 CONTROL

The control node runs Ansible and SSHes into the target. It can be a separate workstation or the target server itself. No agent is needed on the target.

### 5.1 Install Ansible

**🟠 UBUNTU**

```bash
# ───────── COMMAND START ─────────
sudo apt update
sudo apt install -y ansible
ansible --version
# ───────── COMMAND END ───────────
```

**🔴 RHEL / other Linux**

```bash
# ───────── COMMAND START ─────────
sudo dnf install -y python3-pip
pip3 install --user ansible
echo 'export PATH=$PATH:$HOME/.local/bin' >> ~/.bashrc && source ~/.bashrc
ansible --version
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
ansible [core 2.16.3]          <- version may be newer
  config file = None           <- becomes the project's ansible.cfg once Phase 3.3 is done
  python version = 3.12.x
```

### 5.2 Install the Carbonio collections

```bash
# ───────── COMMAND START ─────────
ansible-galaxy collection install zxbot.carbonio_install
ansible-galaxy collection install zxbot.carbonio_ssinstall
ansible-galaxy collection list | grep -E 'zxbot|ansible.utils|community.postgresql'
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
zxbot.carbonio_install        26.9.0
zxbot.carbonio_ssinstall      26.3.1
zxbot.sps_ansible_addons      1.0.1
ansible.utils                 6.1.1     <- installed automatically as dependencies
community.postgresql          5.1.1
```

| Collection | Purpose |
|---|---|
| `zxbot.carbonio_install` | Core roles (all Carbonio roles) |
| `zxbot.carbonio_ssinstall` | Single-server playbook and tuning |
| `zxbot.sps_ansible_addons` | Support collection (dependency) |

> Versions above are the ones tested at Ace Micro. Record the versions you install; reuse the same versions for repeat deployments. If `ansible.utils` or `community.postgresql` are missing or old, install them explicitly:
> `ansible-galaxy collection install ansible.utils community.postgresql`

### 5.3 Create the deployment directory

All generated files (passwords, logs) land in the directory you run the playbook from — keep one directory per environment.

```bash
# ───────── COMMAND START ─────────
mkdir -p ~/carbonio-deployment/credentials
chmod 700 ~/carbonio-deployment ~/carbonio-deployment/credentials
cd ~/carbonio-deployment
nano ansible.cfg
# ───────── COMMAND END ───────────
```

```ini
── FILE CONTENT (ansible.cfg) ──
[defaults]
log_path=/home/ANSIBLE_USER/carbonio-deployment/carbonio-ansible.log
```

> **Use a path inside your home directory.** The commonly documented `/var/log/carbonio-ansible.log` is not writable for a normal user and Ansible prints `log file ... is not writeable`.

```bash
# ───────── COMMAND START ─────────
cd ~/carbonio-deployment
ansible --version | grep 'config file'
ls -la
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
  config file = /home/ANSIBLE_USER/carbonio-deployment/ansible.cfg
drwx------ ... credentials
-rw-rw-r-- ... ansible.cfg
```

> Ansible only reads `ansible.cfg` when you run it **from inside this directory**. Always `cd ~/carbonio-deployment` first.

---

## 6. Phase 4 — Inventory &nbsp; 🟩 CONTROL

The inventory tells Ansible which server runs which Carbonio role. On a single node every populated group points to the same `SERVER_FQDN`.

```bash
# ───────── COMMAND START ─────────
cd ~/carbonio-deployment
nano carbonio-inventory-single
# ───────── COMMAND END ───────────
```

```ini
── FILE CONTENT (carbonio-inventory-single) ──
[postgresServers]
SERVER_FQDN

[masterDirectoryServers]
SERVER_FQDN

[masterDirectoryServers:vars]
default_domain=MAIL_DOMAIN

[serviceDiscoverServers]
SERVER_FQDN

[mtaServers]
SERVER_FQDN

[proxyServers]
SERVER_FQDN

[proxyServers:vars]
webmailHostname=WEBMAIL_FQDN

[applicationServers]
SERVER_FQDN

############ Optional — same node ############
[previewServers]
SERVER_FQDN

[filesServers]
SERVER_FQDN

############ Optional — need a DIFFERENT node, keep EMPTY here ############
[taskServers]
[docsServers]
[workStreamServers]
[videoServers]

############ Not supported on single node — keep present and EMPTY ############
[dbsConnectorServers]
[replicaDirectoryServers]
[prometheusServers]
[syslogServer]
```

### Rules

| Rule | Why |
|---|---|
| `default_domain` stays under `[masterDirectoryServers:vars]` | Outside `:vars`, Ansible treats it as a host name |
| `webmailHostname` stays under `[proxyServers:vars]` | Same reason |
| The four "not supported" groups must exist, empty | The playbook expects them |
| Skip `previewServers` / `filesServers` if not needed | Optional on single node |
| Check the inventory against the exact collection version | The expected layout can change between releases |

```bash
# ───────── COMMAND START ─────────
ansible-inventory -i carbonio-inventory-single --graph
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
@all:
  |--@ungrouped:
  |--@postgresServers:
  |  |--SERVER_FQDN
  |--@masterDirectoryServers:
  |  |--SERVER_FQDN
  |--@serviceDiscoverServers:
  |  |--SERVER_FQDN
  |--@mtaServers:
  |  |--SERVER_FQDN
  |--@proxyServers:
  |  |--SERVER_FQDN
  |--@applicationServers:
  |  |--SERVER_FQDN
  |--@previewServers:
  |  |--SERVER_FQDN
  |--@filesServers:
  |  |--SERVER_FQDN
  |--@taskServers:
  |--@docsServers:
  |--@workStreamServers:
  |--@videoServers:
  |--@dbsConnectorServers:
  |--@replicaDirectoryServers:
  |--@prometheusServers:
  |--@syslogServer:
```

> No `WARNING` lines should appear. A `log file ... not writeable` warning means `ansible.cfg` is wrong (see §5.3).

---

## 7. Phase 5 — SSH and Privilege Escalation &nbsp; 🟩 CONTROL → 🟦 TARGET

Ansible logs in over SSH with a key. The official Zextras method uses `root`; Ace Micro's standard is a **dedicated sudo-enabled account** (`ANSIBLE_USER`) with `--become -K` (tested in the Ace Micro lab). Do not enable unrestricted root SSH just to make a command work.

### 7.1 Confirm the account has sudo &nbsp; 🟦 TARGET

```bash
# ───────── COMMAND START ─────────
sudo -l
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
User ANSIBLE_USER may run the following commands on SHORT_HOSTNAME:
    (ALL : ALL) ALL
```

### 7.2 Name resolution from the control node &nbsp; 🟩 CONTROL

```bash
# ───────── COMMAND START ─────────
getent hosts SERVER_FQDN
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
SERVER_IP SERVER_FQDN SHORT_HOSTNAME
```

### 7.3 Generate and install the SSH key &nbsp; 🟩 CONTROL

```bash
# ───────── COMMAND START ─────────
ls -l ~/.ssh/id_ed25519* 2>/dev/null || ssh-keygen -t ed25519
ssh-copy-id ANSIBLE_USER@SERVER_FQDN
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Your identification has been saved in /home/ANSIBLE_USER/.ssh/id_ed25519
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh 'ANSIBLE_USER@SERVER_FQDN'"
```

> Keep the private key on the control node only. Set a passphrase according to company policy.

### 7.4 Test SSH &nbsp; 🟩 CONTROL

```bash
# ───────── COMMAND START ─────────
ssh ANSIBLE_USER@SERVER_FQDN 'hostname -f && whoami'
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
SERVER_FQDN
ANSIBLE_USER
```

No password prompt should appear (passphrase excepted).

### 7.5 Test Ansible connectivity and sudo &nbsp; 🟩 CONTROL

```bash
# ───────── COMMAND START ─────────
cd ~/carbonio-deployment
ansible all -i carbonio-inventory-single -u ANSIBLE_USER -m ping
ansible all -i carbonio-inventory-single -u ANSIBLE_USER --become -K -m command -a 'id -u'
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
SERVER_FQDN | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
BECOME password:
SERVER_FQDN | CHANGED | rc=0 >>
0
```

> **Gate:** `pong` **and** `0`. The `0` proves sudo escalation works. Without `-K` you will see `Missing sudo password`.

---

## 8. Phase 6 — Final Gate before Install &nbsp; 🟦 TARGET

| # | Check | Done |
|---|---|---|
| 1 | Pre-flight script (§4) — all `PASS` | ☐ |
| 2 | Inventory graph correct (§6) | ☐ |
| 3 | Ansible `pong` and `id -u` = `0` (§7.5) | ☐ |
| 4 | Single-node architecture **approved in writing** (§1) | ☐ |
| 5 | **VM snapshot taken** (there is no uninstaller — a snapshot is the rollback) | ☐ |
| 6 | Minimum 24 GB RAM confirmed (or lab exception recorded) | ☐ |

---

## 9. Phase 7 — Installation &nbsp; 🟩 CONTROL

### 9.1 Run the single-server playbook

Always run from `~/carbonio-deployment`. Use the **single-server** playbook (`carbonio_ssinstall`) and pass the core collection as `install_path`.

**Standard (Ace Micro) — sudo account**

```bash
# ───────── COMMAND START ─────────
cd ~/carbonio-deployment
ansible-playbook zxbot.carbonio_ssinstall.carbonio_ssinstall \
  -u ANSIBLE_USER -i carbonio-inventory-single \
  --become -K \
  --extra-vars "install_path=zxbot.carbonio_install.carbonio_install"
# ───────── COMMAND END ───────────
```

**Official Zextras form — root SSH access**

```bash
# ───────── COMMAND START ─────────
cd ~/carbonio-deployment
ansible-playbook zxbot.carbonio_ssinstall.carbonio_ssinstall \
  -u root -i carbonio-inventory-single \
  --extra-vars "install_path=zxbot.carbonio_install.carbonio_install"
# ───────── COMMAND END ───────────
```

> Running `zxbot.carbonio_install.carbonio_install` **directly** is the command for multi-node scenarios; for a single node it skips the single-server tuning stage. The playbook name is an *argument to* `ansible-playbook`, not a command on its own.

> **⚠ Optimization prompt.** If the installer asks to apply the single-server optimization, accept **only** with the approval from §8 — it cannot be undone and blocks future multi-node scale-out.

### 9.2 Watch progress &nbsp; (second terminal)

```bash
# ───────── COMMAND START ─────────
tail -f ~/carbonio-deployment/carbonio-ansible.log
# ───────── COMMAND END ───────────
```

### 9.3 Expected output

```text
── EXPECTED OUTPUT (shape) ──
BECOME password:
PLAY [...] ************************************************
TASK [Gathering Facts] ************************************
ok: [SERVER_FQDN]
...
PLAY [Bootstrap primary cluster services] *****************
...
RUNNING HANDLER [... : restart_postgres] ******************
changed: [SERVER_FQDN]
...
PLAY RECAP ************************************************
SERVER_FQDN : ok=NN  changed=NN  unreachable=0  failed=0  skipped=NN  rescued=0  ignored=0
localhost   : ok=2   changed=0   unreachable=0  failed=0  skipped=7   rescued=0  ignored=0
```

**Success criteria:** `failed=0` and `unreachable=0` for every host.

### 9.4 If the playbook fails — re-run, don't repair by hand

Ansible tracks completed tasks. After fixing the cause, **run the same command again**; it resumes safely.

| Step | Action |
|---|---|
| 1 | Read the failing task: `grep -n -B 15 -A 25 -E 'FAILED!\|fatal:\|ERROR!' ~/carbonio-deployment/carbonio-ansible.log \| tail -n 100` |
| 2 | Match the message in [§10](#10-troubleshooting-matrix) and apply the fix |
| 3 | Re-run the identical `ansible-playbook` command |
| 4 | Never delete the `zextras` user or generated files between runs |

### 9.5 Generated files — protect them

The installer writes the generated service passwords next to the inventory file. **Do not delete or regenerate them**; they are needed for upgrades.

```bash
# ───────── COMMAND START ─────────
cd ~/carbonio-deployment
ls -l carbonio-inventory-single*
chmod 600 carbonio-inventory-single_*
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
-rw-rw-r-- ... carbonio-inventory-single
-rw------- ... carbonio-inventory-single_consulpassword
-rw------- ... carbonio-inventory-single_ldap_password
-rw------- ... carbonio-inventory-single_postgrespassword
```

> Store these according to Ace Micro's credential policy; never e-mail or paste them in chat.

---

## 10. Troubleshooting Matrix

Real failures seen during Ace Micro lab deployments, with exact messages.

| # | Exact message | Cause | Fix | Prevent |
|---|---|---|---|---|
| 1 | `Missing sudo password` | `--become` used without a password | Add `-K` (`--ask-become-pass`) | Always use `--become -K` |
| 2 | `chmod: invalid mode: 'A+user:zextras:rx:allow'` — *Failed to set permissions on the temporary files…* | `acl` package missing on the target | `sudo apt install -y acl` / `sudo dnf install -y acl`, re-run | §3.7 |
| 3 | `Multiple localhost definitions found in /etc/hosts (2). Please ensure only '127.0.0.1 localhost' remains and remove localhost from IPv6.` | `::1` line contains `localhost` | Change to `::1 ip6-localhost ip6-loopback`, re-run | §3.4 |
| 4 | `Incorrect su password` (at a task that switches to the unprivileged user) | Wrong or missing become (sudo) password on that run | Re-run with `--become -K` and the correct sudo password; confirm `sudo -l` shows `(ALL : ALL) ALL` | §7.5 |
| 5 | `Failed to restart postgresql.service: Connection timed out` | Transient systemd timeout, often on a low-RAM VM | Steps in §10.1 below, then re-run | 24 GB RAM, §2 |
| 6 | `log file at /var/log/carbonio-ansible.log is not writeable` | Log path not writable by the user | Set `log_path` inside the home directory | §5.3 |
| 7 | `ansible-playbook: error: the following arguments are required: playbook` | Playbook name missing from the command | Add `zxbot.carbonio_ssinstall.carbonio_ssinstall` | §9.1 |
| 8 | `zxbot.carbonio_install.carbonio_install: command not found` | Playbook name typed as a shell command | Put it after `ansible-playbook` | §9.1 |
| 9 | `ls: cannot access '/home/…/.ssh/id_ed25519*'` | No SSH key yet | `ssh-keygen -t ed25519`, then `ssh-copy-id` | §7.3 |
| 10 | `Connection refused` on SSH | `ssh` service disabled or firewall | 🟠 `sudo systemctl enable --now ssh` / 🔴 `sudo systemctl enable --now sshd`; allow port 22 | §3.7, §3.10 |
| 11 | Collection or dependency version errors (`ansible.utils`, `community.postgresql`) | Old system-wide collections picked up | `ansible-galaxy collection install ansible.utils community.postgresql` | §5.2 |
| 12 | Inventory host treated as a variable / wrong group | `default_domain` / `webmailHostname` outside `:vars` | Move into the `:vars` sections | §6 |

### 10.1 PostgreSQL restart timeout — step by step &nbsp; 🟦 TARGET

```bash
# ───────── COMMAND START ─────────
systemctl status postgresql --no-pager -l
free -h && df -h /
sudo systemctl reset-failed postgresql
sudo systemctl restart postgresql
sudo -u postgres psql -c "select version();"
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
Active: active (exited)            <- Ubuntu wrapper unit; this is normal
PostgreSQL 16.x on x86_64-pc-linux-gnu ...
(1 row)
```

> On Ubuntu, `postgresql.service` is only a wrapper and shows `active (exited)`. The server itself is the per-cluster unit; `psql` returning a version proves it works. Then re-run the playbook (§9.4).
> If it repeats, collect: `journalctl -u 'postgresql@*' -n 50 --no-pager` and `journalctl -b -p err -n 30 --no-pager`.

---

## 11. Phase 8 — Service Validation &nbsp; 🟦 TARGET

### 11.1 Services and failed units

```bash
# ───────── COMMAND START ─────────
systemctl status 'carbonio-*' --no-pager | grep -E '●|Active:'
systemctl --failed
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
● carbonio-....service - ...
     Active: active (running) ...
  UNIT LOAD ACTIVE SUB DESCRIPTION
0 loaded units listed.
```

### 11.2 Installed packages

```bash
# ───────── COMMAND START ─────────
# 🟠 UBUNTU:
dpkg -l | grep carbonio
# 🔴 RHEL:
rpm -qa | grep carbonio
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
ii  carbonio-...        <- Ubuntu: one line per package, status "ii"
carbonio-...x86_64      <- RHEL: one line per package
```

### 11.3 Ports and web access

```bash
# ───────── COMMAND START ─────────
sudo ss -tlnp | grep -E ':(25|80|443|465|587|993|995)[[:space:]]'
curl -kIs https://WEBMAIL_FQDN | head -n 1
# ───────── COMMAND END ───────────
```

```text
── EXPECTED OUTPUT ──
LISTEN ... :25 / :80 / :443 ...      <- mail and proxy ports listening
HTTP/2 200                           <- or an HTTP 3xx redirect
```

### 11.4 Record the result

| Item | Expected | Actual |
|---|---|---|
| Carbonio services | Running | `[RECORD]` |
| Failed units | `0 loaded units listed` | `[RECORD]` |
| Packages | Present | `[RECORD]` |
| Web access | HTTP 200 / 3xx | `[RECORD]` |
| Validated on | — | `[DD Month YYYY]` |
| Validated by | Responsible engineer | `[NAME / ROLE]` |

> Keep command output or screenshots as deployment evidence. Do **not** mark the deployment complete while any failed unit remains.

---

## 12. After Installation — Out of Scope Here

Handled in the separate post-installation procedure:

| Task | Note |
|---|---|
| SSL certificate | Replace the self-signed certificate |
| License activation | Required for the licensed edition |
| Administrator password | Set per company policy |
| Backup initialization | Configure destination and schedule |
| Production DNS | MX, PTR, SPF/DKIM/DMARC finalised |

---

## 13. Deployment Record (fill after every install)

| Field | Value |
|---|---|
| Date / time | |
| Engineer | |
| Server FQDN / IP | |
| OS and version | |
| CPU / RAM / Disk | |
| Ansible core version | |
| `zxbot.carbonio_install` version | |
| `zxbot.carbonio_ssinstall` version | |
| Inventory file used | `carbonio-inventory-single` |
| Optional roles enabled | Files ☐  Preview ☐ |
| Single-server optimization approved by | |
| Playbook result (`PLAY RECAP`) | |
| Warnings / errors encountered | |
| Log location | `~/carbonio-deployment/carbonio-ansible.log` |
| Password files location | `~/carbonio-deployment/` |
| Snapshot name | |

---

## 14. Quick Reference

| Task | Command |
|---|---|
| Pre-flight check | `bash ~/carbonio-preflight.sh SERVER_FQDN` |
| Show inventory | `ansible-inventory -i carbonio-inventory-single --graph` |
| Test connection | `ansible all -i carbonio-inventory-single -u ANSIBLE_USER -m ping` |
| Test sudo | `ansible all -i carbonio-inventory-single -u ANSIBLE_USER --become -K -m command -a 'id -u'` |
| Install | `ansible-playbook zxbot.carbonio_ssinstall.carbonio_ssinstall -u ANSIBLE_USER -i carbonio-inventory-single --become -K --extra-vars "install_path=zxbot.carbonio_install.carbonio_install"` |
| Collection versions | `ansible-galaxy collection list \| grep zxbot` |
| Find failures in log | `grep -n -B 15 -A 25 -E 'FAILED!\|fatal:\|ERROR!' carbonio-ansible.log \| tail -n 100` |
| Failed units | `systemctl --failed` |
| Carbonio packages | `dpkg -l \| grep carbonio` (Ubuntu) · `rpm -qa \| grep carbonio` (RHEL) |

---