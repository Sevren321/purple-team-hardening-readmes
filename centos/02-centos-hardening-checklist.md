# CentOS Hardening Checklist

For competition CentOS hosts only (lab / authorized PvP box: CentOS **chomper**). Test on kamino first. Do not run blind on a scored box.

## 1) **Firewall**

- [ ] **firewalld** is running; scored ports are allowed only from scoring IPs via rich rules; those allow rules (including **SSH**) were added before tightening the default zone so you are not locked out. **SELinux** is left **enforcing**.

## 2) **Disable unneeded services and daemons**

- [ ] **systemd** is used to stop, disable, and mask anything the scoring engine does not check, while keeping anything a scored service (including the database on **chomper**) depends on.

## 3) **Lock down accounts and passwords**

- [ ] Reviews **wheel**/sudo and root access, enforces a 14-character minimum via **pam_pwquality**, locks unused accounts, and does not remove users until scoring dependencies are confirmed.

## 4) **Patch and reboot**

- [ ] Applies security updates with **dnf** (`dnf update --security`), or **yum** on **CentOS 7** (`yum update --security`), then reboots.

## 5) **Enable auditing**

- [ ] Turns on **auditd** to log authentication, file access, and process events, and watches **`/var/log/secure`** for sshd auth.

## 6) **Sync time with NTP**

- [ ] Configures **chrony** to the competition NTP server (or pool.ntp.org).

## 7) **Back up before changes**

- [ ] Takes a hypervisor snapshot as the primary rollback, and archives critical `/etc` configs (and logs) to a path on the VM as a fallback.

## Closing reminder

Verify what the scoring engine checks before disabling anything (services, ports, users, or firewall). Keep the database reachable on **chomper**, and submit password change requests for any scored users. A box reset **costs 200** points.
