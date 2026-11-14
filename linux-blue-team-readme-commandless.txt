linux-blue-team-readme-commandless.txt
For competition Linux hosts only (lab / authorized PvP boxes: Ubuntu cherrybomb, CentOS chomper, Debian peashooter). Test on kamino first. Do not run blind on a scored box.

============================================================
1) Firewall
============================================================
May need to be installed first if absent (for example ufw or firewalld); once present, blocks inbound traffic not explicitly allowed.

============================================================
2) Disable unneeded services and daemons
============================================================
Stops and masks anything the scoring engine does not check.

============================================================
3) Lock down accounts and passwords
============================================================
Reviews sudoers and root access, enforces a strong password policy via pam_pwquality or similar, disables or locks unused accounts, and does not remove users until scoring dependencies are confirmed.

============================================================
4) Patch and reboot
============================================================
Updates packages with the distro package manager (apt or yum), security updates only, then reboots.

============================================================
5) Enable auditing
============================================================
Turns on auditd or similar to log authentication, file access, and process events.

============================================================
6) Sync time with NTP
============================================================
Configures chrony or systemd-timesyncd to pool.ntp.org.

============================================================
7) Back up before changes
============================================================
Copies critical configs like /etc/ssh, /etc/passwd, /etc/shadow-related policy files, and logs off the box for rollback.

============================================================
Closing reminder
============================================================
Verify what the scoring engine checks before disabling anything (services, ports, users, or firewall). Keep Pi-hole and HTTP reachable on cherrybomb, the database reachable on chomper, and SSH reachable on peashooter, and submit password change requests for any scored users.
