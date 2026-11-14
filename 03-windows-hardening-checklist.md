# Windows Hardening Checklist

For competition Windows hosts only (lab / authorized PvP boxes). Test on kamino first. Do not run blind on a scored box.

## 1) Firewall baseline

- [ ] Blocks inbound traffic you did not explicitly allow.

## 2) Disable unneeded services and features

- [ ] Removes optional Windows components and running programs that add attack surface.

## 3) Lock down accounts and passwords

- [ ] Deactivates built-in admin where safe, enforces a strong password policy, reviews admin group membership, disables guest accounts, and does not remove any users until scoring dependencies are confirmed.

## 4) Patch and reboot

- [ ] Installs the latest security updates only (no feature updates) and reboots so they apply.

## 5) Enable auditing

- [ ] Records logon events, file system access, and process creation to the Windows event log.

## 6) Sync time with NTP

- [ ] Sets the clock to pool.ntp.org so log timestamps match the scoring engine.

## 7) Back up before changes

- [ ] Copies critical configs and event logs off the box for rollback.

## Closing reminder

Verify what the scoring engine checks before disabling anything (services, ports, users, or firewall). LDAP on thehouse, SMB on sunflower, and any Quotient-checked accounts must stay reachable and usable after password change requests.
