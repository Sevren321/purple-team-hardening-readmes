# Purple vs Purple Blue-Team Hardening READMEs

Numbered in the order to use them. For competition hosts only; verify what the scoring engine checks before disabling anything.

- [01-windows-hardening-with-commands.md](01-windows-hardening-with-commands.md): Windows hardening steps 1 to 7 with Find and Do commands (firewall, features, accounts, patching, auditing, NTP, VM backup).
- [02-linux-hardening-with-commands.md](02-linux-hardening-with-commands.md): Linux hardening steps 1 to 7 with Find and Do commands (ufw/firewalld, services, accounts, patching, auditd, NTP, VM backup).
- [03-windows-hardening-checklist.md](03-windows-hardening-checklist.md): Windows hardening checklist, the seven steps without commands.
- [03-linux-hardening-checklist.md](03-linux-hardening-checklist.md): Linux hardening checklist, the seven steps without commands.
- [04-windows-incidents.md](04-windows-incidents.md): Windows incident fixes during a shift (lockouts, scheduled tasks, scripts, firewall, services, cleared logs, time).
- [04-linux-incidents.md](04-linux-incidents.md): Linux incident fixes during a shift (lockouts, cron, scripts, firewall, services, audit tampering, time).
- [06-password-playbook.md](06-password-playbook.md): Password security playbook: policy, generating and rotating strong passwords, blue-team watch, and what not to do.

## Red team (local only, not for the public repo)

- [red-team/05-red-team-attack-playbook.md](red-team/05-red-team-attack-playbook.md): Kali attack playbook: recon, credential attacks, exploitation, persistence/lateral movement, and post-exploitation flag hunting.
