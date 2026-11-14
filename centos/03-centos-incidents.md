# CentOS Incident Fixes (Blue-Team Shift)

For competition incidents on your own team's CentOS boxes only (authorized PvP environment, including **chomper**). Each incident lists Find (confirm what happened), then Fix.

Placeholders in angle brackets are values you fill in: **`<username>`**, **`<new-password>`**, **`<rogue-user>`**, **`<pattern>`**, **`<cron-file>`**, **`<timer-name>`**, **`<minutes>`**, **`<file-path>`**, **`<quarantine-folder>`**, **`<process-id>`**, **`<service-name>`**, **`<scoring-IP>`**, **`<scored-port>`**, **`<tcp-or-udp>`**, **`<allowed-port>`**, **`<NTP-server>`**.

Never modify, move, or delete any flag file (for example `/root/flag.txt` or `/user/flag.txt`). That is a **disqualification**.

## 1) **Locked out / rogue accounts**

An attacker locked, expired, or changed your accounts, or planted their own way in.

**Find**

```bash
sudo passwd -S <username>
sudo chage -l <username>
sudo faillock --user <username>
getent passwd <username>
awk -F: '$3==0' /etc/passwd
getent group wheel
sudo cat /home/<username>/.ssh/authorized_keys
sudo cat /root/.ssh/authorized_keys
```

- `passwd -S` shows L for locked. Also check that the user's shell (last field of `getent passwd`) is not `/usr/sbin/nologin`, that no account besides root has UID 0, and that there are no SSH keys you did not add.

**Fix**

```bash
sudo passwd -u <username>
sudo usermod -U <username>
sudo faillock --user <username> --reset
sudo chage -E -1 <username>
sudo usermod -s /bin/bash <username>
sudo passwd <username>
sudo gpasswd -d <rogue-user> wheel
sudo usermod -L <rogue-user>
sudo passwd -l <rogue-user>
```

- Remove any unknown keys from **`authorized_keys`**. Scored users: submit any **`<new-password>`** through the scoring engine's password change request.

## 2) **Malicious cron jobs**

An attacker added a cron job or systemd timer to re-open access or undo your fixes.

**Find**

```bash
sudo crontab -l
sudo crontab -l -u <username>
for u in $(cut -d: -f1 /etc/passwd); do sudo crontab -l -u $u 2>/dev/null | sed "s/^/$u: /"; done
cat /etc/crontab
ls -la /etc/cron.d /etc/cron.hourly /etc/cron.daily /etc/cron.weekly /etc/cron.monthly /var/spool/cron
sudo grep -r "<pattern>" /etc/cron* /var/spool/cron 2>/dev/null
systemctl list-timers --all
```

- The loop shows every user's crontab. Check `/etc/cron.*` and `/var/spool/cron` as well as `crontab -l`. **`<pattern>`** is something suspicious to search for, like an IP, `nc`, `bash -i`, or `curl`. Timers are the systemd version of cron.

**Fix**

```bash
sudo crontab -e -u <username>
sudo rm /etc/cron.d/<cron-file>
sudo systemctl disable --now <timer-name>.timer
```

- In `crontab -e`, delete the bad line and save. Confirm a job is not part of a scored service before removing it.

## 3) **Dropped scripts / webshells**

An attacker left shell, Python, or web-shell scripts on the box.

**Find**

```bash
sudo find / \( -name "*.sh" -o -name "*.py" -o -name "*.php" -o -name "*.jsp" -o -name "*.cgi" \) -type f -newer /etc/passwd -not -path "/proc/*" 2>/dev/null
sudo find /tmp /var/tmp /dev/shm /home /root /var/www /usr/share/nginx /var/lib/mysql -type f -mmin -<minutes> 2>/dev/null
cat <file-path>
ps aux | grep "<file-path>"
```

- The parentheses and quotes matter, or find mismatches. `-newer /etc/passwd` lists files changed after the last account change, so if you just rotated passwords, use the `-mmin` line instead (files changed in the last **`<minutes>`**). Read a script or suspected webshell before you remove it.

**Fix**

