*This project has been created as part of the 42 curriculum by fjose-hi.*

# Born2beRoot

## Description

Born2beRoot is a system administration project from the 42 curriculum. The goal is to create and configure a virtual machine from scratch, setting up a secure Linux server while following strict rules regarding partitioning, user management, firewall configuration, and security policies.

By completing this project, you gain hands-on experience with virtualization, Linux system administration, and best practices for server hardening — all without a graphical interface.

**Operating System chosen: Debian**

Debian was chosen over Rocky Linux for several reasons:
- It is explicitly recommended by the subject for newcomers to system administration.
- It has a larger community and more extensive documentation, making it easier to troubleshoot.
- APT is an intuitive and well-documented package manager.
- AppArmor (Debian's MAC system) is simpler to configure than SELinux (used on Rocky).
- Debian is known for its stability and long-term support (LTS) releases.
- Rocky Linux, being an enterprise-grade RHEL clone, is more complex to set up for a first server project.

---

## Instructions

### Requirements

- [VirtualBox](https://www.virtualbox.org/) (or UTM for Apple Silicon Macs)
- Debian ISO (latest stable version — no testing/unstable)

### 1. Virtual Machine Setup

1. Download the latest stable Debian ISO from [debian.org](https://www.debian.org/).
2. Open VirtualBox and create a new VM:
   - Type: **Linux**, Version: **Debian (64-bit)**
   - RAM: at least **1024 MB**
   - Disk: at least **8 GB** (dynamically allocated VDI)
3. Attach the Debian ISO to the VM's optical drive and start the VM.

### 2. Installing Debian

During the Debian installer:

1. **Configure locals** — choose your language, country, and locale.
2. **Configure the network** — set the hostname to `<login>42`.
3. **Set up users and passwords** — create a root password and a non-root user `<login>`.
4. **Configure the clock** — select your timezone.
5. **Partition disks** — use manual partitioning to create at least 2 encrypted LVM partitions:

**Mandatory partition layout (minimum):**
```
sda
├─sda1          /boot    (unencrypted, ~487MB)
├─sda2          (1KB, extended)
└─sda5_crypt    (encrypted LVM group)
    ├─LVMGroup-root    /        (~2.8GB, lvm)
    ├─LVMGroup-swap    [SWAP]   (~976MB, lvm)
    └─LVMGroup-home    /home    (~3.8GB, lvm)
```

**Bonus partition layout:**
```
sda
├─sda1              /boot     500M
└─sda5_crypt        (encrypted LVM group, ~30GB)
    ├─LVMGroup-root     /         10G
    ├─LVMGroup-swap     [SWAP]    2.3G
    ├─LVMGroup-home     /home     5G
    ├─LVMGroup-var      /var      3G
    ├─LVMGroup-srv      /srv      3G
    ├─LVMGroup-tmp      /tmp      3G
    └─LVMGroup-var--log /var/log  4G
```

6. **Configure the package manager** — skip the CD-ROM scan, configure the network mirror (Portugal → deb.debian.org).
7. **Install the GRUB boot loader** — select Yes and choose `/dev/sda`.
8. **Finish installation** and reboot.

### 3. First Connection

After rebooting:
1. Select **Debian GNU/Linux** from the GRUB menu.
2. Enter the **encryption password** you set during partitioning.
3. Log in with your non-root username (`<login>`) and your password.

### 4. System Configuration

#### Installing sudo & Configuring Users and Groups

```bash
# Switch to root
su -

# Install sudo
apt install sudo

# Add <login> to sudo group
usermod -aG sudo <login>

# Create the user42 group and add <login> to it
groupadd user42
usermod -aG user42 <login>

# Verify groups
groups <login>
```

#### Installing & Configuring SSH

```bash
# Install OpenSSH server
sudo apt install openssh-server

# Edit SSH config
sudo nano /etc/ssh/sshd_config
```

Change the following lines:
```
Port 4242
PermitRootLogin no
```

```bash
# Restart SSH service
sudo systemctl restart ssh

# Verify SSH is running on port 4242
sudo ss -tlnp | grep ssh
```

#### Installing & Configuring UFW Firewall

```bash
# Install UFW
sudo apt install ufw

# Enable UFW
sudo ufw enable

# Allow only port 4242
sudo ufw allow 4242

# Verify status
sudo ufw status
```

#### Connecting via SSH

From another terminal on the host machine:
```bash
ssh <login>@127.0.0.1 -p 4242
```

> Make sure to configure port forwarding in VirtualBox: Host port **4242** → Guest port **4242**.

#### Sudo Policies

```bash
# Create the sudo log directory
sudo mkdir /var/log/sudo

# Create the sudo config file
sudo touch /etc/sudoers.d/sudo_config

# Edit the file
sudo nano /etc/sudoers.d/sudo_config
```

Add the following content:
```
Defaults  passwd_tries=3
Defaults  badpass_message="Wrong password. Try again!"
Defaults  logfile="/var/log/sudo/sudo_config"
Defaults  log_input, log_output
Defaults  iolog_dir="/var/log/sudo"
Defaults  requiretty
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

#### Password Policy

**Step 1 — Edit `/etc/login.defs`:**
```bash
sudo nano /etc/login.defs
```

Modify these values:
```
PASS_MAX_DAYS   30
PASS_MIN_DAYS   2
PASS_WARN_AGE   7
```

**Step 2 — Install and configure `libpam-pwquality`:**
```bash
sudo apt install libpam-pwquality

sudo nano /etc/pam.d/common-password
```

Below the line containing `retry=3`, add:
```
minlen=10 ucredit=-1 dcredit=-1 lcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

| Parameter | Meaning |
|---|---|
| `minlen=10` | Minimum 10 characters |
| `ucredit=-1` | At least 1 uppercase letter |
| `dcredit=-1` | At least 1 digit |
| `lcredit=-1` | At least 1 lowercase letter |
| `maxrepeat=3` | No more than 3 consecutive identical characters |
| `reject_username` | Cannot contain the username |
| `difok=7` | At least 7 characters different from previous password |
| `enforce_for_root` | Policy applies to root too |

> ⚠️ After setting up the policy, change all existing passwords (including root) with `passwd <username>`.

### 5. Monitoring Script

Create `/usr/local/bin/monitoring.sh`:

```bash
#!/bin/bash

# ARCH
arch=$(uname -a)

# CPU PHYSICAL
cpuf=$(grep "physical id" /proc/cpuinfo | wc -l)

# CPU VIRTUAL
cpuv=$(grep "processor" /proc/cpuinfo | wc -l)

# RAM
ram_total=$(free --mega | awk '$1 == "Mem:" {print $2}')
ram_use=$(free --mega | awk '$1 == "Mem:" {print $3}')
ram_percent=$(free --mega | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')

# DISK
disk_total=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_t += $2} END {printf ("%.1fGb\n"), disk_t/1024}')
disk_use=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} END {print disk_u}')
disk_percent=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} {disk_t+= $2} END {printf("%d"), disk_u/disk_t*100}')

# CPU LOAD
cpul=$(vmstat 1 2 | tail -1 | awk '{printf $15}')
cpu_op=$(expr 100 - $cpul)
cpu_fin=$(printf "%.1f" $cpu_op)

# LAST BOOT
lb=$(who -b | awk '$1 == "system" {print $3 " " $4}')

# LVM USE
lvmu=$(if [ $(lsblk | grep "lvm" | wc -l) -gt 0 ]; then echo yes; else echo no; fi)

# TCP CONNECTIONS
tcpc=$(ss -ta | grep ESTAB | wc -l)

# USER LOG
ulog=$(users | wc -w)

# NETWORK
ip=$(hostname -I)
mac=$(ip link | grep "link/ether" | awk '{print $2}')

# SUDO
cmnd=$(journalctl _COMM=sudo | grep COMMAND | wc -l)

wall "	Architecture: $arch
	CPU physical: $cpuf
	vCPU: $cpuv
	Memory Usage: $ram_use/${ram_total}MB ($ram_percent%)
	Disk Usage: $disk_use/${disk_total} ($disk_percent%)
	CPU load: $cpu_fin%
	Last boot: $lb
	LVM use: $lvmu
	Connections TCP: $tcpc ESTABLISHED
	User log: $ulog
	Network: IP $ip ($mac)
	Sudo: $cmnd cmd"
```

Give it execute permissions:
```bash
sudo chmod +x /usr/local/bin/monitoring.sh
```

### 6. Crontab

Schedule the script to run every 10 minutes as root:

```bash
sudo crontab -e
```

Add this line:
```
*/10 * * * * /usr/local/bin/monitoring.sh
```

To **interrupt the script without modifying it**, simply comment out or remove the cron entry:
```bash
sudo crontab -e
# Comment out the line: #*/10 * * * * /usr/local/bin/monitoring.sh
```

### 7. Generating the Signature

```bash
# On Linux
sha1sum ~/VirtualBox\ VMs/<login>42/<login>42.vdi

# On macOS
shasum ~/VirtualBox\ VMs/<login>42/<login>42.vdi
```

Paste the hash into `signature.txt` at the root of your Git repository.

> ⚠️ The VM signature changes every time the machine is started. Keep a duplicate of the `.vdi` or use snapshots. Never include the VM in your Git repository.

---

## Project Description

### What is a Virtual Machine?

A virtual machine (VM) is software that simulates a complete computer system, allowing programs to run as if on real hardware. It allows the creation of multiple isolated environments from a single physical machine — useful for testing, development, running different operating systems, and server administration.

### What is LVM?

LVM (Logical Volume Manager) is a device mapper framework that provides flexible logical volume management for Linux. It separates storage into three layers:

- **Physical Volumes (PV):** Physical drives or partitions.
- **Volume Groups (VG):** One or more PVs pooled together.
- **Logical Volumes (LV):** Virtual partitions carved from a VG, which can be resized on-the-fly.

Benefits over traditional partitioning: dynamic resizing without unmounting, snapshots for backups, and easy management across multiple disks.

### apt vs aptitude

| Feature | apt | aptitude |
|---|---|---|
| Level | Lower-level | Higher-level (built on top of apt) |
| Interface | Command-line only | CLI + interactive text UI |
| Dependency resolution | Standard | Smarter, more interactive |
| Orphan packages | Manual cleanup needed | Better automatic handling |
| Speed | Faster and lighter | Slightly heavier |
| Default use | Scripts and automation | Interactive package management |

### OS Choice: Debian vs Rocky Linux

| Feature | Debian | Rocky Linux |
|---|---|---|
| Base | Independent | RHEL clone |
| Package manager | APT (`apt`) | DNF (`dnf`) |
| MAC system | AppArmor | SELinux |
| Firewall | UFW | firewalld |
| Complexity | Beginner-friendly | More complex |
| Target | General-purpose servers | Enterprise environments |
| LTS support | ~5 years | ~10 years |

**Why Debian:** More approachable for beginners, better community documentation, simpler security tooling with AppArmor, and a well-established ecosystem.

### AppArmor vs SELinux

| Feature | AppArmor | SELinux |
|---|---|---|
| Model | Path-based profiles | Label-based (type enforcement) |
| Complexity | Simpler — profiles per application | More complex — system-wide labeling |
| Default on | Debian/Ubuntu | RHEL/Rocky/Fedora |
| Configuration | `/etc/apparmor.d/` | `/etc/selinux/` |
| Ease of use | Easier to write and manage | Steeper learning curve |
| Granularity | Per-program policies | Fine-grained system-wide access control |

Both are Mandatory Access Control (MAC) frameworks that restrict what processes can do beyond standard Unix permissions. AppArmor uses file path-based profiles; SELinux uses labels assigned to every object on the system.

### UFW vs firewalld

| Feature | UFW | firewalld |
|---|---|---|
| Full name | Uncomplicated Firewall | Dynamic Firewall Manager |
| Default on | Debian/Ubuntu | RHEL/Rocky/Fedora |
| Backend | iptables / nftables | nftables / iptables |
| Interface | Simple command-line | Zones-based (CLI + D-Bus) |
| Complexity | Very simple | More feature-rich |
| Dynamic rules | No (requires reload) | Yes (changes apply live) |

UFW is designed to be easy to use, ideal for this project. firewalld introduces zones and services for more flexibility, but requires more configuration knowledge.

### VirtualBox vs UTM

| Feature | VirtualBox | UTM |
|---|---|---|
| Developer | Oracle | Open-source (QEMU-based) |
| Platform | Windows, macOS, Linux | macOS only |
| Apple Silicon (M1/M2) | Not natively supported | Native via QEMU/Apple Hypervisor |
| Performance on Intel/AMD | Excellent | Good |
| Ease of use | Very user-friendly GUI | Good GUI, slightly more technical |
| Snapshot support | Yes | Yes |

VirtualBox is the standard choice for x86/x64 machines. UTM is the required alternative for Apple Silicon Macs where VirtualBox does not run natively.

### Main Design Choices

**Partitioning:** LVM over an encrypted LUKS partition was used to allow flexible volume management while securing data at rest. The encrypted group contains logical volumes for `/`, `/home`, and `[SWAP]` (mandatory), plus `/var`, `/srv`, `/tmp`, and `/var/log` for the bonus.

**Security policies:**
- SSH locked to port 4242 with root login disabled (`PermitRootLogin no`).
- UFW configured to allow only port 4242.
- Strong password policy enforced via PAM (`libpam-pwquality`) and `/etc/login.defs`.
- sudo restricted to 3 attempts, with a custom error message, full I/O logging to `/var/log/sudo/`, TTY mode enabled, and secure paths enforced.

**User management:** Non-root user `<login>` created during installation and added to both `sudo` and `user42` groups.

**Services installed:** openssh-server, ufw, sudo, libpam-pwquality. AppArmor is active by default on Debian.

---

## Resources

- [Debian Official Documentation](https://www.debian.org/doc/)
- [Born2beRoot comprehensive guide](https://noreply.gitbook.io/born2beroot)
- [UFW man page](https://manpages.ubuntu.com/manpages/focal/en/man8/ufw.8.html)
- [AppArmor documentation](https://apparmor.net/)
- [LVM HOWTO](https://tldp.org/HOWTO/LVM-HOWTO/)
- [PAM pwquality documentation](https://linux.die.net/man/8/pam_pwquality)
- [cron documentation](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [VirtualBox User Manual](https://www.virtualbox.org/manual/)
- [ss command reference](https://man7.org/linux/man-pages/man8/ss.8.html)
- [journalctl documentation](https://www.freedesktop.org/software/systemd/man/journalctl.html)

**AI usage:** AI was used to help structure and review this README, to clarify tool comparisons (AppArmor vs SELinux, UFW vs firewalld, apt vs aptitude, etc.), and to verify bash script syntax in the monitoring script. All technical implementation, configuration choices, and system administration steps were performed and understood manually before being documented here.

---

## License

This project was developed for educational purposes as part of the 42 curriculum.

---

## Author

Felipe José Hillebrand

GitHub: https://github.com/felipehillebrand-ops
