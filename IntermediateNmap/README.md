# Intermediate NMap
Description hints at using nmap to find a high port and use netcat to get credentials from that. Room is exactly that.

## Enumeration and Netcat

Full port scan (nmap -sC -p- (IP) revealed three open ports: 22 (SSH), 2222, and 31337 (Elite).

Connecting to the high port with netcat:

nc MACHINE_IP 31337

The service returned valid SSH credentials for user ubuntu.

Using these credentials to log in:

ssh ubuntu@MACHINE_IP

After gaining access, enumeration of the home directories showed another user directory at /home/user.

The flag was located at:

/home/user/flag.txt

cat /home/user/flag.txt
