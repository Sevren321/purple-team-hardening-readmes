# Linux Incident Fixes (Blue-Team Shift)

For competition incidents on your own team's Linux boxes only (authorized PvP environment). Each incident lists Find (confirm what happened), then Fix.

Placeholders in angle brackets are values you fill in: **`<username>`**, **`<new-password>`**, **`<rogue-user>`**, **`<pattern>`**, **`<cron-file>`**, **`<timer-name>`**, **`<minutes>`**, **`<file-path>`**, **`<quarantine-folder>`**, **`<process-id>`**, **`<service>`**, **`<scoring-engine-IP>`**, **`<scored-port>`**, **`<tcp-or-udp>`**, **`<allowed-port>`**, **`<NTP-server>`**.

Never modify, move, or delete any flag file (for example `/root/flag.txt` or `/user/flag.txt`). That is a **disqualification**.

## 1) **Locked out**

An attacker locked, expired, or changed your accounts, or planted their own way in.

**Find**

```bash
sudo passwd -S <username>
sudo chage -l <username>
sudo faillock --user <username>
getent passwd <username>
awk -F: '$3==0' /etc/passwd
getent group sudo wheel
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
sudo gpasswd -d <rogue-user> sudo
sudo usermod -L <rogue-user>
```

- Remove any unknown keys from **`authorized_keys`**. Scored users: submit any new password through the scoring engine's password change request.

## 2) **Cron sabotage**

An attacker added a cron job or systemd timer to re-open access or undo your fixes.

**Find**

```bash
sudo crontab -l
sudo crontab -l -u <username>
for u in $(cut -d: -f1 /etc/passwd); do sudo crontab -l -u $u 2>/dev/null | sed "s/^/$u: /"; done
cat /etc/crontab
ls -la /etc/cron.d /etc/cron.hourly /etc/cron.daily /var/spool/cron
sudo grep -r "<pattern>" /etc/cron* /var/spool/cron 2>/dev/null
systemctl list-timers --all
```

- The loop shows every user's crontab. **`<pattern>`** is something suspicious to search for, like an IP, `nc`, `bash -i`, or `curl`. Timers are the systemd version of cron.

**Fix**

```bash
sudo crontab -e -u <username>
sudo rm /etc/cron.d/<cron-file>
sudo systemctl disable --now <timer-name>.timer
```

- In `crontab -e`, delete the bad line and save. Confirm a job is not part of a scored service before removing it.

## 3) **Malicious scripts dropped**

An attacker left shell or Python scripts on the box.

**Find**

```bash
sudo find / \( -name "*.sh" -o -name "*.py" \) -type f -newer /etc/passwd -not -path "/proc/*" 2>/dev/null
sudo find /tmp /var/tmp /dev/shm /home /root -type f -mmin -<minutes> 2>/dev/null
cat <file-path>
ps aux | grep "<file-path>"
```

- The parentheses and quotes matter, or find mismatches. `-newer /etc/passwd` lists files changed after the last account change, so if you just rotated passwords, use the `-mmin` line instead (files changed in the last **`<minutes>`**). Read a script before you remove it.

**Fix**

```bash
sudo kill -9 <process-id>
sudo mkdir -p <quarantine-folder>
sudo mv <file-path> <quarantine-folder>
sudo rm <file-path>
```

- Kill any process running it first. Moving it to quarantine keeps evidence; remove it once you are sure.

## 4) **Firewall flushed or disabled**

An attacker turned off the firewall or wiped your rules.

**Find**

```bash
sudo ufw status verbose
sudo firewall-cmd --state
sudo firewall-cmd --list-all
sudo iptables -S
sudo nft list ruleset
```

**Fix (ufw: Ubuntu / Debian), allow rules BEFORE enabling**

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow from <scoring-engine-IP> to any port <scored-port> proto <tcp-or-udp>
sudo ufw allow <allowed-port>/tcp
sudo ufw enable
sudo ufw status numbered
```

- `ufw reset` turns ufw off and deletes all rules, so rebuild the scoped rules from the scoring sheet before `ufw enable`, or **you **lose SSH****.

**Fix (firewalld: CentOS)**

```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<scoring-engine-IP>" port port="<scored-port>" protocol="<tcp-or-udp>" accept'
sudo firewall-cmd --reload
```

## 5) **Services restarted**

An attacker turned back on a service you disabled, or added a new one.

**Find**

```bash
systemctl list-units --type=service --state=running
systemctl list-unit-files --type=service --state=enabled
sudo find /etc/systemd/system /lib/systemd/system -name "*.service" -mmin -<minutes>
systemctl cat <service>
```

- A new unit file, or an ExecStart pointing at /tmp or a home folder, is suspicious.

**Fix**

```bash
sudo systemctl stop <service>
sudo systemctl disable <service>
sudo systemctl mask <service>
```

- Masking stops it from being started again, even by another service.

## 6) **Audit tampering**

An attacker stopped auditd, removed your rules, or wiped logs.

**Find**

```bash
sudo systemctl status auditd
sudo auditctl -s
sudo auditctl -l
sudo ausearch -m USER_AUTH
sudo ausearch -m CONFIG_CHANGE,DAEMON_END -ts today
ls -la /var/log
```

- `auditctl -s` shows if auditing is enabled. CONFIG_CHANGE and DAEMON_END events show rule changes and auditd stopping. Tiny or empty log files in `/var/log` point to wiping.

**Fix**

```bash
sudo systemctl start auditd
sudo service auditd restart
sudo auditctl -e 1
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
```

- On CentOS, systemctl refuses to restart auditd, so use `service auditd restart`. Re-add any watch rules that went missing.

## 7) **Time changed**

An attacker changed the clock or the time source, which breaks log timelines.

**Find**

```bash
date
timedatectl
chronyc tracking
chronyc sources
```

**Fix**

```bash
sudo timedatectl set-ntp true
sudo chronyc makestep
sudo systemctl restart chronyd
sudo systemctl restart systemd-timesyncd
chronyc tracking
```

- If the NTP server was changed, set it back to **`<NTP-server>`** in `/etc/chrony.conf` or `/etc/systemd/timesyncd.conf` before restarting. On Debian / Ubuntu the chrony service may be named chrony.

## Before destructive action

Report to the scoring team (competition staff) before any destructive action on a scored item: deleting a scored user, stopping a service a scored check uses, or anything you are not sure about. A broken scored service costs uptime points, and a box reset **costs 200**.
