# 🖥️ Born2beroot

![Score](https://img.shields.io/badge/Score-100%2F100-brightgreen)
![42](https://img.shields.io/badge/42-Project-black)

## 📌 Project Goal

**Born2beroot** is a system administration project. You set up a Linux virtual machine from scratch following strict security and configuration rules, then defend it in an oral evaluation where the evaluator will inspect your setup and ask you to demonstrate and explain everything live.

No code — pure sysadmin: partitioning, user management, firewall, SSH, password policy, sudo rules, and a monitoring script.

---

## 🧠 Key Concepts

### Virtual Machine
A VM runs an OS inside another OS using a **hypervisor** (VirtualBox or UTM). The guest OS is isolated — it has its own disk, network interfaces, and kernel, but shares the host's hardware through the hypervisor.

### Debian vs Rocky Linux
- **Debian**: community-maintained, uses `apt`, no SELinux by default (use AppArmor instead). Recommended for beginners.
- **Rocky Linux**: RHEL-based, uses `dnf`, uses **SELinux** by default.

The choice affects which security module you configure. Debian + AppArmor is the most common path.

### LVM — Logical Volume Manager
LVM adds an abstraction layer over physical partitions:
- **PV (Physical Volume)**: raw disk or partition (`/dev/sda5`)
- **VG (Volume Group)**: pool of PVs
- **LV (Logical Volume)**: virtual partition carved from a VG, mounted like a regular partition

Benefit: resize volumes without repartitioning. The bonus requires a specific LVM partition layout with an **encrypted** partition (`LUKS`).

### Encrypted Partition (LUKS)
LUKS encrypts a partition — you enter a passphrase at boot to unlock it. The LVM volumes live inside the encrypted partition. Commands: `cryptsetup luksFormat`, `cryptsetup open`.

### AppArmor
A **Mandatory Access Control (MAC)** system built into the Linux kernel. It confines programs to a defined set of resources using profiles. Must be active and enforcing at boot.
- Check status: `aa-status`

### SSH — Secure Shell
Encrypted remote login protocol. Configuration file: `/etc/ssh/sshd_config`.
- **Port**: must be changed to `4242` (not 22)
- **Root login**: must be disabled (`PermitRootLogin no`)
- Connect from host: `ssh user@127.0.0.1 -p 4242`

### UFW — Uncomplicated Firewall
A frontend for `iptables`. Only port **4242** should be open.
```bash
ufw allow 4242
ufw enable
ufw status
```

### sudo
Allows specific users to run commands as root with logging and restrictions. Configured in `/etc/sudoers` via `visudo` or a file in `/etc/sudoers.d/`.

Required sudo rules:
- Max 3 password attempts
- Custom error message on wrong password
- Log all sudo inputs and outputs to `/var/log/sudo/`
- Enable TTY mode (`Defaults requiretty`)
- Restrict PATH used by sudo (`Defaults secure_path=...`)

### Password Policy
Configured in `/etc/login.defs` and `/etc/pam.d/common-password` (with `libpam-pwquality`):
- Expire every **30 days** (`PASS_MAX_DAYS 30`)
- Minimum **2 days** before change allowed (`PASS_MIN_DAYS 2`)
- Warning **7 days** before expiry (`PASS_WARN_AGE 7`)
- Minimum **10 characters**, must contain uppercase + lowercase + digit
- Max **3 consecutive identical characters**
- Cannot contain the username
- At least **7 characters different** from previous password (non-root)

### User & Group Management
- Create a user in groups `user42` and `sudo`
- `useradd`, `usermod`, `groupadd`
- `groups <user>` — check group membership
- `chage -l <user>` — inspect password expiry settings

### monitoring.sh
A bash script that runs every **10 minutes** via **cron** (`@reboot` + `*/10 * * * *`) and broadcasts system info to all terminals with `wall`.

Must display:
- OS architecture and kernel version (`uname -a`)
- Physical and virtual CPU count (`/proc/cpuinfo`)
- RAM usage (used/total + %)
- Disk usage (used/total + %)
- CPU load (%)
- Last reboot (`who -b`)
- LVM active? (`lsblk | grep lvm`)
- Active TCP connections (`ss -ta | grep ESTAB`)
- Logged-in users (`who | wc -l`)
- IPv4 address and MAC address (`hostname -I`, `ip link`)
- Number of sudo commands run (`journalctl _COMM=sudo | grep COMMAND | wc -l`)

---

## 🔧 Step-by-Step Setup

### 1. Create the VM
- New VM in VirtualBox, choose Debian ISO
- Allocate ~12 GB disk (30 GB for bonus), 1–2 GB RAM
- No graphical interface — server install only

### 2. Partition (Bonus: LVM + Encryption)
During Debian installer:
- Choose **Manual** partitioning
- Create a small `/boot` partition (~500 MB, not encrypted)
- Create an encrypted partition with the rest → set up LVM inside it
- Carve LVM into: `root`, `swap`, `home`, `var`, `srv`, `tmp`, `var--log`

### 3. Base System
- No desktop environment
- Install only: `sudo`, `openssh-server`, `ufw`, `libpam-pwquality`, `vim`

### 4. Configure SSH
```bash
nano /etc/ssh/sshd_config
# Port 4242
# PermitRootLogin no
systemctl restart ssh
```

### 5. Configure UFW
```bash
ufw allow 4242
ufw enable
```

### 6. Configure sudo
```bash
visudo -f /etc/sudoers.d/sudoconfig
```
```
Defaults passwd_tries=3
Defaults badpass_message="Wrong password."
Defaults logfile="/var/log/sudo/sudo.log"
Defaults log_input, log_output
Defaults requiretty
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

### 7. Password Policy
```bash
# /etc/login.defs
PASS_MAX_DAYS 30
PASS_MIN_DAYS 2
PASS_WARN_AGE 7

# /etc/pam.d/common-password — add to pam_pwquality line:
minlen=10 ucredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

### 8. Users & Groups
```bash
useradd -m -G sudo,user42 <login>
passwd <login>
groupadd user42        # if not already created
chage -M 30 -m 2 -W 7 <login>
```

### 9. monitoring.sh + cron
```bash
chmod +x /usr/local/bin/monitoring.sh
crontab -e
# Add:
@reboot sleep 10 && /usr/local/bin/monitoring.sh
*/10 * * * * /usr/local/bin/monitoring.sh
```

---

## ⚠️ Important Notes

- **Hostname must be your login + `42`** (e.g. `hguo42`). The evaluator will ask you to change it live — `hostnamectl set-hostname <new>` then update `/etc/hosts`.
- **Password policy applies to existing users too**: after setting up `login.defs` and PAM, run `chage` on root and your user manually — new settings are not retroactive.
- **`/var/log/sudo/` must exist** before sudo tries to write there: `mkdir -p /var/log/sudo`.
- **AppArmor must be active at boot**: verify with `aa-status` and `systemctl status apparmor`.
- **Snapshot your VM** before the evaluation. If something breaks during the oral, you can revert.
- **Know your partition layout by heart**: `lsblk` output will be shown to the evaluator — be ready to explain every partition and why LVM is useful.
- **The evaluator will create a new user and assign it a group live** — practice `useradd`, `groupadd`, `usermod` until they're reflexive.
- **`wall` vs `echo`**: `monitoring.sh` must use `wall` to broadcast to all terminals, not just print to stdout.
- **Cron runs as root** — make sure the script path is absolute and the file is executable.
- **SSH port forwarding in VirtualBox**: in Network settings → Advanced → Port Forwarding, add a rule: host port `4242` → guest port `4242`, so you can SSH into the VM from the host terminal.
