# CentOS Blue-Team Hardening Commands

For competition CentOS hosts only (lab / authorized PvP box: CentOS **chomper**). Test on kamino first. Do not run blind on a scored box.

Placeholders in angle brackets are values you fill in: **`<scoring-IP>`**, **`<scored-port>`**, **`<tcp-or-udp>`**, **`<allowed-port>`**, **`<interface>`**, **`<service-name>`**, **`<username>`**, **`<NTP-server>`**, **`<hypervisor>`**, **`<vm-name>`**, **`<vmid>`**, **`<snapshot-name>`**, **`<backup-archive-path>`**.

Each step lists Find (discover the values), then Do (make the change).

## 1) **Firewall**

Use **firewalld** (not ufw). Allow scored ports only from scoring IPs, then tighten the default zone. Add allow rules **before** tightening so you do not lock yourself out of **SSH**.

**Find**

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules
sudo firewall-cmd --get-default-zone
sudo ss -tulpn
ip -br addr
sudo ss -tn state established '( sport = :<scored-port> )'
sudo tcpdump -i <interface> -n port <scored-port>
```

- `firewall-cmd --list-all` shows the active zone, services, ports, and rich rules. `ss -tulpn` shows listening ports and programs (**`<scored-port>`**, **`<allowed-port>`**). `ip -br addr` gives **`<interface>`**. The last two show who connects to a scored port; run them while scoring checks happen to get **`<scoring-IP>`**.

**Do: start firewalld if it is not running**

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
```

**Do: add allow rules BEFORE tightening the default zone**

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<scoring-IP>" port port="<scored-port>" protocol="<tcp-or-udp>" accept'
sudo firewall-cmd --permanent --add-port=<allowed-port>/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

- Add one rich rule per scoring IP and scored port. Scope the source on every scored-port rule to scoring IPs only so no other source can exploit the port. Confirm **SSH** (port **22**) is allowed before the next block, or **you lose SSH**. Get the real ports and IPs from the scoring sheet or by watching the scoring VM's outbound traffic on kamino.

**Do: tighten the default zone only after the allow rules are in place**

```bash
sudo firewall-cmd --permanent --set-target=DROP
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

- `--set-target=DROP` silently drops everything the zone does not explicitly allow. If you set this before adding the SSH and scored-port rules, you lock yourself out. A lockout without a snapshot means a box reset, which **costs 200** points. Never break a scored service or its port.

### **SELinux**

Keep **SELinux enforcing**. Do not switch it to permissive or disabled as a shortcut — that is a common backdoor and it weakens the box.

**Find**

```bash
getenforce
sestatus
sudo ausearch -m avc -ts recent
```

- `getenforce` should print **Enforcing**. `sestatus` shows the current mode and the mode in `/etc/selinux/config`. `ausearch -m avc` lists SELinux denials (a service that fails only after a policy change often shows up here).

**Do**

```bash
sudo setenforce 1
sudo nano /etc/selinux/config
```

- In `/etc/selinux/config` set: `SELINUX=enforcing` so it stays on after reboot. Use `setenforce 1` for the running system; do not use `setenforce 0`.

If a scored service is denied on a nonstandard port, allow that port in policy instead of disabling SELinux:

```bash
sudo semanage port -l | grep <scored-port>
sudo semanage port -a -t ssh_port_t -p tcp <scored-port>
sudo restorecon -Rv /etc
```

- Pick the port type that matches the service (`ssh_port_t`, `http_port_t`, `mysqld_port_t`, and so on). `restorecon` resets labels on files after you change configs or restore a backup. Re-check denials with `sudo ausearch -m avc -ts recent` if something still fails.

## 2) **Disable unneeded services and daemons**

Stop, disable, and mask services the scoring engine does not check, keeping anything a scored service depends on.

**Find**

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service --state=enabled
systemctl list-dependencies <service-name>
systemctl list-dependencies --reverse <service-name>
systemctl cat <service-name>
```

