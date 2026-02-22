# PickleRick

## Initial Enumeration with nmap and gobuster

Nmap revealed only two relevant services: SSH on 22 and Apache on 80. Since the room description explicitly mentions exploiting a web server, HTTP was the intended attack surface.

Visiting the main page and inspecting the HTML source revealed a leaked username:

(redacted by yours truly, GL)

Directory enumeration with gobuster identified several interesting endpoints:

login.php
portal.php
robots.txt

Retrieving robots.txt via curl http://10.82.190.181/robots.txt revealed the password:

(redacted by yours truly, GL)

## Web Exploitation

Using the leaked credentials on /login.php granted access to portal.php, which contained a command execution panel. This provided remote command execution as the web server user.

Verification:

whoami → www-data
id → uid=33(www-data)

The command panel had filtering in place: cat was blocked. So we needed a different command. I chose "less".
also i tried find / -name "*txt" to let it list all .txt files. I put the output in ChatGPT to look for interesting texts. there were clue.txt and supersecretingridient.txt (which I would also have found via ls -la /home)

## Ingredient Discovery

First ingredient was located in the web directory and read using:

less supersecretingredient.txt

Second ingredient was discovered by enumerating /home:

/home/rick was world-readable and world-writable (777). Inside was:

second ingredients

Because of the space in the filename, it was accessed using quoting or escaping:

less "/home/rick/second ingredients"
or
less second\ ingredients

## Privilege Escalation to root
I prob needed root to get the last ingridient. so we checked /checked

Running:

sudo -l

Revealed:

(ALL) NOPASSWD: ALL

This meant www-data could execute any command as root without a password.

Attempting an interactive shell with sudo -s failed due to the account using nologin. Instead of pivoting, root commands were executed directly:

sudo ls -la /root
sudo /bin/bash -c "cat /root/3rd.txt"

This provided the third ingredient.

## Key Reusable Patterns

Credentials are often leaked in HTML source or robots.txt in easy web rooms.

If a web command panel blocks specific commands, assume keyword filtering. Use alternatives such as less, head, sed, or absolute paths.

Spaces in filenames must be escaped with backslashes or handled with quotes.

In easy rooms, sudo NOPASSWD ALL is an intended privilege escalation path. Execute single root commands directly instead of trying to spawn interactive shells in limited web environments.
