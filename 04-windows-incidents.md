# Windows Incident Fixes (Blue-Team Shift)

For competition incidents on your own team's Windows boxes only (authorized PvP environment). Each incident lists Find (confirm what happened), then Fix.

Placeholders in angle brackets are values you fill in: `<username>`, `<new-password>`, `<rogue-user>`, `<task-name>`, `<file-path>`, `<quarantine-folder>`, `<process-id>`, `<script-name>`, `<service-name>`, `<scoring-engine-IP>`, `<scored-port>`, `<tcp-or-udp>`, `<firewall-backup.wfw>`, `<NTP-server>`.

Never modify, move, or delete any flag file (for example `C:\Windows\flag.txt`). That is a disqualification.

## 1) Locked out of accounts

An attacker disabled, locked, or changed your accounts, or added their own admin.

**Find**

```
net user
net user <username>
net localgroup administrators
Search-ADAccount -LockedOut | Select-Object SamAccountName
```

- `net user <username>` shows "Account active" and whether it is locked. `Search-ADAccount` is for the domain controller (thehouse).

**Fix**

```
net user <username> /active:yes
net user <username> <new-password>
net localgroup administrators <username> /add
net localgroup administrators <rogue-user> /delete
Unlock-ADAccount -Identity <username>
Enable-ADAccount -Identity <username>
```

- Scored users: submit any new password through the scoring engine's password change request.

## 2) Scheduled task sabotage

An attacker added a scheduled task (the Windows version of cron) to re-open access or undo your fixes.

**Find**

```
schtasks /query /fo LIST /v
Get-ScheduledTask | Where-Object { $_.TaskPath -notlike "\Microsoft\*" } | Select-Object TaskPath, TaskName, State
schtasks /query /tn <task-name> /v /fo LIST
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Run
taskschd.msc
```

- In Task Scheduler (`taskschd.msc`), check the Task Scheduler Library for tasks you did not create. The "Task To Run" line shows what each task executes. The Run keys are another common way to auto-start malware.

**Fix**

```
schtasks /change /tn <task-name> /disable
schtasks /delete /tn <task-name> /f
```

- Disable first if you are unsure, then delete once you confirm it is not part of a scored service.

## 3) Malicious scripts dropped

An attacker left PowerShell, batch, or VBScript files on the box.

**Find**

```
Get-ChildItem -Recurse C:\ -Include *.ps1,*.bat,*.vbs -ErrorAction SilentlyContinue
Get-ChildItem -Recurse C:\Users, C:\ProgramData, C:\Windows\Temp -Include *.ps1,*.bat,*.vbs -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 30 FullName, LastWriteTime
Get-Content <file-path>
Get-CimInstance Win32_Process | Where-Object CommandLine -match "<script-name>" | Select-Object ProcessId, CommandLine
```

- Scanning all of C:\ is slow. The second command checks the usual drop folders and shows the newest files first. Read a script before you remove it.

**Fix**

```
Stop-Process -Id <process-id> -Force
New-Item -ItemType Directory -Path <quarantine-folder> -Force
Move-Item <file-path> <quarantine-folder>
Remove-Item <file-path> -Force
```

- Stop any process running the script first. Moving it to a quarantine folder keeps evidence; remove it once you are sure.

## 4) Firewall rules deleted or firewall turned off

An attacker disabled the firewall or wiped your allow rules.

**Find**

```
netsh advfirewall show allprofiles
netsh advfirewall firewall show rule name=all
Get-Service MpsSvc
```

**Fix**

```
Set-Service MpsSvc -StartupType Automatic
Start-Service MpsSvc
netsh advfirewall set allprofiles state on
netsh advfirewall firewall add rule name="Allow <service-name> from scoring" dir=in action=allow protocol=<tcp-or-udp> remoteip=<scoring-engine-IP> localport=<scored-port>
netsh advfirewall export <firewall-backup.wfw>
netsh advfirewall import <firewall-backup.wfw>
```

- Rebuild allow rules from the scoring sheet, one per scored port. Once the rules are right, export them so next time you can restore with one import.

## 5) Services re-enabled

An attacker turned back on a service you disabled, or installed a new one.

**Find**

```
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-CimInstance Win32_Service | Where-Object StartMode -ne 'Disabled' | Select-Object Name, StartMode, State, PathName
Get-WinEvent -FilterHashTable @{LogName='System';ID=7045} -MaxEvents 20
```

- Event 7045 is a new service installed. A PathName in a user or temp folder is suspicious.

**Fix**

```
Stop-Service <service-name> -Force
Set-Service <service-name> -StartupType Disabled
sc.exe delete <service-name>
```

- Only use `sc.exe delete` on a service you confirmed is malicious and not scored.

## 6) Audit log cleared

An attacker wiped the Security log to hide their tracks.

**Find**

```
Get-WinEvent -FilterHashTable @{LogName='Security';ID=1102} -MaxEvents 5 | Format-List TimeCreated, Message
Get-WinEvent -FilterHashTable @{LogName='System';ID=104} -MaxEvents 5 | Format-List TimeCreated, Message
vssadmin list shadows
auditpol /get /category:*
```

- 1102 means the Security log was cleared, and its message names the account that did it. 104 means another log was cleared. Shadow copies may still hold older files. If the list is empty, the attacker may have deleted them too.

**Fix**

```
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Process Creation" /success:enable
net user <username> <new-password>
```

- Re-apply your audit policy if it was changed, and reset the password of the account that cleared the log.

## 7) Time changed

An attacker changed the clock or time source, which breaks log timelines and can break Kerberos logons on the domain.

**Find**

```
Get-Date
w32tm /query /status
w32tm /query /source
Get-WinEvent -FilterHashTable @{LogName='Security';ID=4616} -MaxEvents 5
```

- Event 4616 means the system time was changed.

**Fix**

```
w32tm /config /manualpeerlist:<NTP-server> /syncfromflags:manual /reliable:yes
net stop w32time
net start w32time
w32tm /resync /force
w32tm /query /status
```

## Before destructive action

Report to the scoring team (competition staff) before any destructive action on a scored item: deleting a scored user, a service a scored check uses, or anything you are not sure about. A broken scored service costs uptime points, and a box reset costs 200.
