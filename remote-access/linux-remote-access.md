# Linux Remote Access (from your laptop)

PowerShell commands to run from your local laptop to reach a Linux competition VM. Authorized boxes only. Placeholders: **`<target-IP>`**, **`<user>`**.

## SSH straight from PowerShell

`ssh` ships with Windows 10/11 and Server.

```powershell
ssh <user>@<target-IP>
```

## Verify SSH port 22 is reachable first

```powershell
Test-NetConnection -ComputerName <target-IP> -Port 22
```

- **TcpTestSucceeded: True** means SSH is open; **False** means it's down, on another port, or firewalled.

## Remote desktop on Debian 12 (no native RDP)

**Debian 12 does not use RDP natively.** Add one of these on the Linux box.

### XRDP — then connect with mstsc on 3389

```bash
sudo apt install xrdp
sudo systemctl enable --now xrdp
```

- Connect from Windows: **`mstsc /v:<target-IP>`** on port **3389**.

### VNC (TightVNC) — connect on 5901

```bash
sudo apt install tightvncserver
vncserver
```

- Connect a VNC client to **`<target-IP>:5901`**.

## Verify XRDP's port after setup

```powershell
Test-NetConnection -ComputerName <target-IP> -Port 3389
```