```bash
sudo kill -9 <process-id>
sudo mkdir -p <quarantine-folder>
sudo mv <file-path> <quarantine-folder>
sudo rm <file-path>
```

- Kill any process running it first. Moving it to quarantine keeps evidence; remove it once you are sure. Do not touch flag files.

## 4) **firewalld rule tampering**

An attacker turned off **firewalld**, wiped your rules, or added a wide-open allow.

**Find**

```bash
sudo firewall-cmd --state
sudo systemctl status firewalld
sudo firewall-cmd --list-all
sudo firewall-cmd --list-rich-rules
sudo iptables -S
sudo nft list ruleset
```

- Look for a dead **firewalld** service, a missing rich rule for **`<scoring-IP>`**, or a new rule that allows any source to a scored port.

**Fix**

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<scoring-IP>" port port="<scored-port>" protocol="<tcp-or-udp>" accept'
sudo firewall-cmd --permanent --add-port=<allowed-port>/tcp
sudo firewall-cmd --permanent --set-target=DROP
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

- Rebuild the scoped rules from the scoring sheet **before** tightening the default zone, or **you lose SSH**. Add allow rules first, then `--set-target=DROP`, then `--reload`. Never break a scored service.

## 5) **Re-enabled / rogue services**

An attacker turned back on a service you disabled, added a new one, or turned **SELinux** off as a backdoor.

**Find**

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service --state=enabled
sudo find /etc/systemd/system /usr/lib/systemd/system -name "*.service" -mmin -<minutes>
systemctl cat <service-name>
getenforce
sestatus
cat /etc/selinux/config
```

- A new unit file, or an ExecStart pointing at `/tmp` or a home folder, is suspicious. If `getenforce` is **Permissive** or **Disabled**, or `/etc/selinux/config` no longer says `SELINUX=enforcing`, treat that as a backdoor and turn enforcing back on.

**Fix**

```bash
sudo systemctl stop <service-name>
sudo systemctl disable <service-name>
sudo systemctl mask <service-name>
sudo setenforce 1
sudo nano /etc/selinux/config
```

- Masking stops the service from being started again, even by another service. In `/etc/selinux/config` set `SELINUX=enforcing`. If a scored service then fails, use `sudo ausearch -m avc -ts recent` and fix the label or port — do not leave SELinux off.

## 6) **auditd tampering / cleared logs**

An attacker stopped **auditd**, removed your rules, or wiped logs.

**Find**

```bash
sudo systemctl status auditd
sudo auditctl -s
sudo auditctl -l
sudo ausearch -m USER_AUTH
sudo ausearch -m CONFIG_CHANGE,DAEMON_END -ts today
ls -la /var/log /var/log/audit
sudo wc -l /var/log/secure /var/log/audit/audit.log
```

- `auditctl -s` shows if auditing is enabled. CONFIG_CHANGE and DAEMON_END events show rule changes and auditd stopping. Tiny or empty log files in `/var/log` or `/var/log/audit` point to wiping.

**Fix**

```bash
sudo systemctl start auditd
sudo service auditd restart
sudo auditctl -e 1
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
```

- On CentOS, **systemctl refuses to restart auditd**, so use `service auditd restart`. Re-add any watch rules that went missing. Note the gap in `/var/log/secure` and `/var/log/audit/audit.log`; you cannot recover wiped lines, but you can resume logging.

## 7) **Time change**

An attacker changed the clock or the time source, which breaks log timelines.

**Find**

```bash
date
timedatectl
chronyc tracking
chronyc sources
cat /etc/chrony.conf
```

**Fix**

```bash
sudo timedatectl set-ntp true
```

- If the NTP server was changed, set it back to **`<NTP-server>`** in `/etc/chrony.conf` (`server <NTP-server> iburst`) before restarting.
```bash
sudo chronyc makestep
sudo systemctl restart chronyd
chronyc sources
chronyc tracking
timedatectl
```

## Before destructive action

Report to the scoring team (competition staff) before any destructive action on a scored item: deleting a scored user, stopping a service a scored check uses, or anything you are not sure about. A broken scored service costs uptime points, and a box reset **costs 200**.
