# Windows Remote Access (from your laptop)

PowerShell commands to run from your local laptop to reach a Windows competition VM. Authorized boxes only. Placeholders: **`<target-IP>`**.

## Connect with RDP (port 3389)

```cmd
mstsc /v:<target-IP>
```

## Verify 3389 is reachable first

```powershell
Test-NetConnection -ComputerName <target-IP> -Port 3389
```

- **TcpTestSucceeded: True** means RDP is open.

## On the Windows box — enable / check RDP

```powershell
Get-Service TermService
Set-Service -Name TermService -StartupType Automatic; Start-Service TermService
```

## Allow RDP through the firewall (scope the source to your IP)

```powershell
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
New-NetFirewallRule -DisplayName "Allow RDP from me" -Direction Inbound -Protocol TCP -LocalPort 3389 -RemoteAddress <target-IP> -Action Allow
```

## If connections are refused, flip the RDP switch

```powershell
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" -Value 0
```
