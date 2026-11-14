# Linux Blue-Team Hardening Commands

For competition Linux hosts only (lab / authorized PvP boxes: Ubuntu cherrybomb, CentOS chomper, Debian peashooter). Test on kamino first. Do not run blind on a scored box.

Placeholders in angle brackets are values you may change: `<interface>`, `<username>`, `<allowed-port>`, `<NTP-server>`, `<distro-package-manager>`, `<backup-location>`, `<service>`.

## 1) Firewall

- Installs a firewall if one is missing, sets inbound default to deny, and allows only the ports you need.

Install (pick the one for your distro):

```
sudo apt install ufw
sudo yum install firewalld
```

ufw (Ubuntu / Debian) - allow your ports BEFORE enabling so you do not lock yourself out:

```
sudo ufw default deny incoming
sudo ufw allow <allowed-port>/tcp
sudo ufw enable
sudo ufw status verbose
```

firewalld (CentOS):

```
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
sudo firewall-cmd --permanent --add-port=<allowed-port>/tcp
sudo firewall-cmd --reload
```

Optional: limit a rule to one interface with `<interface>`:

```
sudo ufw allow in on <interface> to any port <allowed-port> proto tcp
```

## 2) Disable unneeded services and daemons

- Stops, disables, and masks services the scoring engine does not check.

```
sudo systemctl stop <service>
sudo systemctl disable <service>
sudo systemctl mask <service>
```

## 3) Lock down accounts and passwords

- Locks unused accounts, reviews sudo access, and enforces a minimum password length.

```
sudo passwd -l <username>
sudo visudo
sudo nano /etc/security/pwquality.conf
```

In `/etc/security/pwquality.conf` set:

```
minlen = 14
```

- Do not remove users until scoring dependencies are confirmed.

## 4) Patch and reboot

- Updates packages with the distro package manager (security updates only where possible), then reboots.

```
sudo <distro-package-manager> update && sudo <distro-package-manager> upgrade
sudo reboot
```

- `<distro-package-manager>` is apt (Ubuntu / Debian) or yum (CentOS).

## 5) Enable auditing

- Turns on auditd and watches /etc/passwd for writes and attribute changes.

```
sudo systemctl enable auditd
sudo systemctl start auditd
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
```

## 6) Sync time with NTP

- Points chrony or systemd-timesyncd at an NTP server and restarts it.

Edit one of these files:

```
/etc/chrony.conf
/etc/systemd/timesyncd.conf
```

Set:

```
NTP=<NTP-server>
```

(in chrony.conf the line is: `server <NTP-server> iburst`)

- Default for `<NTP-server>`: pool.ntp.org

Restart the one you use:

```
sudo systemctl restart chronyd
sudo systemctl restart systemd-timesyncd
```

## 7) Back up before changes

- Copies critical configs and logs so you can roll back without a full box reset.

```
sudo cp -r /etc/ssh /etc/passwd /etc/group /var/log <backup-location>
```

## Closing reminder

Verify what the scoring engine checks before disabling anything (services, ports, users, or firewall). Keep Pi-hole and HTTP reachable on cherrybomb, the database reachable on chomper, and SSH reachable on peashooter, and submit password change requests for any scored users.