- **`<service-name>`** comes from the running list, checked against the scoring sheet. `list-dependencies` shows what it requires, `--reverse` shows what depends on it, and `systemctl cat` shows its Requires= and Wants= lines. On **chomper**, keep the database service and anything it needs.

**Do**

```bash
sudo systemctl stop <service-name>
sudo systemctl disable <service-name>
sudo systemctl mask <service-name>
```

- Keep any service a scored service depends on, even if it is not scored itself. Never stop a scored service; a broken check costs uptime points.

## 3) **Lock down accounts and passwords**

Lock unused accounts, review wheel/sudo access, enforce a 14-character minimum with **pam_pwquality**, and rotate passwords.

**Find**

```bash
cat /etc/passwd
grep -v -E 'nologin|false' /etc/passwd
awk -F: '$3==0' /etc/passwd
getent group wheel
groups <username>
sudo chage -l <username>
```

- These list all accounts, accounts that can log in, UID 0 accounts (any besides root is a backdoor), **wheel** members, one user's groups, and password aging.

**Do**

```bash
sudo usermod -L <username>
sudo passwd -l <username>
sudo visudo
sudo nano /etc/security/pwquality.conf
```

- In `/etc/security/pwquality.conf` set: `minlen = 14`
```bash
sudo passwd <username>
sudo chage -M 90 <username>
```

- `usermod -L` and `passwd -l` both lock an account; either is fine. Do not remove users until scoring dependencies are confirmed.
- Scored users: also submit the new password through the scoring engine's password change request, or the check will fail.

## 4) **Patch and reboot**

Apply security updates with **dnf** (or **yum** on CentOS 7), then reboot.

**Find**

```bash
cat /etc/os-release
sudo dnf check-update
sudo yum check-update
```

- Current CentOS (8/9/Stream) uses **dnf**. **CentOS 7** uses **yum**. Use the line that matches this box.

**Do (current CentOS: dnf)**

```bash
sudo dnf update --security
sudo reboot
```

**Do (CentOS 7: yum)**

```bash
sudo yum install yum-plugin-security
sudo yum update --security
sudo reboot
```

- On CentOS 7, `yum update --security` needs the security plugin. After reboot, confirm scored services and **firewalld** came back, and that **SELinux** is still **Enforcing** (`getenforce`).

## 5) **Enable auditing**

Turn on **auditd**, watch `/etc/passwd` for changes, and watch login logs for attacks.

**Find**

```bash
sudo systemctl status auditd
sudo auditctl -l
```

**Do**

```bash
sudo dnf install audit
sudo systemctl enable auditd
sudo systemctl start auditd
sudo service auditd restart
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
```

- On **CentOS 7**, use `sudo yum install audit` instead of dnf. Install audit only if it is missing. On CentOS, **systemctl refuses to restart auditd**, so use `service auditd restart` whenever you need a restart.

**Monitor in real time**

```bash
sudo tail -f /var/log/secure
sudo journalctl -f -u sshd
sudo grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
sudo ausearch -m USER_AUTH,USER_LOGIN -ts recent
sudo aureport --auth
sudo ausearch -k passwd_changes
```

- CentOS writes sshd auth to **`/var/log/secure`**, not `auth.log`. `journalctl -u sshd` is the same trail from the journal. Red flag: repeated failed su or ssh attempts from one IP. The grep command counts failed SSH passwords by IP.

## 6) **Sync time with NTP**

Point **chrony** at an NTP peer, the time server the clock follows, so log timestamps line up with the scoring engine.

**Find**

- Get **`<NTP-server>`** from the scoring sheet or briefing first; competitions often publish their own. Otherwise the default is pool.ntp.org.
```bash
timedatectl status
systemctl status chronyd
chronyc sources
chronyc tracking
```

**Do**

- Edit `/etc/chrony.conf` and set: `server <NTP-server> iburst`
- Default for **`<NTP-server>`**: pool.ntp.org
```bash
sudo systemctl enable --now chronyd
sudo systemctl restart chronyd
```

