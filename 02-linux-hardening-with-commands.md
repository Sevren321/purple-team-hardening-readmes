# Linux Blue-Team Hardening Commands

For competition Linux hosts only (lab / authorized PvP boxes: Ubuntu cherrybomb, CentOS chomper, Debian peashooter). Test on kamino first. Do not run blind on a scored box.

Placeholders in angle brackets are values you fill in: `<scored-port>`, `<scoring-engine-IP>`, `<scoring-IP-1>`, `<scoring-IP-2>`, `<tcp-or-udp>`, `<allowed-port>`, `<interface>`, `<service>`, `<username>`, `<distro-package-manager>`, `<NTP-server>`, `<hypervisor>`, `<vm-name>`, `<vmid>`, `<snapshot-name>`, `<backup-archive-path>`.

Each step lists Find (discover the values), then Do (make the change).

## 1) Firewall

Install a firewall, deny inbound by default, and allow only scored ports from scoring IPs.

**Find**

```
which ufw firewall-cmd
sudo ufw status numbered
sudo firewall-cmd --list-all
sudo ss -tulpn
ip -br addr
sudo ss -tn state established '( sport = :<scored-port> )'
sudo tcpdump -i <interface> -n port <scored-port>
```

- ss -tulpn shows listening ports and programs (`<scored-port>`, `<allowed-port>`). ip -br addr gives `<interface>`. The last two show who connects to a scored port; run them while scoring checks happen to get `<scoring-engine-IP>`.

**Do: install (pick your distro)**

```
sudo apt install ufw
sudo yum install firewalld
```

**Do: ufw (Ubuntu / Debian), add allow rules BEFORE enabling so you do not lock yourself out**

```
sudo ufw default deny incoming
sudo ufw allow from <scoring-engine-IP> to any port <scored-port> proto <tcp-or-udp>
sudo ufw allow from <scoring-IP-1> to any port <scored-port> proto <tcp-or-udp>
sudo ufw allow from <scoring-IP-2> to any port <scored-port> proto <tcp-or-udp>
sudo ufw allow <allowed-port>/tcp
sudo ufw allow in on <interface> to any port <allowed-port> proto tcp
sudo ufw enable
sudo ufw status verbose
```

- ufw takes one source per rule, so add one rule per scoring IP. The interface rule is optional.

**Do: firewalld (CentOS)**

```
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<scoring-engine-IP>" port port="<scored-port>" protocol="<tcp-or-udp>" accept'
sudo firewall-cmd --permanent --add-port=<allowed-port>/tcp
sudo firewall-cmd --reload
```

- Scope the source on every scored-port rule to scoring IPs only so no other source can exploit the port. Get the real ports and IPs from the scoring sheet or by watching the scoring VM's outbound traffic on kamino.

## 2) Disable unneeded services and daemons

Stop, disable, and mask services the scoring engine does not check, keeping anything a scored service depends on.

**Find**

```
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service --state=enabled
systemctl list-dependencies <service>
systemctl list-dependencies --reverse <service>
systemctl cat <service>
```

- `<service>` comes from the running list, checked against the scoring sheet. list-dependencies shows what it requires, --reverse shows what depends on it, and systemctl cat shows its Requires= and Wants= lines.

**Do**

```
sudo systemctl stop <service>
sudo systemctl disable <service>
sudo systemctl mask <service>
```

- Keep any service a scored service depends on, even if it is not scored itself.

## 3) Lock down accounts and passwords

Lock unused accounts, review sudo access, enforce a 14-character minimum, and rotate passwords.

**Find**

```
cat /etc/passwd
grep -v -E 'nologin|false' /etc/passwd
awk -F: '$3==0' /etc/passwd
getent group sudo wheel
groups <username>
sudo chage -l <username>
```

- These list all accounts, accounts that can log in, UID 0 accounts (any besides root is a backdoor), sudo/wheel members, one user's groups, and password aging.

**Do**

```
sudo passwd -l <username>
sudo visudo
sudo nano /etc/security/pwquality.conf
```

- In /etc/security/pwquality.conf set: minlen = 14
```
sudo passwd <username>
sudo chage -M 90 <username>
```

- Do not remove users until scoring dependencies are confirmed.
- Scored users: also submit the new password through the scoring engine's password change request, or the check will fail.

## 4) Patch and reboot

Update packages with the distro package manager (security updates where possible), then reboot.

**Find**

```
cat /etc/os-release
apt list --upgradable
sudo yum check-update
```

- `<distro-package-manager>` is apt (Ubuntu / Debian) or yum (CentOS).

**Do**

