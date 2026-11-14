# Windows Blue-Team Hardening Commands

For competition Windows hosts only (lab / authorized PvP boxes). Test on kamino first. Do not run blind on a scored box.

Placeholders in angle brackets are values you may change: `<feature-name>`, `<username>`, `<database>`, `<policy-file>`, `<NTP-server>`, `<backup-drive>`.

## 1) Firewall baseline

- Turns the firewall on for all profiles, then deletes existing rules so you can rebuild an allow-list.

```
netsh advfirewall set allprofiles state on
netsh advfirewall firewall delete rule name=all
```

## 2) Disable unneeded services and features

- Lists installed features, then uninstalls optional components that add attack surface.

```
Get-WindowsFeature
Uninstall-WindowsFeature <feature-name>
```

Examples for `<feature-name>`:

```
Telnet-Client
SNMP-Service
Web-Server
```

## 3) Lock down accounts and passwords

- Deactivates built-in or unused accounts, applies a strong password policy via secedit, and reviews local Administrators membership.

```
net user Administrator /active:no
net user <username> /active:no
secedit /configure /db <database> /cfg <policy-file> /areas SECURITYPOLICY
net localgroup administrators
```

- Password policy goals in `<policy-file>`: minimum 14 characters, complexity on, reversible encryption off.

Example:

```
secedit /configure /db C:\Windows\security\local.sdb /cfg C:\temp\passpol.inf /areas SECURITYPOLICY
```

## 4) Patch and reboot

- Installs security updates only (no feature updates), then reboots so they apply.

```
sconfig
```

(choose option 6 for Download and Install Updates)

Or use Windows Update (security updates only), then:

```
shutdown /r /t 0
```

## 5) Enable auditing

- Records logon events, failed file-system access, and process creation to the Windows event log.

```
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"File System" /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable
```

## 6) Sync time with NTP

- Points Windows Time at an NTP peer and forces a resync so log timestamps align with the scoring engine.

```
w32tm /config /manualpeerlist:<NTP-server> /syncfromflags:manual /reliable:yes
w32tm /resync
```

- Default for `<NTP-server>`: pool.ntp.org

## 7) Back up before changes

- Takes a backup or copies critical configs so you can roll back without a full box reset.

```
wbadmin start backup -backupTarget:<backup-drive> -include:C: -allCritical -quiet
```

- Or copy critical configs to a folder you control (adjust paths as needed).

## Closing reminder

Verify what the scoring engine checks before disabling anything (services, ports, users, or firewall). LDAP on thehouse, SMB on sunflower, and any Quotient-checked accounts must stay reachable and usable after password change requests.
