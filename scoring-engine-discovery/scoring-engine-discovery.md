# Scoring Engine Discovery

For competition hosts only (lab / authorized PvP boxes: Windows, Ubuntu cherrybomb, CentOS chomper, Debian peashooter). Use this to find the IPs you use for remote access, and to identify the scoring engine.

**The scoring engine is the remote IP that connects TO your scored ports on a regular interval, NOT an open listener you scan for.** Scanning finds targets; watching inbound connections to your scored ports finds the scorer.

Placeholders in angle brackets are values you fill in: **`<scored-port>`**, **`<scored-ports>`**, **`<target-IP>`**.

## Windows (PowerShell)

### Find your own IP addresses

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.IPAddress -notlike "127.*"} | Select-Object IPAddress,InterfaceAlias
```

- Ignore **127.0.0.1**. The remaining **IPAddress** values are the ones you (and the scoring engine) can reach this box on. Note which **InterfaceAlias** is the competition NIC.

### Find the scoring engine

Watch who is connected **to** your scored ports. Swap the port list for whatever the scoring sheet says is scored. Example ports: **389**, **445**, **88**, **135**, **139**.

```powershell
Get-NetTCPConnection -State Established | Where-Object {$_.LocalPort -in 389,445,88,135,139} | Select-Object LocalPort,RemoteAddress,RemotePort | Sort-Object RemoteAddress
```

- Replace `389,445,88,135,139` with **`<scored-ports>`**. The **RemoteAddress** on those lines is the scoring engine candidate. A genuine scorer reconnects on a regular interval (often every 30–60 seconds). Ignore your Kali box and known Microsoft/cloud IPs.

To include non-established states (Listen, TimeWait, SynReceived) on one port at a time:

```powershell
Get-NetTCPConnection -LocalPort <scored-port> | Select-Object RemoteAddress,State
```

- A line that only shows **Listen** with no remote client means that port is open but nobody is connected right now. Re-run while a scoring check should be happening.

### Mapping live hosts and ports (this does not find the scorer)

`Test-NetConnection` is useful for mapping which neighbors are up, or whether a given port is open on a host. That finds **targets**, not the engine that is connecting to you.

Single host, one port:

```powershell
Test-NetConnection -ComputerName <target-IP> -Port <scored-port> -WarningAction SilentlyContinue
```

Subnet sweep for one port (slow; stop with **Ctrl+C** once you have what you need):

```powershell
1..254 | ForEach-Object { $ip = "192.168.1.$_"; if (Test-NetConnection -ComputerName $ip -Port <scored-port> -WarningAction SilentlyContinue -InformationLevel Quiet) { $ip } }
```

- Any IP printed has that port open. Confirm the actual scorer with **Get-NetTCPConnection** above, not with this sweep.

## Linux (Debian / Ubuntu)

### Find your own IP addresses

```bash
ip addr show
ip -4 addr
```

- `ip -4 addr` lists IPv4 only. Ignore **127.0.0.1**. The remaining addresses on the competition interface are the ones used for remote access and scoring checks.

### Find the scoring engine

See who is connected **to** a scored port on this box:

```bash
ss -tnp
ss -tn state established '( sport = :<scored-port> )'
```

- `ss -tnp` shows all TCP connections with process names (needs root for some process detail). The second line filters to established connections whose **local** (source) port is **`<scored-port>`**. The peer address on those lines is the scoring engine candidate.

Older alternative if `ss` is missing:

```bash
sudo netstat -tnp
```

Watch for the address that keeps coming back:

```bash
watch -n2 'ss -tn | grep -E ":<scored-port>|:<scored-port-2>"'
```

- Refresh every 2 seconds. The remote IP that reappears on a regular interval is the scorer. **Ctrl+C** to stop. Use **`grep -E`** with **`|`** between ports (e.g. `:<scored-port>|:<scored-port-2>`) to watch more than one scored port at once.

### Mapping targets (this does not find the scorer)

`nmap` (and port probes in general) finds hosts that **listen** on a port — other boxes, not the client that is scoring you.

```bash
nmap -p <scored-port> 192.168.1.0/24
```

- Any host that shows the port open is a candidate **target**. **It does not find the scoring engine.** Confirm the scorer with `ss` / `watch` on your own scored ports.

## CentOS

Same discovery tools as Linux: **`ip addr`**, **`ss`**, and **`netstat`**. The difference is the firewall and package manager.

### Find your own IP addresses

```bash
ip addr show
ip -4 addr
```

### Find the scoring engine

```bash
ss -tnp
ss -tn state established '( sport = :<scored-port> )'
sudo netstat -tnp
watch -n2 'ss -tn | grep -E ":<scored-port>|:<scored-port-2>"'
```

- Same meaning as on Debian / Ubuntu: the recurring **remote** address connecting to your **`<scored-port>`** is the scoring engine candidate.

If `ss` or `netstat` is missing, install the tools with **dnf** (current CentOS) or **yum** (CentOS 7):

```bash
sudo dnf install iproute net-tools
sudo yum install iproute net-tools
```

- `ip` and `ss` come from **iproute**; `netstat` comes from **net-tools**.

### firewalld and SELinux

Check the running **firewalld** policy so you know which ports are actually reachable (this is not a substitute for watching connections):

```bash
sudo firewall-cmd --list-all
```

- Use **firewalld**, not ufw. **SELinux** does not block observing connections with `ss` or `netstat`, but it can matter if a scored service itself fails to bind or accept. Keep SELinux **enforcing**; do not disable it to "see" traffic.

### Mapping targets (this does not find the scorer)

Same caveat as Linux: a port scan finds listeners, not the client that connects to you. Use `ss` on your scored ports to identify the scorer.

## Chaining discovery commands

You can string steps together instead of running each one by hand.

- **Pipe (`|`)** — send one command's output into the next. Linux / CentOS example: `ss -tn | grep -E ":<scored-port>|:<scored-port-2>"`. Windows already does this with `| Where-Object` and `| Select-Object` in the PowerShell examples above.
- **Back to back in one block** — paste several lines and run them as a unit (PowerShell or a bash script). Each line runs in order; use this when you want "find my IPs, then watch scored ports" without retyping.

```bash
ip -4 addr
ss -tn state established '( sport = :<scored-port> )'
watch -n2 'ss -tn | grep -E ":<scored-port>|:<scored-port-2>"'
```

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.IPAddress -notlike "127.*"} | Select-Object IPAddress,InterfaceAlias
Get-NetTCPConnection -State Established | Where-Object {$_.LocalPort -in <scored-ports>} | Select-Object LocalPort,RemoteAddress,RemotePort | Sort-Object RemoteAddress
```

- Piping filters a single stream. A multi-line block runs a short checklist. Use whichever fits the moment.

## What to look out for

- Start from the scoring sheet: know **`<scored-port>`** / **`<scored-ports>`** before you watch traffic.
- The scorer is a **client** connecting inbound to you, on a regular interval. An IP that only answers a ping or has a port open is not automatically the engine.
- Re-run the established-connection commands during a scoring window if the first pass shows only **Listen** and no remote address.
- Do not confuse your own jump box, Kali, or a Microsoft/cloud IP with the scorer. Confirm against the scoring sheet when you can.
