# CentOS Remote Access (from your laptop)

PowerShell commands to run from your local laptop to reach a CentOS competition VM (authorized boxes only, including **chomper**). Placeholders: **`<target-IP>`**, **`<user>`**.

## 1) SSH

**Verify port 22 is reachable first:**
```powershell
Test-NetConnection -ComputerName <target-IP> -Port 22
```
- **TcpTestSucceeded: True** means SSH is open; **False** means it's down, on another port, or firewalled.

**Connect (ssh ships with Windows 10/11 and Server):**
```powershell
ssh <user>@<target-IP>
```

## 2) XRDP setup (connect with mstsc on 3389)

**CentOS uses dnf, not apt.** On the CentOS box:
```bash
sudo dnf install xrdp
sudo systemctl enable --now xrdp
```

**Open 3389 in firewalld (not ufw):**
```bash
sudo firewall-cmd --permanent --add-port=3389/tcp
sudo firewall-cmd --reload
```

**Then connect from Windows:**
```cmd
mstsc /v:<target-IP>
```

**Verify 3389 after setup:**
```powershell
Test-NetConnection -ComputerName <target-IP> -Port 3389
```

**SELinux:** if **xrdp** will not start, do not disable SELinux. Check the mode and recent denials:
```bash
getenforce
sestatus
sudo ausearch -m avc -ts recent
sudo systemctl status xrdp
```
- `getenforce` should stay **Enforcing**. Fix the denial (package policy, `restorecon`, or a port context) rather than running `setenforce 0`.

## 3) VNC alternative (connect on 5901)

On the CentOS box:
```bash
sudo dnf install tigervnc-server
vncserver
```

**Open 5901 in firewalld:**
```bash
sudo firewall-cmd --permanent --add-port=5901/tcp
sudo firewall-cmd --reload
```

- Connect a VNC client to **`<target-IP>:5901`** (display `:1`).

## 4) Package manager note

**CentOS 7 uses `yum`; current versions (8/9/Stream) use `dnf`.** If `dnf` isn't found, swap it for `yum` in the install commands above — the package names are the same.