```
sudo <distro-package-manager> update && sudo <distro-package-manager> upgrade
sudo reboot
```

## 5) Enable auditing

Turn on auditd, watch /etc/passwd for changes, and watch login logs for attacks.

**Find**

```
sudo systemctl status auditd
sudo auditctl -l
```

**Do**

```
sudo apt install auditd
sudo yum install audit
sudo systemctl enable auditd
sudo systemctl start auditd
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
```

- Install auditd only if it is missing.

**Monitor in real time**

```
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/secure
```

- auth.log is Debian / Ubuntu, secure is RHEL / CentOS. If auth.log is missing (newer Debian), use the journal:
```
sudo journalctl -f -u ssh
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
sudo ausearch -m USER_AUTH,USER_LOGIN -ts recent
sudo aureport --auth
sudo ausearch -k passwd_changes
```

- Red flag: repeated failed su or ssh attempts from one IP. The grep command counts failed SSH passwords by IP.

## 6) Sync time with NTP

Point chrony or systemd-timesyncd at an NTP peer, the time server the clock follows, so log timestamps line up with the scoring engine.

**Find**

- Get `<NTP-server>` from the scoring sheet or briefing first; competitions often publish their own. Otherwise the default is pool.ntp.org.
```
timedatectl status
systemctl status chronyd systemd-timesyncd
chronyc sources
```

- On Debian / Ubuntu the chrony service may be named chrony instead of chronyd.

**Do**

- Edit the config for the service in use: /etc/chrony.conf or /etc/systemd/timesyncd.conf
- timesyncd.conf line: NTP=`<NTP-server>`
- chrony.conf line: server `<NTP-server>` iburst
- Default for `<NTP-server>`: pool.ntp.org
```
sudo systemctl restart chronyd
sudo systemctl restart systemd-timesyncd
```

**Confirm**

```
chronyc tracking
timedatectl
```

## 7) Back up before changes

Snapshot the VM and archive configs off the box so you can roll back without a box reset.

**Find**

```
df -h
lsblk
```

- Find a mounted share or second disk with free space for `<backup-archive-path>`.

**Do: VM snapshot (primary rollback)**

- Snapshot the VM from `<hypervisor>` before any changes, using the line for your platform. `<vm-name>`, `<vmid>`, and `<snapshot-name>` are yours to fill in.
```
New-Snapshot -VM <vm-name> -Name <snapshot-name>
```

- VMware vSphere: PowerCLI command above, or in the vSphere Client right-click the VM > Snapshots > Take Snapshot.
```
VBoxManage snapshot "<vm-name>" take "<snapshot-name>"
```

- VirtualBox: run on the host.
```
Checkpoint-VM -Name <vm-name> -SnapshotName <snapshot-name>
```

- Hyper-V: run in PowerShell on the host.
```
qm snapshot <vmid> <snapshot-name>
```

- Proxmox: run on the node, or use the VM's Snapshots tab > Take Snapshot.
- Snapshots need hypervisor access. If your team only has the VM itself, skip to the config backup below.

**Do: config archive**

```
sudo tar czf <backup-archive-path> /etc/ssh /etc/passwd /etc/group /etc/sudoers /etc/security/pwquality.conf /etc/chrony.conf /etc/systemd/timesyncd.conf /var/log
```

- `<backup-archive-path>` is a .tar.gz file on that share or second disk. tar warns about any file that does not exist on this distro and still archives the rest.
- The hypervisor snapshot is the primary rollback; the archive is the fallback.

## Closing reminder

Verify what the scoring engine checks before disabling anything (services, ports, users, or firewall). Keep Pi-hole and HTTP reachable on cherrybomb, the database reachable on chomper, and SSH reachable on peashooter, and submit password change requests for any scored users.

## What to look out for

*Prerequisites and fallbacks*

- Before you start, know your scoring engine IPs and scored ports from the briefing or scoring sheet.
- Watch the scoring VM's outbound traffic on kamino to confirm where probes come from.
- Never disable a service or feature a scored item depends on. Check dependencies first.
- Firewall: add allow rules before enabling, and scope the source (remoteip) to scoring IPs only.
- ufw: add allow rules before ufw enable, or you lose SSH.
- Scored account password changes may need to go through the scoring engine's change request process.
- The hypervisor snapshot is your primary rollback. Take it before changes.
- If you lock yourself out, the VM snapshot is how you get back in. Without hypervisor access, the fallback is a box reset, which costs 200 points.
- Re-check after reboot: patches and time sync can reset. Verify with chronyc tracking or timedatectl, and sudo ss -tulpn for listening ports.
