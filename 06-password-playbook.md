# Password Playbook — Purple vs Purple

How to set, generate, and rotate passwords that resist brute force and cracking, and how to spot an attack against them. A companion to the hardening and incident READMEs.

## Rules of engagement (read first)

- For your own team's authorized competition accounts only. Use this on the PvP boxes you are cleared to manage.
- **Scored accounts:** any password change for a scored user must also go through the **scoring engine's password change request**, or the scored check fails.
- Placeholders in angle brackets are values you fill in: **`<username>`**, **`<new-password>`**.

---

## 1) Password policy requirements

**Must-have rules (from team policy):**

- **Minimum 14 characters.** Longer is better; length beats complexity against cracking.
- **At least one irreversibly stored / hashed password** — never plaintext. Store with a strong one-way hash (**bcrypt**, **Argon2**, or the OS default: Windows NTLM/AD, Linux `/etc/shadow` via `passwd`). On Windows, keep **"Store passwords using reversible encryption"** set to **Disabled**.

**Also follow these:**

- **Complexity:** at least one **uppercase**, one **lowercase**, one **digit**, and one **symbol**.
- **No dictionary words** on their own, and no predictable substitutions (`P@ssw0rd` is a dictionary word).
- **No username, hostname, team name, or competition name** anywhere in the password.
- **Unique per account** — never the same password on two boxes or two users.

**More rules (add as you review):**

- <!-- e.g. max age, lockout threshold, banned-password list, MFA where available -->
- 
- 

---

## 2) How to generate strong passwords

**Random string (Linux / Kali):**

```bash
# 18 random bytes, base64 encoded (~24 chars)
openssl rand -base64 18

# pwgen: one 20-char password with at least one symbol and capital
pwgen -sy 20 1
```

**Passphrase approach** (easier to type, still strong): pick **4-5 random, unrelated words**, then add a **digit** and a **symbol**.

- Example shape (do not use this exact one): `copper-lantern-river-maple7!`
- Keep the words genuinely random, not a quote or a phrase.

On Windows you can run the same **`openssl`** from Git Bash or WSL, or generate on Kali and transfer the value securely (not over chat).

---

## 3) How to change passwords

**Windows:**

```bash
net user <username> <new-password>

# Prompt for the password instead of putting it on the command line (safer):
net user <username> *
```

**Linux:**

```bash
# Set a new password interactively (does not echo)
sudo passwd <username>

# Set maximum password age to 90 days
sudo chage -M 90 <username>

# Verify the aging settings
sudo chage -l <username>
```

- Prefer the interactive forms (**`net user <username> *`**, **`sudo passwd`**) so the password never lands in your shell history.
- **Scored users:** submit the new password through the **scoring engine's password change request** as well.

---

## 4) How often to change

- **At least every 90 days** as a baseline (enforced on Linux with `chage -M 90`).
- **Immediately** if a credential may have leaked, been typed into the wrong place, or sent over an insecure channel.
- **After any suspected compromise** of the box, rotate every account on it, not just the one you think was hit.
- At the **start of the competition**, change every default and handed-out password before the grace period ends.

---

## 5) What blue team should watch for

- **Windows:** a burst of event ID **4625** (failed logon) from one source IP; **4740** on account lockout; a **4624** success right after a run of failures means a guess landed. Count 4625 by source IP (see the incident README).
- **Linux:** repeated **`Failed password`** lines in **`/var/log/auth.log`** (Debian/Ubuntu) or **`/var/log/secure`** (RHEL/CentOS); **`auditd`** USER_AUTH records with `res=failed`.
- **Password spray:** one password tried across **many usernames** from a single IP. It stays under per-account lockout limits, so watch for many 4625s / failed auths spread across accounts rather than piled on one.
- Follow logins live:

```bash
sudo tail -f /var/log/auth.log
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr
```

---

## 6) What NOT to do

- **Never reuse** a password across accounts or boxes.
- **Never share** passwords in chat, Discord, email, or on sticky notes.
- **Never store** them in a plain-text file on the VM (that is the first thing an attacker greps for).
- **Never** put a password on the command line where history or process listings can capture it — use the prompt forms from section 3.
- **Never** keep default or handed-out passwords once the game starts.
- If the team must share credentials for shared stations, agree them verbally or through an approved password manager, not in a repo or a text file.

---

## 7) Finding placeholder values

- **`<username>`**: on Windows, run **`net user`** (all local accounts) and **`net localgroup administrators`** (admins); on the DC, **`Get-ADUser -Filter *`**. On Linux, **`cat /etc/passwd`** (all accounts) and **`getent group sudo wheel`** (privileged users).
- **`<new-password>`**: generate per section 2 (**`openssl rand -base64 18`**, **`pwgen -sy 20 1`**, or a 4-5 word passphrase).

---

## Closing note

For authorized competition environments only. Report any password issue — a suspected leak, a brute-force spike, or a locked scored account — to the **scoring team** before taking destructive action on a scored user.