**Confirm**

```bash
chronyc sources
chronyc tracking
timedatectl
```

## 7) **Back up before changes**

Archive configs on the VM so you can roll back without a box reset, staying inside the virtual environment.

**Ask Green Team first:** taking a machine snapshot or reverting a machine **without Green Team knowledge is a disqualification risk**. Ask Green Team **before** any hypervisor snapshot or revert. Only snapshot or revert if Green Team says yes.

**Find**

```bash
df -h
lsblk
```

- Pick a path on the VM with free space for **`<backup-archive-path>`** (a second disk or a data mount is better than filling `/`).

**Do: guest-side config archive (recommended default — always within the rules)**

```bash
sudo tar czf <backup-archive-path> /etc/ssh /etc/passwd /etc/group /etc/sudoers /etc/security/pwquality.conf /etc/chrony.conf /etc/selinux/config /etc/firewalld /var/log
```

- **`<backup-archive-path>`** is a `.tar.gz` file on the VM (or a disk attached to it). tar warns about any file that does not exist and still archives the rest.
- This is the **default** backup: it stays guest-side on the VM and does **not** use a hypervisor snapshot.

**Do: VM snapshot (optional — only with Green Team approval)**

- Use this **only after Green Team says yes**. Snapshot the VM from **`<hypervisor>`** before any changes, using the line for your platform. **`<vm-name>`**, **`<vmid>`**, and **`<snapshot-name>`** are yours to fill in.
```powershell
New-Snapshot -VM <vm-name> -Name <snapshot-name>
```

- VMware vSphere: PowerCLI command above, or in the vSphere Client right-click the VM > Snapshots > Take Snapshot.
```bash
VBoxManage snapshot "<vm-name>" take "<snapshot-name>"
```

- VirtualBox: run on the host.
```powershell
Checkpoint-VM -Name <vm-name> -SnapshotName <snapshot-name>
```

- Hyper-V: run in PowerShell on the host.
```bash
qm snapshot <vmid> <snapshot-name>
```

- Proxmox: run on the node, or use the VM's Snapshots tab > Take Snapshot.
- Snapshots need hypervisor access **and** Green Team approval. If Green Team says no, or your team only has the VM itself, use the guest-side config archive above.

## Closing reminder

Verify what the scoring engine checks before disabling anything (services, ports, users, or firewall). Keep the database reachable on **chomper**, and submit password change requests for any scored users. Never break a scored service.

## What to look out for

*Prerequisites and fallbacks*

- Before you start, know your scoring engine IPs and scored ports from the briefing or scoring sheet.
- Watch the scoring VM's outbound traffic on kamino to confirm where probes come from.
- Never disable a service or feature a scored item depends on. Check dependencies first.
- Firewall: add **firewalld** allow rules (SSH and scored-port rich rules) **before** tightening the default zone (`--set-target=DROP`), and scope the source to scoring IPs only. If you tighten first, **you lose SSH**.
- Keep **SELinux enforcing**. Do not set it to permissive or disabled to "fix" a service; use `ausearch -m avc`, `semanage port`, and `restorecon` instead.
- Scored account password changes may need to go through the scoring engine's change request process.
- The **guest-side** config archive (`tar` on the VM) is your default backup — always within the rules. Stay inside the virtual environment; do not depend on copying files off the lab. Do **not** take a hypervisor snapshot or revert unless **Green Team** has approved it.
- If you lock yourself out and have no approved snapshot, the fallback is a box reset, which **costs 200** points.
- On CentOS, restart **auditd** with `service auditd restart` — **systemctl** refuses.
- Current CentOS uses **dnf**; **CentOS 7** uses **yum**.
- Re-check after reboot: patches and time sync can reset. Verify with `chronyc tracking` or `timedatectl`, `sudo ss -tulpn` for listening ports, `sudo firewall-cmd --list-all`, and `getenforce`.
